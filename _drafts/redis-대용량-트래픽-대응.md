---
title: "Redis 대용량 트래픽 대응"
categories: [Redis]
tags: [redis, cache-stampede, hot-key, traffic, fault-tolerance]
---

 ++ Cache Penetration (없는 key 자꾸 조회하면..~? DB에 조회가 가요~?)
   - null로 cache / bloom filter라는게 있다는데..

### 개요
트래픽이 몰리는 경우 redis에서 발생가능한 문제에 대해 고민해보자.

---

### 대용량 트래픽 캐시 설계 구조

```
Client
 ↓
L1 Cache (Caffeine) ← 각 Pod 로컬, 초고속, 용량 제한
 ↓ miss
L2 Cache (Redis) ← 분산 캐시, Pod간 공유
 ↓ miss
SingleFlight ← 동일 요청 dedupe, DB 부하 방지
 ↓
DB
```

- L1 hit → 즉시 반환 (네트워크 비용 없음)
- L1 miss, L2 hit → Redis에서 가져오고 L1에 적재
- L2 miss → SingleFlight로 중복 DB 조회 방지 후 L1/L2 모두 적재
- 캐시 무효화 시 → Redis Pub/Sub 등으로 각 Pod의 L1도 함께 무효화

---

### 1. Redis 장애 대응
Redis 장애 나면 어떻게 할거냐?
- Circuit Breaker (Resilience4j) → Redis 장애 감지 시 빠르게 fallback
- Fallback 전략: L1 Cache만으로 서빙 / DB 직접 조회 (rate limit 필요)
- Redis Sentinel / Cluster → 자동 failover
- 분산락이 죽으면..?

### 2. Cache Stampede
캐시가 만료되는 순간 다수의 요청이 동시에 DB를 조회하는 문제.
- jitter + algorithm등을 이용한 random ttl (만료분산)
- single flight : 같은 요청을 하나만 실행하고 나머지는 결과를 기다린다 (하나의 pod)
  - 동일요청 dedupe -> 쓰기에는 적용이 어려울듯? 2번쓴거지 중복인건지 모를것 같은데
   -> 어떻게 구현하지?
- distributed lock : 한 요청만 DB를 조회하도록 한다. (분산환경) - 서버간 lock
- [아직모름] stale-while-revalidate (이건 뭐고 어케 구현하지?)

### 3. Cache Invalidation
캐시를 언제, 어떻게 무효화할 것인가.
- TTL 기반 자연 만료
- Write 시 명시적 invalidate (delete or update)
- 이벤트 기반 무효화 (Redis Pub/Sub, Kafka 등)
- L1/L2 다계층 캐시 시, L2만 무효화하면 L1에 stale 남는 문제
  - → Pub/Sub으로 전 Pod L1 동시 무효화 필요

### 4. Write Race Condition
동시 쓰기 시 캐시와 DB 간 정합성이 깨지는 문제.
- DB 업데이트 후 캐시 삭제 vs 캐시 업데이트 → 순서에 따른 race
- 예: A가 DB=100 쓰고 캐시 삭제 전에, B가 DB=200 쓰고 캐시=200 세팅, 그 후 A가 캐시 삭제 → 캐시 miss → DB=200 다시 로드 (이 경우는 OK)
- 예: A가 DB=100 쓰고 캐시=100 세팅 전에, B가 DB=200 쓰고 캐시=200 세팅, 그 후 A가 캐시=100 세팅 → stale!
- 해결: 캐시 업데이트 대신 삭제(invalidate) 전략 선호
- distributed race..?

### 5. Hot Key
특정 키 하나에 트래픽이 집중되는 문제.
- key replication (item:123:1, item:123:2, ...) 읽기분산과 중복쓰기
- LocalCache(L1) 사용으로 Redis 부하 분산
- Redis는 분산 저장 시 데이터(Key)를 해시 슬롯으로 나누어 여러 노드에 분산시킵니다. 하지만 특정 데이터 하나에만 트래픽이 미친 듯이 몰리면 어떻게 될까요?
  - redis cluster에 대해 알아보아요.
  - 인기검색어 인기 상품 등.

### 6. Cache Penetration
존재하지 않는 데이터에 대한 반복 요청으로 캐시를 뚫고 매번 DB를 조회하는 문제.
- Bloom Filter로 존재 여부 사전 체크
- null 결과도 짧은 TTL로 캐싱 (negative caching)
- 요청 패턴 검증 (비정상 요청 필터링)

### 7. Cache Avalanche
대량의 캐시가 동시에 만료되어 DB에 순간 부하가 몰리는 문제.
- TTL에 random jitter 추가하여 만료 시점 분산
- Cache Stampede와 유사하지만, stampede는 단일 키 / avalanche는 다수 키 동시 만료
- 캐시 워밍업 (서버 기동 시 주요 데이터 사전 적재)
- Rate limiting으로 DB 보호
* Cache stampede랑 뭐가 다른가?
- stampede는 하나의 key만료시, 1000 request w
- avalanceh는 대량의 key 만료시, 모든 key 조회 request


### 8. Sharding / L1&L2 Cache & request dedupe(Single flight)
L1/L2 cache 주의점.
---

🔥 캐시 설계의 핵심 철학

실무에서는 보통 이렇게 한다.

DB = source of truth
Redis = eventually consistent

그래서 전략

write → DB
read → cache
update → cache delete

즉

Cache Aside
1 데이터 언제 invalidate 할 것인가
2 동시에 요청 들어오면 어떻게 할 것인가
3 cache와 DB 정합성 어떻게 유지할 것인가

### TODO
- 실제 서비스 캐시 설계에서 나오는 어려운 질문 TOP 7도
- 각 항목별 실무 경험 / 코드 예시 추가

---
title: "[Redis 캐시설계] Cache Stampede와 대응 전략"
date: 2026-03-16 10:00:00 +0900
categories: [Redis, 캐시설계, Stampede]
tags: [redis, cache-stampede, distributed-lock, traffic]
---

> 이 글은 블로그 카테고리 구조 테스트용 예시 포스트입니다.

### Cache Stampede란?

캐시가 만료되는 순간 다수의 요청이 동시에 DB를 조회하는 문제.

### 대응 전략

1. **분산 락** — 1개 요청만 DB 조회, 나머지 대기
2. **TTL Jitter** — 만료 시점 분산
3. **Stale-while-revalidate** — 만료된 캐시를 일단 반환, 백그라운드에서 갱신

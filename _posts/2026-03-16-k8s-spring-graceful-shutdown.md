---
title: "[실무연계] K8s 위에서 Spring을 안전하게 종료하기"
date: 2026-03-16 12:00:00 +0900
categories: [실무연계]
tags: [k8s, spring, graceful-shutdown]
---

> 이 글은 블로그 카테고리 구조 테스트용 예시 포스트입니다.

### 왜 둘 다 설정해야 하는가?

Spring의 `server.shutdown=graceful`만으로는 부족하다.
K8s가 Pod을 종료하는 타이밍과 Spring의 종료 처리가 맞지 않으면 요청이 유실될 수 있다.

### 조합 포인트

- K8s preStop → readinessProbe 제거 → 트래픽 차단
- Spring graceful shutdown → 처리 중인 요청 완료 대기
- 두 타이밍의 순서가 핵심

### 관련 글
- [K8s Pod Lifecycle과 Graceful Shutdown](/posts/k8s-pod-lifecycle/)

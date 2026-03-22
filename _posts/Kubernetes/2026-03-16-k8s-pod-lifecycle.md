---
title: "[Kubernetes 운영] Pod Lifecycle과 Graceful Shutdown"
date: 2026-03-16 11:00:00 +0900
categories: [Kubernetes, 운영]
tags: [k8s, graceful-shutdown, pod-lifecycle]
---

> 이 글은 블로그 카테고리 구조 테스트용 예시 포스트입니다.

### Pod 종료 흐름

1. API Server가 Pod 삭제 요청 수신
2. preStop hook 실행
3. SIGTERM 전송
4. terminationGracePeriodSeconds 대기
5. SIGKILL 강제 종료

### Graceful Shutdown 설정

preStop hook과 readinessProbe를 조합하여 안전한 종료를 보장한다.

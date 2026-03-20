---
title: "서킷브레이커를 직접 테스트했더니, AI가 알려준 동작 방식이 틀렸다"
date: 2026-03-20 00:00:00 +0900
categories: [Development, Architecture]
tags: [resilience4j, circuit-breaker, spring-boot, kotlin, testing, k6, grafana]
---

> **TL;DR** 💡 Resilience4j HALF_OPEN에서 permitted 건수를 다 채우지 않아도, 수학적으로 CLOSED 복귀가 불가능하면 즉시 OPEN으로 전이된다. 공식 문서에 없는 동작이고, AI도 정확히 답하지 못했다. 직접 테스트하고 소스코드를 까봐야 알 수 있었다.
{: .prompt-tip }

(작성 중)

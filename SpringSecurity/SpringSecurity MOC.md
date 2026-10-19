---
up: "[[개발 세컨드 브레인]]"
---
# SpringSecurity MOC

> 핵심 축: 인증은 한 번, 인가는 매번 (신원은 SecurityContextHolder 에 놓인다)

## 읽는 순서

1. [[SpringSecurity]] — 필터 체인이 요청을 가로채는 자리.
2. [[Authentication]] → [[Authorization]] — 신원을 세우고, 그 신원으로 허락을 가린다.
3. [[로그인]] — 위 흐름이 JWT 로 실제로 도는 모습.

## 가지
- [[SpringSecurity]] — 인증/인가 흐름의 입구

#sprout

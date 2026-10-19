---
up: "[[SpringSecurity]]"
---
# Authentication

**"너 누구냐"** 를 확정하는 단계. 결과물은 판정(참/거짓)이 아니라 **신원이 확인된 주체(principal)** 이고, 이걸 어디에 담아 두느냐가 이후 전부를 좌우한다.

## 흐름 — 요청을 가로채 신원을 만든다

[[SpringSecurity]] 는 Filter chain 으로 요청을 가로챈다. 인증은 그 체인 위에서 일어난다.

```
요청 ─▶ 인증 필터 ─▶ 인증 토큰 생성 ─▶ AuthenticationManager 위임
                                          │ 검증 성공
                                          ▼
                                  SecurityContextHolder 에 저장
                                          │
                                          ▼
                                    컨트롤러로 진행
```

폼 로그인이면 `UsernamePasswordAuthenticationFilter` 가 제출된 정보로 토큰을 만들어 `AuthenticationManager` 에게 넘기고, JWT 방식이면 필터가 헤더의 토큰 유효성을 검증한다([[로그인]] 참조). 방식이 달라도 **끝나는 자리는 같다** — `SecurityContextHolder`.

## 저장 위치가 핵심이다

인증이 끝나면 그 신원은 **요청 하나가 사는 동안 계속 참조된다.** 컨트롤러도, [[Authorization]] 판정도 전부 `SecurityContextHolder` 를 본다. 즉 인증 필터의 진짜 산출물은 "통과시켰다" 가 아니라 **"이후 단계가 읽을 수 있는 곳에 신원을 놓았다"** 이다.

그래서 인증이 안 먹는 문제의 상당수는 검증 로직이 아니라 **저장이 안 됐거나, 저장 전에 다른 필터가 먼저 판정한 경우**다. 체인의 순서가 곧 인과다.

## 인증은 한 번, 인가는 매번

인증은 요청 초입에서 한 번 끝난다. 그 뒤로는 "이 신원이 이걸 해도 되나" 만 반복해서 묻는다 — 그게 [[Authorization]] 이다. 둘을 한 덩어리로 보면 "로그인했는데 왜 403 이 나지" 가 설명되지 않는다. **로그인은 신원을 만든 것이지 권한을 준 게 아니다.**

> 상태: #wip — 필터 체인의 구체적 순서와 커스텀 필터를 끼우는 자리는 아직 안 봤다.

> 왜 여기 붙나: 인증은 필터 체인 위에서 일어나 신원을 SecurityContextHolder 에 둔다

## 가지
- [[Authorization]]

#sprout

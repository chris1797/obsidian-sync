
주요 로직에는 회원가입, [[로그인]]이 있음. 로그아웃의 경우 클라이언트 로컬에서 JWT 토큰을 제거함으로써 구현.

Spring Security는 기본적으로 다양한 Filter들의 chain으로 구성되어 있다. 이 Filter chain은 클라이언트의 Request를 가로채어 필요한 절차를 처리한다.

- UsernamePasswordAuthenticationFilter 는 사용자가 제출한 인증 정보를 처리한다.

- UsernamePasswordAuthenticationToken 생성
  : UsernamePasswordAuthenticationFilter는 UsernamePasswordAuthenticationToken을 생성하여 AuthenticationManager에게 전달한다. 이 토큰에는 사용자가 제출한 인증 정보가 포함되어 있다.
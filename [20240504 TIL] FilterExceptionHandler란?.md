## TIL - FilterExceptionHandler란?
---
지금 진행하는 졸업 프로젝트에서는 OAuth 로그인과 JWT 토큰을 적극 활용하고 있다. 우리 서비스, NESS에서는 많은 API에서 ChatGPT를 사용하고 있는 만큼, 회원가입/로그인을 하지 않은 사용자가 우리 서비스를 쓰지 못하게 막아야 했다. 따라서 프로젝트의 초기부터 JWT 토큰 로직을 도입했는데 그만큼 JWT 토큰 관련해서 발생하는 에러 처리를 하는 부분이 필수적이었다.

참고로, JWT는 "JSON Web Token"의 약자로, 정보를 안전하게 전달하기 위한 표준 방법 중 하나이다. 이 토큰은 JSON 형식으로 데이터를 표현하며 정보를 안전하게 전달하기 위해 서명(signature)이나 클래임(claim) 등이 암호화되어 있다. 서버가 발급해준 JWT를 클라이언트가 저장해두었다가 추후 API Call을 할 때 같이 전송한다. 서버에서는 이 토큰을 가지고 사용자 인증 및 권한 부여 로직을 처리할 수 있다.

이때 클라이언트단의 에러, 즉 Expired된 토큰을 클라이언트가 전송했거나 잘못된 서명을 가진 토큰을 전송한 경우는 클라이언트의 에러라는 것을 알려주기 위해서 400 에러를 반환해야 한다. 그러나 기본적으로 토근 Validation에 실패한 경우에는 Internal Server Error, 즉 500 에러가 throw된다. 구체적으로 에러가 발생하는 시점은 바로 아래의 `JwtAuthorizationFilter`에서였다.

```java
@Slf4j
public class JwtAuthorizationFilter extends BasicAuthenticationFilter {
    private JwtTokenProvider jwtTokenProvider;
    private AuthDetailService authDetailService;

    public JwtAuthorizationFilter(AuthenticationManager authenticationManager, JwtTokenProvider jwtTokenProvider, AuthDetailService authDetailService) {
        super(authenticationManager);
        this.jwtTokenProvider = jwtTokenProvider;
        this.authDetailService = authDetailService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        String jwtHeader = request.getHeader("Authorization");
        if (jwtHeader == null || !jwtHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String jwtToken = jwtHeader.substring(7);

        // 여기서 에러 발생
        Member tokenMember = jwtTokenProvider.validJwtToken(jwtToken);

        if(tokenMember != null){
            AuthDetails authDetails = new AuthDetails(tokenMember, Collections.singletonList(new SimpleGrantedAuthority("ROLE_USER")));

            Authentication authentication = new UsernamePasswordAuthenticationToken(authDetails, null, authDetails.getAuthorities());

            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        chain.doFilter(request, response);
    }
}
```

이때 `validJwtToken` 메소드가 던지는 Exception은 다음과 같은 것들이 있다.

```bash
com.auth0.jwt.exceptions.SignatureVerificationException: The Token's Signature resulted invalid when verified using the Algorithm: HmacSHA512
	at com.auth0.jwt.algorithms.HMACAlgorithm.verify(HMACAlgorithm.java:56) ~[java-jwt-3.14.0.jar:3.14.0]
	at com.auth0.jwt.JWTVerifier.verify(JWTVerifier.java:299) ~[java-jwt-3.14.0.jar:3.14.0]
	at com.auth0.jwt.JWTVerifier.verify(JWTVerifier.java:283) ~[java-jwt-3.14.0.jar:3.14.0]
	at Ness.Backend.domain.auth.jwt.JwtTokenProvider.getAuthKeyClaim(JwtTokenProvider.java:103) ~[main/:na]
	at Ness.Backend.domain.auth.jwt.JwtTokenProvider.validJwtToken(JwtTokenProvider.java:130) ~[main/:na]
	...
```

```bash
com.auth0.jwt.exceptions.TokenExpiredException: The Token has expired on Mon Mar 25 16:07:54 KST 2024.
	at com.auth0.jwt.JWTVerifier.assertDateIsFuture(JWTVerifier.java:420) ~[java-jwt-3.14.0.jar:3.14.0]
	at com.auth0.jwt.JWTVerifier.assertValidDateClaim(JWTVerifier.java:411) ~[java-jwt-3.14.0.jar:3.14.0]
	at com.auth0.jwt.JWTVerifier.verifyClaimValues(JWTVerifier.java:331) ~[java-jwt-3.14.0.jar:3.14.0]
	at com.auth0.jwt.JWTVerifier.verifyClaims(JWTVerifier.java:315) ~[java-jwt-3.14.0.jar:3.14.0]
	at com.auth0.jwt.JWTVerifier.verify(JWTVerifier.java:300) ~[java-jwt-3.14.0.jar:3.14.0]
	...
```

이 에러들을 400 코드로 바꿔주기 위해서는 직접 Exception을 catch 한 후, Handler에 넘겨줘서 바로 return되어야 한다. 그런데 여기서 문제는 JWT 토큰 에러는 Spring MVC까지 도달하지 않고, 필터에서 바로 throw되는 에러라는 점이다. 아래 그림을 참고하면 JWT 토큰이 Controller까지 도착하지 않고 Servlet Filter에서 던져진 에러가 바로 500으로 클라이언트에게 도달한다는 것을 알 수 있다.

<img width="100%" alt="jwt token error" src="https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/b52d74b8-9287-4503-b9cb-f8ee7970d2bb">

결국 이 문제를 해결하기 위해서는 필터에서 catch한 에러를 Handler가 받아서 HttpServletResponse에 바로 write해야 한다.

## QnA
---



## 레퍼런스
---
- [stackoverflow의 exceptions-thrown-in-filters](https://stackoverflow.com/questions/34595605/how-to-manage-exceptions-thrown-in-filters-in-spring)
- [javadoc의 com.auth0.jwt.exceptions](https://javadoc.io/doc/com.auth0/java-jwt/3.4.1/com/auth0/jwt/exceptions/package-summary.html)

## TIL - FilterExceptionHandler란?
---
### 기술적 문제 상황
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

이 에러들을 400 코드로 바꿔주기 위해서는 직접 Exception을 catch 한 후, Handler에 넘겨줘서 바로 return되어야 한다. 그런데 여기서 문제는 JWT 토큰 에러는 `Spring MVC`까지 도달하지 않고, 필터에서 바로 throw되는 에러라는 점이다. 아래 그림을 참고하면 JWT 토큰이 Controller까지 도착하지 않고 `Servlet Filter`에서 던져진 에러가 바로 500으로 클라이언트에게 도달한다는 것을 알 수 있다.

<img width="100%" alt="jwt token error" src="https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/b52d74b8-9287-4503-b9cb-f8ee7970d2bb">

결국 이 문제를 해결하기 위해서는 필터에서 catch한 에러를 Handler가 받아서 `HttpServletResponse`에 바로 write해야 한다. 즉, 기존의 ApiResponse에 ErrorCode를 담아서 리턴하는 방식은 일단 에러가 컨트롤러에 도달해야 사용할 수 있는 것이고, 여기서는 `HttpServletResponse`에 바로 JSON으로 write해야 클라이언트에게 에러 코드를 리턴해줄 수 있다.

### Step 1 - 에러 Catch
먼저 발생하는 에러를 catch하는 로직을 `JwtAuthorizationFilter`에 추가해야 한다. 여기서 중요한 것은 실제 에러가 발생하는 `jwtTokenProvider.validJwtToken(jwtToken)`뿐만 아니라 `chain.doFilter(request, response)`까지도 반드시 try 문에 포함해야 한다는 것이다. `doFilter`는 `SecurityFilterChain`에 등록된 다음 필터에게 요청/응답 쌍을 보내주는 메소드이므로, 이 코드를 try 문에 포함하지 않으면 에러가 발생해도 다음 필터로 자연스럽게 넘어가기 때문에 에러가 발생해도 200 OK 응답코드를 받게 되는 특이한 일이 생긴다.  

```java
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        String jwtHeader = request.getHeader("Authorization");
        if (jwtHeader == null || !jwtHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String jwtToken = jwtHeader.substring(7);

        try {
            // 여기서 에러 발생
            Member tokenMember = jwtTokenProvider.validJwtToken(jwtToken);

            if(tokenMember != null){
                AuthDetails authDetails = new AuthDetails(tokenMember, Collections.singletonList(new SimpleGrantedAuthority("ROLE_USER")));

                Authentication authentication = new UsernamePasswordAuthenticationToken(authDetails, null, authDetails.getAuthorities());

                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
            // doFilter 반드시 포함
            chain.doFilter(request, response);

        } catch (TokenExpiredException e){
            setResponse(response, ErrorCode.EXPIRED_TOKEN);
        } catch (SignatureVerificationException e){
            setResponse(response, ErrorCode.INVALID_TOKEN_SIGNATURE);
        }
    }
}
```

### Step 2 - Handler에 에러 넘겨주기
위에서 catch한 에러는 핸들러에 넘겨줘서 처리 가능하게 해야 한다. 핸들러는 `HttpServletResponse`에 직접적으로 `ErrorCode`를 write해주는 역할을 한다. 이때 서블릿인 `HttpServletResponse`가 기본적으로 가지고 있는 `setContentType()`, `setStatus()`, `getWriter().print()` 등을 사용하면 엔티티나 DTO를 사용하지 않아도 HTTP Status를 세팅하고 응답을 클라이언트에게 보내주는 게 가능하다.
```java
public class FilterExceptionHandler {
    public static void setResponse(HttpServletResponse response, ErrorCode errorCode) throws IOException {
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.getWriter().print(ApiResponse.jsonOf(errorCode));
    }
}
```

### Step 3 - 
위에서 본 `response.getWriter()`는 클라이언트에게 텍스트 데이터를 보낼 수 있는 `PrintWriter` 객체를 반환하고, `print()`는 `PrintWriter` 클래스의 메서드로 텍스트를 출력할 수 있게 해준다. 이때 서블릿 컨테이너는 자동으로 객체를 JSON 직렬화해주므로, `JSONObject`를 리턴해도 된다. 나는 `ApiResponse` 클래스 안에 `JSONObject`로 에러 코드 등을 직접 반환하는 `jsonOf` 메소드를 만들어서 사용하였다.

```java
    public static JSONObject jsonOf(ErrorCode errorCode) {
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("timestamp", LocalDateTime.now().format(DateTimeFormatter.ISO_DATE_TIME));
        jsonObject.put("success", false);
        jsonObject.put("message", errorCode.getMessage());
        jsonObject.put("status", errorCode.getHttpStatus().value());
        jsonObject.put("code", errorCode.getCode());
        return jsonObject;
    }
```

### 트러블 슈팅

처음에는 아래 코드와 같이 `setResponse`를 필터 안에 같이 제공했었다.

```java
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        String jwtHeader = request.getHeader("Authorization");
        if (jwtHeader == null || !jwtHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String jwtToken = jwtHeader.substring(7);

        try {
            Member tokenMember = jwtTokenProvider.validJwtToken(jwtToken);

            if(tokenMember != null){
                AuthDetails authDetails = new AuthDetails(tokenMember);

                Authentication authentication = new UsernamePasswordAuthenticationToken(authDetails, null, authDetails.getAuthorities());

                SecurityContextHolder.getContext().setAuthentication(authentication);
           }

        } catch (TokenExpiredException e){
            setResponse(response, ErrorCode.EXPIRED_TOKEN);
        }
        chain.doFilter(request, response);
    }

    private void setResponse(HttpServletResponse response, ErrorCode errorCode) throws RuntimeException, IOException {
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(errorCode.getHttpStatus());
        response.getWriter().print(errorCode.getMessage());
    }
```

그랬더니 아래와 같은 에러가 발생하였다. 이 에러는 이미 `getWriter()`가 호출되어 응답이 시작되었는데, 다시 `getWriter()`나 `getOutputStream()` 메서드를 호출하여 응답을 변경하려고 할 때 발생한다. 

```java
2024-04-03T15:31:33.936+09:00 ERROR 24192 --- [nio-8080-exec-1] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed: java.lang.IllegalStateException: getWriter() has already been called for this response] with root cause

java.lang.IllegalStateException: getWriter() has already been called for this response
	at org.apache.catalina.connector.Response.getOutputStream(Response.java:518) ~[tomcat-embed-core-10.1.19.jar:10.1.19]
	at org.apache.catalina.connector.ResponseFacade.getOutputStream(ResponseFacade.java:179) ~[tomcat-embed-core-10.1.19.jar:10.1.19]
	at jakarta.servlet.ServletResponseWrapper.getOutputStream(ServletResponseWrapper.java:100) ~[tomcat-embed-core-10.1.19.jar:6.0]
	at jakarta.servlet.ServletResponseWrapper.getOutputStream(ServletResponseWrapper.java:100) ~[tomcat-embed-core-10.1.19.jar:6.0]
	at jakarta.servlet.ServletResponseWrapper.getOutputStream(ServletResponseWrapper.java:100) ~[tomcat-embed-core-10.1.19.jar:6.0]
	at org.springframework.security.web.util.OnCommittedResponseWrapper.getOutputStream(OnCommittedResponseWrapper.java:146) ~[spring-security-web-6.2.2.jar:6.2.2]
	...
```

이게 `chain.doFilter(request, response)`를 반드시 try 문에 포함시켜야 하는 또다른 이유인데, 이 메소드 또한 `getWriter()`를 호출하기 때문에 `setResponse()`가 호출하는 `getWriter()`와 이중 커밋이 되어서 에러가 난다. 따라서 다음 필터로 넘어가지 못하게 하기 위해서 + `getWriter()` 호출을 막기 위해서 꼭 이 메소드를 try 문 안에 넣어서 에러가 나면 중단되도록 해야 한다. 쉽게 말하자면, `에러 발생이 가능한 코드~chain.doFilter()가 있는 코드`까지 전부 try 문에 있어야 한다.

## QnA
---
(스터디 이후 채워질 예정)

## 레퍼런스
---
- [stackoverflow의 exceptions-thrown-in-filters](https://stackoverflow.com/questions/34595605/how-to-manage-exceptions-thrown-in-filters-in-spring)
- [javadoc의 com.auth0.jwt.exceptions](https://javadoc.io/doc/com.auth0/java-jwt/3.4.1/com/auth0/jwt/exceptions/package-summary.html)
- [stackoverflow의 how-to-properly-handle-jwtexception](https://stackoverflow.com/questions/64015805/how-to-properly-handle-jwtexception)

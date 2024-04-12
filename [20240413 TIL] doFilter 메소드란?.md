## TIL - doFilter 메소드에 대해서
---
개발하면서 매우 궁금했던 게 바로 `chain.doFilter(request, response);` 메소드였다. `JwtAuthorizationFilter`에 존재하는 이 `doFilter`는 도대체 무엇을 하는 얘일까? [내 첫 번째 TIL](https://github.com/JeonHaeseung/TodayILearnStudy/blob/main/%5B20240401%20TIL%5D%20%EC%8A%A4%ED%94%84%EB%A7%81%20%EC%84%9C%EB%B8%94%EB%A6%BF(Servlet)%20%EC%9D%B4%EB%9E%80%3F.md)에서 공부했던 게 바로 `doFilter` 안의 인자로 들어가는 `HttpServletRequest request`와 `HttpServletResponse response`였는데, 얘들은 일종의 요청과 응답을 나타내는 객체라는 것을 알게되었다. 그러나 이제 궁금한 점은 그렇다면 요청과 응답을 동시에 가지고 있는 `doFilter` 메소드는 무엇을 하는 것 인지였다.

oracle에 따르면, 필터는 리소스(서블릿 또는 정적 콘텐츠)에 대한 요청이나 리소스의 응답, 또는 둘 다에 대해 필터링 작업을 수행하는 객체라고 한다. 여기서 약간의 힌트를 얻을 수 있다. 우리는 JWT 토큰을 request에 대해서만 검증하면 되긴 하지만, 일반적으로 필터의 일종인 `JwtAuthorizationFilter`은 리소스의 요청/응답 모두에 필터링을 할 수 있는 객체인 것이다. 조금 더 구체적인 메소드의 동작 방식은 다음과 같다:
1. request를 검토한다.
2. 선택적으로 request 객체(`HttpServletRequest request`)를 사용자 정의 구현으로 래핑하여 입력에서 contents 또는 header를 필터링한다.
3. 선택적으로 response 객체(`HttpServletResponse response`)를 사용자 정의 구현으로 래핑하여 출력에서 콘텐츠 또는 헤더를 필터링한다.
4. FilterChain 개체(`chain.doFilter()`)를 사용하여 체인의 다음 엔터티를 호출하거나, 또는 request 처리를 차단하기 위해 request/response 쌍을 필터 체인의 다음 엔터티로 전달하지 않는다.
5. 필터 체인에서 다음 엔터티를 호출한 후 응답에 헤더를 직접 설정한다.

나는 아래와 같은 `JwtAuthorizationFilter` 클래스에서 `doFilter`를 호출했었다.
``` java
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        /* 헤더 추출 및 정상적인 헤더인지 확인 */
        String jwtHeader = request.getHeader("Authorization");
        if (jwtHeader == null || !jwtHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        /* 헤더 안의 JWT 토큰을 검증해 정상적인 사용자인지 확인 */
        String jwtToken = jwtHeader.substring(7);

        try {
            Member tokenMember = jwtTokenProvider.validJwtToken(jwtToken);

            if(tokenMember != null){ //토큰이 정상일 경우
                AuthDetails authDetails = new AuthDetails(tokenMember);

                /* JWT 토큰 서명이 정상이면 Authentication 객체 생성 */
                Authentication authentication = new UsernamePasswordAuthenticationToken(authDetails, null, authDetails.getAuthorities());

                /* 시큐리티 세션에 Authentication 을 저장 */
                SecurityContextHolder.getContext().setAuthentication(authentication);
           }

            chain.doFilter(request, response);

        } catch (TokenExpiredException e){
            log.error(e + " EXPIRED_TOKEN");
            //request.setAttribute("exception", ErrorCode.EXPIRED_TOKEN.getCode());
            setResponse(response, ErrorCode.EXPIRED_TOKEN);
        } catch (SignatureVerificationException e){
            log.error(e + " INVALID_TOKEN_SIGNATURE");
            setResponse(response, ErrorCode.INVALID_TOKEN_SIGNATURE);
        }
    }
```
그리고 다음과 같은 `SecurityConfig`에서 필터 체인을 정의했었다.
```java
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
        httpSecurity
                .csrf(csrf -> csrf.disable()) //악의적인 공격 방어를 위해서 CSRF 토큰 사용 비활성화
                .cors(cors -> cors.configurationSource(corsConfigurationSource())) //CORS 설정
                .formLogin(formLogin -> formLogin.disable()) //폼 기반 로그인을 비활성화->토큰 기반 인증 필요
                .httpBasic(httpBasic -> httpBasic.disable()) //HTTP 기본 인증을 비활성화->비밀번호를 평문으로 보내지 않음
                .sessionManagement(c -> c.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) //세션을 생성하지 않음->토큰 기반 인증 필요
                .addFilter(new JwtAuthenticationFilter(authenticationManager(), jwtTokenProvider(), refreshTokenService))  //사용자 인증
                .addFilter(new JwtAuthorizationFilter(authenticationManager(),  jwtTokenProvider(), authDetailService)) //사용자 권한 부여
                .authorizeHttpRequests(requests -> requests
                        .dispatcherTypeMatchers(DispatcherType.FORWARD).permitAll()
                        .anyRequest().permitAll()	//개발 환경: 모든 종류의 요청에 인증 불필요
                );
        return httpSecurity.build();
    }
```
이 코드에 따르면, `doFilter`은 `JwtAuthorizationFilter`의 로직을 종료하고 필터 체인의 다음 엔티티로 넘겨주는 역할을 한다. 이러한 디자인 패턴을 Chain-of-responsibility 패턴이라고 한다. 이 체인이 종료되어야지만 요청/응답 쌍이 원래 목표했던 타겟 리소스/서블릿으로 이동이 가능한 것이다. 아마 DB와 긴밀하게 연관되어 있는 타겟 리소스/서블릿으로 이동하기 전이나 후에 데이터의 유효성을 검증하고 싶기 때문에 필터라는 것을 만든 것 같다. 프론트엔드에도 필터를 만들 수 있지만, JS 비활성화로 간단하게 무시될 수도 있기 때문이다. 그래서 필터는 아래 그림에서 볼 수 있듯이, `DispatcherServlet`으로 이동하기 전에 작동한다.
<img src=https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/97c5f01b-e108-4bcf-b2cf-6a8a7ec1bb3a width="100%">

여기서 `HttpServletResponse response`가 무엇인지를 잠깐 보자. oracle에 따르면 이는 HTTP 관련 기능을 제공하도록 servlet에 의해 생성된 객체이며 doGet, doPost 등의 메소드에 전달되는 인수라고 한다. 상태 코드를 비롯해서, `addCookie`나 `addHeader` 등의 다양한 메소드를 포함하고 있다. 
 

## QnA
---
(스터디 이후 채워질 예정)

## 레퍼런스
---
- [oracle의 Filter](https://docs.oracle.com/javaee%2F6%2Fapi%2F%2F/javax/servlet/Filter.html)
- [stackoverflow의 what-is-chain-dofilter-doing-in-filter-dofilter-method](https://stackoverflow.com/questions/2057607/what-is-chain-dofilter-doing-in-filter-dofilter-method)
- [geeksforgeeks의 java-servlet-filter-with-example](https://www.geeksforgeeks.org/java-servlet-filter-with-example/)
- [docs.spring의 spring-security architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
- [linkedin의 filters-interceptors-spring-boot](https://www.linkedin.com/pulse/demystifying-filters-interceptors-spring-boot-ali-as-ad--0mdlf)
- [tistory의 스프링 필터의 동작과정](https://emgc.tistory.com/125)
- [oracle의 HttpServletResponse](https://docs.oracle.com/javaee/6/api/javax/servlet/http/HttpServletResponse.html)
- [docs.spring의 DefaultErrorAttributes](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/web/servlet/error/DefaultErrorAttributes.html)
- [[docs.spring의 HandlerExceptionResolver](https://docs.spring.io/spring-framework/docs/6.1.5/javadoc-api/org/springframework/web/servlet/HandlerExceptionResolver.html)

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
이렇게 스터디를 진행했는데 다음과 같은 질문을 받았다.
> 그럼 filter가 SecurityFilterChain에 addFilter 메소드로 등록된다는 건 알겠는데, SecurityFilterChain는 어떻게 스프링에 등록되나요? 자동으로 실행 가능하게 등록되는 건가요?

이 부분은 나도 잘 모르겠어서 추가로 찾아봐야겠다는 생각이 들었다. 우선 `SecurityFilterChain`의 위에 `@Bean`이 있으므로 실행 가능하게 만들어지는 이유는 명확하다. 여기서 이게 중요한 것은 어떻게 이게 가장 먼저 돌아가는지를 스프링이 아는지이다.
찾아보니, 스프링 시큐리티는 서비스 설정(Configuration)에 라서 filter를 순서대로 실행할 수 있다고 한다. 서블릿 필터를 사용할 때는 당연히 web.xml에 해당 필터들을 선언해야 한다(그렇지 않으면 서블릿 컨테이너에 의해서 무시된다). web.xml에 선언된 `DelegatingFilterProxy`이 바로 web.xml과 애플리케이션 컨텍스트 사이의 연결을 제공하는 역할을 한다.
```xml
<filter> 
<filter-name> myFilter </filter-name> 
<filter-class> org.springframework.web.filter.DelegatingFilterProxy </filter-class> 
</filter>

<filter-mapping> 
<filter-name> myFilter </filter-name> 
<url-pattern> /* </url-pattern> 
</filter-mapping>
```

`DelegatingFilterProxy`가 하는 일은 Spring 애플리케이션 컨텍스트에서 가져온 빈(`springSecurityFilterChain`)을 통해 필터의 메서드를 위임하는 것이다. Spring Security의 웹 인프라는 `FilterChainProxy`의 인스턴스로 위임함으로써 사용되어야 한다. 물론 필요한 각 Spring Security 필터 빈을 애플리케이션 컨텍스트 파일에 선언하고, 각 필터에 대한 해당 DelegatingFilterProxy 항목을 모두 web.xml에 추가해줄 수 있겠지만, 이는 너무 번거롭다. 이때 `FilterChainProxy`를 사용하면 web.xml에 단일 항목만 추가해도 웹 보안 빈을 관리하기 위해 완전히 애플리케이션 컨텍스트 파일을 처리할 수 있다. 이는 앞서 언급한 예제와 마찬가지로 DelegatingFilterProxy를 사용하여 연결되지만, filter-name이 빈 이름 "filterChainProxy"로 설정된다. 그런 다음 아래와 같이 필터 체인은 동일한 빈 이름으로 애플리케이션 컨텍스트에 선언된다:
```xml
<bean id="filterChainProxy" class="org.springframework.security.web.FilterChainProxy">
<constructor-arg>
	<list>
	<sec:filter-chain pattern="/restful/**" filters="
		securityContextPersistenceFilterWithASCFalse,
		basicAuthenticationFilter,
		exceptionTranslationFilter,
		filterSecurityInterceptor" />
	<sec:filter-chain pattern="/**" filters="
		securityContextPersistenceFilterWithASCTrue,
		formLoginFilter,
		exceptionTranslationFilter,
		filterSecurityInterceptor" />
	</list>
</constructor-arg>
</bean>
```
정리하자면 다음과 같다.
1. web.xml에 선언된 DelegatingFilterProxy은 서블릿 컨테이너의 라이프사이클(web.xml)과 애플리케이션 컨텍스트 사이의 연결을 제공한다. 즉, ApplicationContext에서 Filter의 Bean을 찾아서 호출할 수 있다.
2. FilterChainProxy는 SecurityFilterChain을 통해 많은 필터 인스턴스 목록에 위임을 허용하는 특별한 필터이다. FilterChainProxy는 Bean이므로 일반적으로 DelegatingFilterProxy에 래핑 가능하다.
3. SecurityFilterChain은 순서대로 필터를 호출한다. 호출된 필터는 이렇게 작동된다.
   a. request를 검토한다.
   b. 선택적으로 request 객체(`HttpServletRequest request`) 또는 response 객체(`HttpServletResponse response`)를 사용자 정의 구현으로 래핑하여 입력/출력에서 contents 또는 header를 필터링한다.
   c. FilterChain 개체(`chain.doFilter()`)를 사용하여 체인의 다음 엔터티를 호출하거나, 또는 request 처리를 차단하기 위해 request/response 쌍을 필터 체인의 다음 엔터티로 전달하지 않는다.
   d. 하나의 필터가 끝나면 필터 체인에서 다음 엔터티를 호출한 후 응답에 헤더를 직접 설정한다.
그림으로 표현하자면 아래와 같이 적용된다:
<img src=https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/4137f38c-4815-400b-b5c9-d422c5d6afb9 width="100%">


## 레퍼런스
---
### Filter
- [oracle의 Filter](https://docs.oracle.com/javaee%2F6%2Fapi%2F%2F/javax/servlet/Filter.html)
- [stackoverflow의 what-is-chain-dofilter-doing-in-filter-dofilter-method](https://stackoverflow.com/questions/2057607/what-is-chain-dofilter-doing-in-filter-dofilter-method)
- [geeksforgeeks의 java-servlet-filter-with-example](https://www.geeksforgeeks.org/java-servlet-filter-with-example/)
- [docs.spring의 spring-security architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html)
- [linkedin의 filters-interceptors-spring-boot](https://www.linkedin.com/pulse/demystifying-filters-interceptors-spring-boot-ali-as-ad--0mdlf)
- [tistory의 스프링 필터의 동작과정](https://emgc.tistory.com/125)
### HttpServletResponse 및 에러 핸들
- [oracle의 HttpServletResponse](https://docs.oracle.com/javaee/6/api/javax/servlet/http/HttpServletResponse.html)
- [docs.spring의 DefaultErrorAttributes](https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/web/servlet/error/DefaultErrorAttributes.html)
- [docs.spring의 HandlerExceptionResolver](https://docs.spring.io/spring-framework/docs/6.1.5/javadoc-api/org/springframework/web/servlet/HandlerExceptionResolver.html)
### SecurityFilterChain
- [github.io의 spring-filter-chain](https://thecodinglog.github.io/spring/2020/04/28/spring-filter-chain.html)
- [docs.spring의 security-filter-chain](https://docs.spring.io/spring-security/site/docs/4.2.x/reference/html/security-filter-chain.html)
- [baeldung의 spring-delegating-filter-proxy](https://www.baeldung.com/spring-delegating-filter-proxy)
- [velog의 Spring-Security](https://velog.io/@sodliersung/Spring-Security)

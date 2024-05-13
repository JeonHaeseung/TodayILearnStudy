## TIL - @ConfigurationProperties 어노테이션에 대해서(feat. @Value 쓰지 말기)
---
OAuth 개발을 하다가 다음과 같이 로직을 짜야하는 상황이 생겼다. 카카오, 구글, 네이버 로그인을 다 지원하는 서비스인데, 문제는 OAuth라는 것은 표준 작동 방식이 있다 보니 각 로그인이 전부 똑같은 로직을 따른다. 로직이 같은데도 불구하고 카카오, 구글, 네이버 로그인마다 서로 다른 메소드를 만들기가 싫어서 switch 문으로 구분했다.
```java
private String getOAuthAccessToken(String authorizationCode, String registration) {
    ...(생략)...
    switch (registration) {
        case "google":
            accessToken = googleOAuthApi
                    .googleGetToken(authorizationCode, clientId, clientSecret,
                            redirectUri,
                            "authorization_code")
                    .getAccessToken();
            break;
        case "kakao":
            accessToken = kakaoOAuthApi
                    .kakaoGetToken(authorizationCode, clientId, clientSecret,
                            redirectUri,
                            "authorization_code")
                    .getAccessToken();
            break;
        case "naver":
            String state = env.getProperty("oauth2." + registration + ".state");
            accessToken = naverOAuthApi
                    .naverGetToken(authorizationCode, clientId, clientSecret,
                            URLEncoder.encode(redirectUri, StandardCharsets.UTF_8),
                            "authorization_code",
                            URLEncoder.encode(state, StandardCharsets.UTF_8))
                    .getAccessToken();
            break;
    }
    return accessToken;
}
```
이 코드가 클린하지 않은 것은 둘째치고, 나를 가장 괴롭힌 건 application.yml에서 설정 값을 가져오는 문제였다. 로직은 같은데 switch 문에 따라서 서로 다른 설정 값을 가져와야 하는데, 이를 static하지 않고 동적으로 결정되게 만들고 싶었다. 이 리팩토링 시행착오 끝에 만들어진 코드를 공유하고자 오늘 글을 작성하였다. 

### 시도 1 - @Value 어노테이션
먼저 기존에 자주 사용하던 방식대로 `@Value` 어노테이션을 만들어보았다.
```
@Value("${oauth2.google.client-secret}")
private String googleClientSecrete;

@Value("${oauth2.kakao.client-secret}")
private String kakaoClientSecrete;

@Value("${oauth2.naver.client-secret}")
private String naverClientSecrete;

...(생략)...
```
위에서도 쉽게 보이듯이, `@Value` 다음과 같은 문제가 있었다.
1. 코드의 중복: 똑같은 역할을 하는 코드를 6줄 써야 한다는 것이 불편했고, googleClientSecrete, kakaoClientSecrete, naverClientSecrete 등 서로 다른 변수명을 남발해야 한다는 점이 불편했다. 변수명이 이렇게 달라지면 코드의 재활용이 힘들어진다.
2. 동적으로 결정되지 않음: @Value의 경우, application.yml 에서 읽어올 값의 경로를 다 결정해 줘야 한다. 뭔가 동적으로 결정할 수 있는 부분이 아니다.  Spring container life cycle에 의해서 @Autowired @Value와 같은 주입이 전부 이루어져야 서비스 접근이 가능하기 때문이다.
3. 설정과 Service 로직이 분리되지 않음: 이게 좀 굉장히 찝찝한 부분이었는데, @Configuration 안에 들어가야 할 내용이 이렇게 @Service에 있다는 점이 별로였다. 물론 따로 클래스를 만들어서 분리할 수 있지만, 다른 어떤 기능도 없고 단순히 @Value로 읽어오는 필드만 있는 클래스를 getter로 접근해서 값을 가져오는 것도 좋은 설계가 아니라고 생각했다.
   
### 시도 2 - Environment 사용
Environment는 application.yml의 설정 값을 읽어오는 객체로, 현재 애플리케이션이 실행 중인 환경을 나타내는 인터페이스이다. profiles과 properties 등을 모델링하는 객체라고 생각하면 될 것 같다.
```java
private final Environment env;

private String getOAuthAccessToken(String authorizationCode, String registration) {
    String clientId = env.getProperty("oauth2." + registration + ".client-id");
    String clientSecret = env.getProperty("oauth2." + registration + ".client-secret");
    String redirectUri = env.getProperty("oauth2." + registration + ".redirect-uri");
    String accessToken = null;
    ...(생략)...
}
```
위에서 `@Value`를 썼을 때와 달리 이제는 `registration` 값이 google/kakao/naver인지에 따라서 결정될 수 있다.
그런데 여전히 `Environment`도 다음과 같은 문제점이 있었다.
1. 객체 지향적이지 않음: Environment는 먼저 설정값을 전부 키-값 구조로 읽어들인 다음, 키를 통해서 원하는 값을 찾아가는 구조이다. 그런데 이렇게 키를 통해서 원하는 값을 명시해주는 방식은 위의 `@Value` 어노테이션과 별 다를 게 없다.
2. 동적으로 결정되었다고 하기에는 찝찝함: 위의 코드를 보면 `registration` 값을 `+` 기호를 통해서 문자열과 연결하여 올바른 키를 만들어주는 구조이다. 자료구조의 특징을 사용했다기 보다는 단순히 문자열을 조합하는 느낌이라서 뭔가 굉장히 찝찝한 느낌이 있다.
3. 필요 없는 값까지 접근 가능 함: 위의 코드에서 설정된 env는 getProperty를 통해서 원하는 설정 값을 모두 읽어올 수 있다는 점이 장점이자 단점이다. 코드에서 필요한 부분만큼의 값을 읽어오는 게 아니라 모든 설정 값들이 노출 가능한 인터페이스라는 점이 위험할 수도 있겠다는 생각이 들었다.

### 시도 3 - @ConfigurationProperties 사용
이런 고민을 하던 도중 medium에서 어떻게 알았는지 "why you should stop using value annotations"라는 글을 메일로 읽어보라고 알려줘서 @ConfigurationProperties의 존재를 알게 되었다. @ConfigurationProperties는 좀 더 객체지향적인 방식으로 설정 값을 접근할 수 있는 방식이다.
예를 들어서, 다음과 같은 application.yml 값이 있다고 가정해보자.
``` yml
spring:
  cloud:
    openfeign:
      client:
        config:
          api1:
            url: https://도메인1.com
          api2:
            url: http://도메인2.com
```
그럼 @ConfigurationProperties을 사용하면 다음과 같이 자료구조를 이용해서 객체지향적으로 값을 읽어올 수 있다.
``` java
@Configuration
@ConfigurationProperties(prefix = "spring.cloud.openfeign.client.config")
public class FeignClientConfigProperties {
    private Map<String, FeignClientConfig> clients;
}
```
```java
public static class FeignClientConfig {
    private String url;
}
```
이제 외부에서 어떤 feign client를 사용하고 싶다면, Map의 key로 그 클라이언트명을 입력해주기만 하면 된다. @ConfigurationProperties는 이렇게 설정 값을 자료구조 적으로 다룰 수 있게 만들 뿐만 아니라 더 다양한 기능을 제공한다. 예를 들어서, 정말 정확한 URL 값이 맞는지 확인해 보고 싶다면 @Pattern을 사용해서 정규식 검사도 가능하다.
```java
public static class FeignClientConfig {
    @Pattern(regexp = "^(https?|ftp):\/\/[^\s/$.?#].[^\s]*$")
    private String url;
}
```

### 결론
결론적으로, OAuth 로직을 직접 구현하고자 하다가 Spring Security에 이미 있는 로직으로 전부 변경하는 바람에 위의 코드는 필요가 없어지게 되었다(스프링 시큐리티를 쓰는 게 클라이언트 단의 부담이 훨씬 적었다). 그러나 @Value 대신 @ConfigurationProperties을 쓰는 것은 자료구조적으로 설정 값을 접근할 수 있는 좋은 방법인 것 같다.

## 레퍼런스
---
- [medium의 why-you-should-stop-using-value-annotations](https://medium.com/@mikael_55667/why-you-should-stop-using-value-annotations-in-spring-and-use-this-instead-2c8a47e5096a)
- [relentlesscoding의 dynamically-inject-values](https://relentlesscoding.com/posts/spring-basics-dynamically-inject-values-with-springs-value/)
- [velog의 Spring-Boot-Environment](https://velog.io/@hellonewtry/Spring-Boot-Environment%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-properties-%EA%B0%92-%EC%89%BD%EA%B2%8C-%EA%B0%80%EC%A0%B8%EC%98%A4%EA%B8%B0)
- [spring.io의 Environment](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/env/Environment.html)
- [blog의 spring-configuration-properties-fetch-application-properties](https://blog.yevgnenll.me/posts/spring-configuration-properties-fetch-application-properties)

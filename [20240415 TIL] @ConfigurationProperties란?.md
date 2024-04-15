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
private String kakaoClientSecrete;

...(생략)...
```
위에서도 쉽게 보이듯이, `@Value` 다음과 같은 문제가 있었다.
1. 코드의 중복: 
2. 동적으로 결정되지 않음: 
3. 설정과 Service 로직이 분리되지 않음: 이게 좀 굉장히 찝찝한 부분이었는데, 
   
### 시도 2 - Environment 사용
Environment는 application.yml의 설정 값을 읽어오는 
```java
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
3. 여전히 설정과 Service 로직이 분리되지 않음: 물론 Class를 따로 만들어서 그 안에서 설정 값을 가져오게 해도 되지만, 그렇게 분리한다고 해도 객체지향적으로 값에 접근하게 되는 게 아니라 여전히 method를 통해서 값에 접근해야 한다(`registration`를 파라미터로 전달해주어야 하므로).
4. 필요 없는 값까지 읽어들어야 함: 

## QnA
---
(스터디 이후 채워질 예정)

## 레퍼런스
---
- [medium의 why-you-should-stop-using-value-annotations](https://medium.com/@mikael_55667/why-you-should-stop-using-value-annotations-in-spring-and-use-this-instead-2c8a47e5096a)
- [blog의 spring-configuration-properties-fetch-application-properties](https://blog.yevgnenll.me/posts/spring-configuration-properties-fetch-application-properties)

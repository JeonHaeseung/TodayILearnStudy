### TIL - 서버 에러에 Discord Webhook 도입하기
---
### 도입 계기
빠르게 개발을 진행하게 되면서 클라이언트 개발자에게서 500에러 났으니 확인 부탁한다는 이야기를 듣는 경우가 많아졌는데, 이때마다 docker logs로 콘솔 에러 로그를 확인하는게 너무 번거로웠다. 기존에 이미 `ExceptionHandler`를 만들어서 일반적인 500 Internal Server Error일 경우 에러 메세지를 보내주긴 했는데, 나는 이렇게 일괄적인 메세지 말고도 진짜 콘솔의 에러 로그를 바로 보고 싶었다.
그런데 문제는 이렇게 `ErrorResponse`로 바로 콘솔 에러를 클라이언트에게 보내줄 경우, 클라이언트 개발자 뿐만 아니라 
### Discord Webhook 구성
먼저 Webhook을 만들어야 한다. 우리 팀은 디스코드를 팀 소통 툴로 쓰고 있기 때문에, 디스코드 팀 워크스페이스에 🥹서버-에러 채널을 하나 만들었다. 그리고 나서 채널 설정 > 연동 > 웹후크에서 새로운 Webhook를 만들어주면 된다.
<img style="width: 100%;" src="https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/e224a474-ae17-456a-9b00-c9bc06fc31e2">
그럼 아래와 같이 간단하게 내가 메세지를 보낼 수 있는 엔드포인트가 생성된다. 
<img style="width: 100%;" src="https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/89163175-2bf7-4ceb-864c-6d3850a6d9fb">
### OpenFeign 설정
이 Webhook 엔드포인트로 API 호출을 해야 한다. 나는 `OpenFeign`을 기존에 프로젝트에서 쓰고 있었으므로, 디스코드 메세지를 보낼 때도 `FeignClient`를 쓰기로 했다. 먼저 `application.yml`에 다음과 같이 설정한다.
```yaml
discord:
  webhook:
    url: [Webhook 엔드포인트]
```
그 다음에 Discord가 지원하는 JSON 구조대로 DTO를 만들었다. 여기서 `content`는 메세지 제목, `embeds`의 `title`은 메세지 본문의 제목, 그리고 `description`은 마크다운 형태로 전달되는 본문 내용이다.
```json
{
  "content": "string"
  "embeds": [
  	{
  		"title": "string"
  		"description": "string"
	},
	...
  ]
}
```
이에 맞춰서 DTO를 다음과 같이 만들어줬다. `DiscordMessageDto`를 만들고, 그 안에 `List`로 `DiscordEmbedDto`가 포함되도록 했다.
```java
@Builder
@AllArgsConstructor
@NoArgsConstructor
@Getter
public class DiscordMessageDto {
    @JsonProperty("content")
    private String content;

    @JsonProperty("embeds")
    private List<DiscordEmbedDto> embeds;
}
```
```java
@Builder
@AllArgsConstructor
@NoArgsConstructor
@Getter
public class DiscordEmbedDto {
    @JsonProperty("title")
    private String title;

    @JsonProperty("description")
    private String description;
}
```
이제 전송할 데이터 구조인 DTO가 명시되었고, 엔드포인트도 설정에 명시해줬으니 `FeignClient`를 쓸 수 있다. 보낼 DTO는 `DiscordMessageDto`이고, 메세지를 보내기만 하면 되므로 받을 DTO는 없으므로 `void`로 해줬다.
```java
@FeignClient(
        name = "discordClientApi",
        url = "${discord.webhook.url}")
public interface DiscordClientApi {
    @PostMapping()
    void sendAlarm(@RequestBody DiscordMessageDto discordMessageDto);
}
```
### 기존의 Exception 처리 구조
이제 FeignClient도 설정되어 있으므로 진짜로 메세지를 보내면 된다. 그 전에 에러 발생 시 어떤 타이밍에 디스코드 메세지를 보내는지 설명하기 위해, 기존에 에러 처리 구조를 설명하려고 한다. 나는 커스텀 에러 처리를 위해서 `RuntimeException`을 상속받는 `BaseException`을 만들어 줬었다. 이 에러 안에는 에러 코드와 에러 메세지가 들어가 있다.
```java
@Getter
@RequiredArgsConstructor
public class BaseException extends RuntimeException {
    private final ErrorCode errorCode;
    private final String message;
}
```
그리고 좀 더 세부적으로 에러를 분류하기 위해서, 이 `BaseException`을 상속받는 다양한 에러 클래스들이 있다. 아래는 기본 카테고리인 "미분류" 카테고리를 삭제하려고 하면 나는 에러인 `DefaultCategoryException`이다.
```java
@Getter
public class DefaultCategoryException extends BaseException {
    public DefaultCategoryException() {
        super(ErrorCode.INVALID_CATEGORY_DELETE, ErrorCode.INVALID_CATEGORY_DELETE.getMessage());
    }
    public DefaultCategoryException(String message) {
        super(ErrorCode.INVALID_CATEGORY_DELETE, message);
    }
    public DefaultCategoryException(ErrorCode errorCode) {
        super(errorCode, errorCode.getMessage());
    }
}
```
`DefaultCategoryException`은 아래와 같은 에러 코드 및 설명을 기본적으로 가지고 있다.
```java
@Getter
@AllArgsConstructor
public enum ErrorCode {

    /* Common */
    _INTERNAL_SERVER_ERROR(INTERNAL_SERVER_ERROR, "C000", "서버 에러, 관리자에게 문의 바랍니다."),

    /* 카테고리 관련 */
    INVALID_CATEGORY_NAME(CONFLICT, "CATE001", "해당 카테고리명이 이미 존재합니다. 카테고리명은 중복될 수 없습니다.");
    
    private final HttpStatus httpStatus;
    private final String code;
    private final String message;
}
```
이런 에러들은 service단에서 `throw` 될 경우, 아래와 같이 `GlobalExceptionHandler`에서 `catch`된다. 그러면 `@ExceptionHandler(BaseException.class)`에 의해서 `BaseException`을 상속받은 모든 종류의 에러가 `catch`되고, 그에 맞는 답변을 클라이언트에게 줄 수 있다. 굳이 `BaseException`을 만들고, `DefaultCategoryException`이 이를 상속받게 만든 이유도 바로 여기에 있는데, `BaseException`를 상속받는 비즈니스 로직 에러들은 한꺼번에 `ExceptionHandler`로 처리하기 위해서이다.
```java
@ControllerAdvice
@RequiredArgsConstructor
public class GlobalExceptionHandler {
    private final DiscordAlertSender discordAlertSender;
    // 비즈니스 로직 에러 처리
    @ExceptionHandler(BaseException.class)
    protected ResponseEntity<ErrorResponse> handleBusinessException(final BaseException baseException, HttpServletRequest httpServletRequest) {
        final ContentCachingRequestWrapper contentCachingRequestWrapper = new ContentCachingRequestWrapper(httpServletRequest);
        return new ResponseEntity<>(ErrorResponse.onFailure(baseException.getErrorCode(), baseException.getMessage()),null, baseException.getErrorCode().getHttpStatus());
    }

    // 따로 처리하지 않은 500 에러 모두 처리
    @ExceptionHandler(Exception.class)
    protected ResponseEntity<ErrorResponse> handleException(Exception exception, HttpServletRequest httpServletRequest) {
        final ContentCachingRequestWrapper contentCachingRequestWrapper = new ContentCachingRequestWrapper(httpServletRequest);
        return new ResponseEntity<>(ErrorResponse.onFailure(ErrorCode._INTERNAL_SERVER_ERROR), null, INTERNAL_SERVER_ERROR);
    }
}
```
그런데 이렇게 내가 직접 처리해준 비즈니스 로직 에러 말고도 온갖 종류의 500 에러가 있는데, 이를 모두 하나하나 커스텀 예외 처리를 만드는 건 효율적이지 않다. 그래서 `@ExceptionHandler(Exception.class)`를 따로 만들어서 따로 처리하지 않은 500 에러를 핸들링하게 해주었다.

그런데 또 이 방식의 문제는 클라이언트가 `"서버 에러, 관리자에게 문의 바랍니다."`라는 일반적인 답변만 받을 수 있다는 것이다. 그렇다고 콘솔 에러를 클라이언트에게 바로 보내주기에는 문제가 많다. 예를 들어 SQL를 처리하다가 생긴 에러 로그를 클라이언트에게 보내주면, DB 내부 구조에 대해 클라이언트가 너무 많이 알게되는 문제가 생긴다. 나랑 같이 협업하는 개발자만 그 에러 로그를 볼 수 있다면 문제가 없지만, 제 3자가 보게 되면 문제가 생길 수 있다. 그래서 바로 이 `@ExceptionHandler(Exception.class)`가 `catch`하는 에러에 대해서는 우리 팀이 쓰는 디스코드 채널에 메세지로 콘솔 에러를 보내서 확인하게 만드는 게 목표이다.
### Discord에 메세지 보내기
사설이 길었는데, 이제 진짜로 디스코드에 메세지를 보내려고 한다. 전반적으로 아래 글에서 알려주는 코드를 많이 참고했다.

- [Spring + Discord Webhook으로 에러 상황 알림 받기](https://velog.io/@eomgerm/AvAb-Spring-Discord-Webhook%EC%9C%BC%EB%A1%9C-%EC%97%90%EB%9F%AC-%EC%83%81%ED%99%A9-%EC%95%8C%EB%A6%BC-%EB%B0%9B%EA%B8%B0)

먼저 `DiscordMessageGenerator`는 실제로 디스코드 메세지를 보내기 위한 DTO를 생성해주는 클래스이다. 이 클래스는 콘솔 에러를 가져오고, 현재 활성화된 프로파일 정보를 가져오고(내 서비스는 개발 및 프로덕션 서버가 따로 있어서, 어느 서버에서 발생한 일인지 알기 위해서는 프로파일 정보가 필요하다), 요청 엔드포인트 및 에러 발생 시간을 수집해서 `DiscordMessageDto`를 생성한다.
```java
@Component
@RequiredArgsConstructor
public class DiscordMessageGenerator {

    @Value("${spring.profiles.active}")
    private String activeProfile;

    /* 메세지 생성 */
    public DiscordMessageDto createMessage(Exception exception, HttpServletRequest httpServletRequest) {
        return DiscordMessageDto.builder()
            .content("## 🚨 서버 에러 발생 🚨")
            .embeds(List.of(DiscordEmbedDto.builder()
                            .title("ℹ️ 에러 정보")
                            .description("### 🕖 에러 발생 시간\n"
                                    + ZonedDateTime.now(ZoneId.of("Asia/Seoul")).format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH시 mm분 ss초(서울 시간)"))
                                    + "\n"
                                    + "### 🔗 요청 엔드포인트\n"
                                    + httpServletRequest.getRequestURI()
                                    + "\n"
                                    + "### 🖥️ 에러 발생 서버\n"
                                    + activeProfile
                                    + "\n"
                                    + "### 📜 에러 로그\n"
                                    + "```\n"
                                    + getStackTrace(exception).substring(0, 1000)
                                    + "\n```")
                            .build()
                            )
            ).build();
    }

    /* 콘솔의 에러 보여주기 */
    private String getStackTrace(Exception exception) {
        StringWriter stringWriter = new StringWriter();
        exception.printStackTrace(new PrintWriter(stringWriter));
        return stringWriter.toString();
    }
}
```
그리고 `DiscordAlertSender`에서는 `DiscordMessageGenerator`를 통해 생성된 DTO를 `DiscordClientApi`에게 전달해 실제로 API 호출을 하는 역할을 한다.
```java
@Component
@RequiredArgsConstructor
public class DiscordAlertSender {
    private final DiscordClientApi discordClientApi;
    private final DiscordMessageGenerator discordMessageGenerator;

    public void sendDiscordAlarm(Exception exception, HttpServletRequest httpServletRequest) {
        discordClientApi.sendAlarm(discordMessageGenerator.createMessage(exception, httpServletRequest));
    }
}
```
마지막으로, 이 `DiscordAlertSender`가 호출되는 시점은 아래처럼 `@ExceptionHandler(Exception.class)`가 활성화된 시점으로 했다.
```java
@ControllerAdvice
@RequiredArgsConstructor
public class GlobalExceptionHandler {
    private final DiscordAlertSender discordAlertSender;
    // 비즈니스 로직 에러 처리
    @ExceptionHandler(BaseException.class)
    protected ResponseEntity<ErrorResponse> handleBusinessException(final BaseException baseException, HttpServletRequest httpServletRequest) {
        final ContentCachingRequestWrapper contentCachingRequestWrapper = new ContentCachingRequestWrapper(httpServletRequest);
        return new ResponseEntity<>(ErrorResponse.onFailure(baseException.getErrorCode(), baseException.getMessage()),null, baseException.getErrorCode().getHttpStatus());
    }

    // 따로 처리하지 않은 500 에러 모두 처리
    @ExceptionHandler(Exception.class)
    protected ResponseEntity<ErrorResponse> handleException(Exception exception, HttpServletRequest httpServletRequest) {
        discordAlertSender.sendDiscordAlarm(exception, httpServletRequest);
        final ContentCachingRequestWrapper contentCachingRequestWrapper = new ContentCachingRequestWrapper(httpServletRequest);
        return new ResponseEntity<>(ErrorResponse.onFailure(ErrorCode._INTERNAL_SERVER_ERROR), null, INTERNAL_SERVER_ERROR);
    }
}
```
이렇게 설정하고, 일부러 존재하지 않는 엔드포인트에 대해서 호출해 보았더니 디스코드 메세지가 아주 잘 오는 것을 확인할 수 있었다. 딱 Exception 부분의 에러 로그만 볼 수 있어서 매우 편리하다.
<img style="width: 100%;" src="https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/663221a8-6bac-4adf-ab77-301bc392db53">

### 🔥서버 공격🔥
이렇게 딱 디스코드 에러 메세지 웹훅을 배포한 날에 갑자기 5분 동안 몇 개의 에러 메세지가 연달아 발생하는 일이 생겼다. 근데 로그를 보니까 듣도 보도 못한 엔드포인트로 자꾸 요청이 오고 있었다. 일단 프론트가 아닌데 js는 왜 가져가려는 건지 모르겠고, `.env` 처럼 중요한 파일도 노리는 것으로 보였다.
```bash
/
/.env
/resolve
/.env.prod
/favicon.ico
/.git/config
/remote/logincheck
/Core/Skin/Login.aspx
/api/v2/static/not.found
/cf_scripts/scripts/ajax/ckeditor/ckeditor.js
/phpunit/src/Util/PHP/eval-stdin.php
/phpunit/phpunit/src/Util/PHP/eval-stdin.php
/app/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php
```
<img style="width: 100%;" src="https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/c2762cea-71fc-4bc8-ae04-5f2b3671125e">
EC2 모니터링도 확인해보니 네트워크 입력이 확실히 많이 들어온 때가 있었다.
<img style="width: 100%;" src="https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/8fce15a8-4447-4407-badc-51e5b7c0a2dd">
너무 이상해서 구글링해서 나온 아래 글을 읽어보니까 ASP.NET 프레임워크를 쓰는 어플리케이션에서 중요한 파일들이라고 한다. 내 서버는 Springboot를 쓰는데도 이러한 요청이 들어오는 이유는, 기술 스택과 관련없이 하나만 걸려라라는 마인드로 공격을 진행하기 때문이라고 한다. 아니...해커치고 너무 대충 공격하는 거 아니야? 아무래도 깃허브에서 이런 엔드포인트를 크롤링해서 공격을 보내는 자동화 로직이 있는 것 같다.

- [NGINX 로그가 드러낸 미확인 공격 시도 - TLS 적용](https://babgeuleus.tistory.com/entry/NGINX-%EB%A1%9C%EA%B7%B8%EA%B0%80-%EB%93%9C%EB%9F%AC%EB%82%B8-%EB%AF%B8%ED%99%95%EC%9D%B8-%EA%B3%B5%EA%B2%A9-%EC%8B%9C%EB%8F%84-SSL-%EC%A0%81%EC%9A%A9)

애썼지만 내가 디스코드 서버 에러 알림 구축하는 바람에 다 들켰죠? 하지만 괜히 쫄려서 CICD하면서 EC2 안에 원래 `application.yml` 등 중요한 파일이 업로드 되던 로직도 변경해주고, 그리고 디스코드 알림에서 클라이언트 IP도 같이 수집되도록 바꿔줬다. 그리고 EC2 보안 그룹도 변경해줬는데, 원래는 https 443과 http 80에 대해서 `0.0.0.0/0 (모든 IP)`가 허용되어 있었다(나도 이렇게 설정해줬던 내 자신이 믿기지X). 이걸 VPC의 CIDR에 대해서만 허용해서 ALB에게서만 오는 트래픽만 허용했다. 도메인 이름을 타고 오는 것은 모든 IP에 대해서 열려있지만, EC2의 퍼블릭 IP에 대해서 직접 접근하는 트래픽에 대한 보안은 조금 높였다.
<img style="width: 100%;" src="https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/4490ac6a-f729-45b9-9df5-d1b445288dc1">
### 클라이언트 IP 수집하기
클라이언트 IP 수집하기 위해서는 다음 글을 참고했다.
<img style="width: 100%;" src="https://velog.io/@haron/Spring-%EC%B5%9C%EC%B4%88-%EC%9A%94%EC%B2%AD-IP-%EB%8A%94-%EC%96%B4%EB%96%BB%EA%B2%8C-%EA%B0%80%EC%A0%B8%EC%98%A4%EB%8A%94-%EA%B1%B8%EA%B9%8C-6aie0x11">
`DiscordMessageGenerator`를 조금 수정해서 Client IP를 에러 메세지에 포함시키는 것으로 바꿨다.
```java
@Component
@RequiredArgsConstructor
public class DiscordMessageGenerator {

    @Value("${spring.profiles.active}")
    private String activeProfile;

    /* 메세지 생성 */
    public DiscordMessageDto createMessage(Exception exception, HttpServletRequest httpServletRequest) {
        return DiscordMessageDto.builder()
            .content("## 🚨 서버 에러 발생 🚨")
            .embeds(List.of(DiscordEmbedDto.builder()
                            .title("ℹ️ 에러 정보")
                            .description("### 🕖 에러 발생 시간\n"
                                    + ZonedDateTime.now(ZoneId.of("Asia/Seoul")).format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH시 mm분 ss초(서울 시간)"))
                                    + "\n"
                                    + "### 🔗 요청 엔드포인트\n"
                                    + httpServletRequest.getRequestURI()
                                    + "\n"
                                    + "### 🧐 요청 클라이언트 IP\n"
                                    + getRemoteIp(httpServletRequest)
                                    + "\n"
                                    + "### 🖥️ 에러 발생 서버\n"
                                    + activeProfile
                                    + "\n"
                                    + "### 📜 에러 로그\n"
                                    + "```\n"
                                    + getStackTrace(exception).substring(0, 1000)
                                    + "\n```")
                            .build()
                            )
            ).build();
    }

    /* 콘솔의 에러 보여주기 */
    private String getStackTrace(Exception exception) {
        StringWriter stringWriter = new StringWriter();
        exception.printStackTrace(new PrintWriter(stringWriter));
        return stringWriter.toString();
    }

    /* 클라이언트 요청 IP 알아내기 */
    private String getRemoteIp(HttpServletRequest httpServletRequest){
        return httpServletRequest.getRemoteAddr();
    }
}
```
시험 삼아서 내 컴퓨터로 존재하지 않는 엔드포인트에 대해 요청을 보내봤는데, 다음과 같이 내 로컬 컴퓨터의 IP를 잘 수집하는 것을 볼 수 있었다(물론 해커가 VPN을 쓰면 클라이언트 IP 수집이 큰 의미는 없겠지만).
<img style="width: 100%;" src="https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/f2091b61-b4ca-4796-8712-bae72fbd5a60">
은근 이런 공격이 많이 들어오는 것 같다. 저번에 GCP 사용할 때 내 서버에서 누가 암호화폐 채굴(?!)을 하는 바람에 프로젝트가 아예 종료되는 일도 있었고...언젠가 해커와 맞다이를 뜰 때가 올지도 모르니 보안을 좀 더 공부해야겠다는 생각이 들었다.
### + 추가)
나는 내가 일시적으로만 공격 받는 줄 알았는데, 그게 아니라 그냥 공개되어 있는 API 서버에 대해서는 이런 공격이 항상 들어오고 있었나보다. 지금 이 글을 쓰고 있는 순간에도 공격이 30번쯤 들어오고 있다. 새벽에도 낮에도 저녁에도 디스코드 메세지가 온다. 클라이언트 IP를 찍어보니 왠 홍콩에서 내 서비스를 접속하고 있다고 한다. 도대체 누구신지...
<img style="width: 100%;" src="https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/08e7e573-3d73-471b-b7b0-52f9b8b06410">
### 프로젝트 깃허브
실제 디스코드 메세지 관련 코드를 확인하고 싶은 사람들은 아래 링크를 참고하면 된다.

- [Discord Webhook Github Repository](https://github.com/studio-recoding/NESS_BE/tree/main/src/main/java/Ness/Backend/infra/discord)

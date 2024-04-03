## TIL - 스프링 서블릿(Servlet) 대해
---
Spring Boot로 코드를 짜다가 어느 날 든 생각. 도대체 `HttpServletRequest request`와 `HttpServletResponse response`가 무엇일까?  그동안 막연하게 `request`와 `response`일 것이라고 생각하고 있었지, 정작 이 중간에 있는 단어, `Servlet` 이 무엇인지에 대해서는 별로 관심이 없었다.
조사해 보니, Spring MVC는 중앙 `Servlet`인 `DispatcherServlet`를 중심으로 디자인되었다고 한다. `DispatcherServlet`는 사용자의 요청을 처리하기 위한 shared algorithm를 제공한다. 

`DispatcherServlet`은 Spring 기반 웹 애플리케이션의 **Front Controller** 역할을 한다. **Front Controller**는 내 웹사이트에 들어오는 모든 종류의 request를 처리하고 수락하는 역할을 한다. 즉, 일단 요청이 들어오면 이를 수락하고, 해당 요청을 처리할 올바른 컨트롤러로 보내주는 역할을 한다.

예를 들어서, `student.com` 이라는 웹사이트가 있고 클라이언트가 다음 `student.com/save`을 통해서 학생 데이터 저장을 요청한다고 가정하자. 그러면 먼저 프론트 컨트롤러인 가 `student.com/*` 경로로 들어오는 모든 요청을 처리한다. 이 요청이 프론트 컨트롤러에 대해서 수락되면 이 컨트롤러가 `/save` 작업에 대한 요청을 처리하고,  `Controller_1`에 할당한다. 그런 다음 클라이언트에 응답을 다시 반환해준다.

![DispatcherServlet](https://github.com/JeonHaeseung/SpringBootTILStudy/assets/89632139/ed1ff7f8-4a4a-481f-9ac3-e7fdfeb59ade)

Spring MVC에서 프론트 컨트롤러는 다음과 같은 방식으로 작동한다. Spring MVC에서 서버 사이드 랜더링까지 하는 상황이라고 했을 때, `controller`로(`@Controller` 어노테이션이 붙은 클래스) 사용자의 request를 보내고, 그에 대한 데이터를 `model`로 받은 다음, `view` 템플렛(여기서는 `.jsp`)을 찾아서 `model` 데이터를 전달하고, 그 결과로 파싱된 데이터를 받아서 최종적으로 response하는 게 프론트 컨트롤러의 역할이었던 것이다. 
![ServletEngine](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/3ab8f973-e9bb-417b-907c-126d575cb101)

그렇다면 나는 `DispatcherServlet`를 만든 적이 없는데, 어떻게 프론트 컨트롤러가 만들어진 것일까?
이는 스프링 내부에 미리 구현되어 있기 때문이다. 아래 코드는 애플리케이션이 시작되는 순간 호출되는 코드이다. `onStartup` 함수는 애플리케이션이 시작되면 호출되는데,
1) `AnnotationConfigWebApplicationContext`에 애플리케이션의 설정을 세팅해주고,
2)  `DispatcherServlet `를 생성해서 앞서 만든 설정 `context`를 넣어준다. 
3) 그 후에는 `ServletContext`에 `app`이라는 이름의 서블릿을 등록해준다. 
4) 마지막으로 `setLoadOnStartup(1)`를 통해서 가장 먼저(1번 순서)로 로드되게 함으로써 전체 애플리케이션을 시작하는 것이다.

``` java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

	@Override
	public void onStartup(ServletContext servletContext) {

		// Load Spring web application configuration
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
		context.register(AppConfig.class);

		// Create and register the DispatcherServlet
		DispatcherServlet servlet = new DispatcherServlet(context);
		ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
		registration.setLoadOnStartup(1);
		registration.addMapping("/app/*");
	}
}
```

그럼 `DispatcherServlet` 말고 그냥 서블릿은 무엇일까? `@Controller` 어노테이션이 붙은 클래스는 아직 서블릿이 아니고, 그냥 단순한 Bean이다. 서블릿은 HTTP 요청과 응답을 전송하는 데 사용되는 것으로, 웹 서비스의 한 단계 아래 계층이라고 보면 된다. Spring을 사용할 때 서블릿 작업은 Spring에 의해 수행되며, `DispatcherServlet` 요청을 올바른 Bean으로 전달한다. 표준 스펙에서는 다음과 같이 소개하고 있다.
> A servlet is a Java™ technology-based Web component, managed by a container, that generates dynamic content. Like other Java technology-based components, servlets are platform-independent Java classes that are compiled to platform-neutral byte code that can be loaded dynamically into and run by a Java technology-enabled Web server. Containers, sometimes called servlet engines, are Web server extensions that provide servlet functionality. Servlets interact with Web clients via a request/response paradigm implemented by the servlet container.

> 서블릿은 컨테이너에 의해 관리되는 Java™ 기술 기반의 웹 컴포넌트로, 동적 콘텐츠를 생성합니다. 다른 Java 기술 기반의 컴포넌트와 마찬가지로, 서블릿은 플랫폼에 독립적인 Java 클래스로 컴파일되어 플랫폼 중립적인 바이트 코드로 변환됩니다. 이는 Java 기술을 지원하는 웹 서버에 동적으로 로드되어 실행될 수 있습니다. **컨테이너는 때때로 서블릿 엔진이라고도 불리며,** 서블릿 기능을 제공하는 웹 서버 확장 기능입니다. 서블릿은 서블릿 컨테이너에 의해 구현된 요청/응답 패러다임을 통해 웹 클라이언트와 상호 작용합니다.

그러니까 위에서 보았던 서블릿 엔진에 서블릿 컨테이너였다. 서블릿에 대해서 좀 더 자세히 알기 위해서 일반적인 이벤트에 대한 Step-by-step 설명을 보자.

1. 클라이언트(예: 웹 브라우저)가 웹 서버에 액세스하고 HTTP 요청을 보낸다.
2. 요청은 웹 서버에 의해 수신되고 서블릿 컨테이너에 전달된다. 서블릿 컨테이너는 호스트 웹 서버와 동일한 프로세스에서 실행될 수도 있고, 동일한 호스트의 다른 프로세스에서 실행될 수도 있으며, 요청을 처리하는 웹 서버와 다른 호스트에서 실행될 수도 있다.
4. 서블릿 컨테이너는 서블릿의 구성에 기반하여 어떤 서블릿을 호출할지 결정하고, 요청과 응답을 나타내는 객체를 사용하여 해당 서블릿을 호출한다.
5. 서블릿은 요청 객체(request object)를 사용하여 원격 사용자가 누구인지, 이 요청의 일부로 HTTP POST 매개변수가 전송되었는지 등의 관련 데이터를 확인한다. 서블릿은 프로그래밍된 논리를 수행하고, 클라이언트에게 다시 전송할 데이터를 생성한다. 이 데이터를 응답 객체(response object)를 통해 클라이언트에게 전송한다.
6. 서블릿이 요청을 처리하는 것을 마치면, 서블릿 컨테이너는 응답이 적절하게 플러시(flush)되도록 보장하고, 제어를 호스트 웹 서버에 반환한다.

## QnA
---
이렇게 스터디를 준비해서 발표했는데, 이런 질문을 받았다.
> Step-by-step의 4번을 보면 일반적으로 API를 통해서 controller를 호출하는 과정과 비슷해 보이는데, 그럼 우리가 개발한 controller가 하나하나 서블릿이라고 하는 건가요?

이 질문을 듣고 뭐라고 답하기가 굉장히 애매해서, 아직 서블릿이 뭔지 모른다는 생각이 들었다. 그래서 추가적으로 검색해보다가 아래 이미지를 찾았다.
![image](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/2f8bcabf-fd61-4267-8f34-26001ac9ab19)
그러니까 서블릿이라는 것은 `DispatcherServlet`에 가기도 전에, `doGet(), doPost(), doPut(), doDelete()`와 같은 메소드들을 먼저 거치게 하는 과정이다. geeksforgeeks에서는 서블릿이 `HttpServletRequest를` 처리하는 과정을 다음과 같이 설명하고 있다.
1. `HttpServletRequest` 인터페이스는 ServletRequest 인터페이스를 확장하여 서블릿에 대한 Http 관련 요청 정보를 제공한다.
2. 서블릿 컨테이너는 `HttpServletRequest` 객체를 생성하고 이를 서블릿의 서비스 메소드(`doPost(),` `doGet()` 등)에 인수로 전달한다.
3. 이 객체는 매개변수 이름과 값, 속성, 입력 스트림과 같은 데이터를 제공한다.
4. 예를 들어 `doPost()`를 하기 위해서 `getParameter()` 및 `getParameterValues()`를 사용하여 양식 필드의 값을 읽을 수 있는데, 이 메소드는 지정된 이름으로 지정된 요청의 필드/매개변수 값을 문자열로 반환한다.

그러니까 예전에 JSP를 통해서 개발할 때는 `doPost(),` `doGet()` 등의 함수를 써야했는데, 이제는 그게 개발자가 직접 다룰 필요가 없도록 백본으로만 남겨두고, 그 위에서 SpringBoot가 돌아가는 형식이다. `@Controller`, `@RequestMapping`과 같은 Spring MVC의 주요 어노테이션들은 내부적으로 서블릿을 사용하는 것이었다. 일종의 내부 API 또는 요청의 전처리를 담당하는 느낌으로 이해하면 될 것 같다. 그래서 Spring Security에서 `@Controller`까지 도달하기 전에도 `HttpServletRequest`를 사용해 보안 관련 로직을 처리할 수 있는 것이다!

## 레퍼런스
---
- [docs.spring.io의 mvc-servlet](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html)
- [geeksforgeeks의 what-is-dispatcher-servlet-in-spring](https://www.geeksforgeeks.org/what-is-dispatcher-servlet-in-spring/)
- [stackoverflow의 when-to-use-servlet-or-controller](https://stackoverflow.com/questions/16439249/when-to-use-servlet-or-controller)
- [stackoverflow의 difference-between-servlet-and-web-service](https://stackoverflow.com/questions/5930795/difference-between-servlet-and-web-service)
- [Java Servlet Specification](https://javaee.github.io/servlet-spec/DOWNLOADS.html)
- [LinkedIn의 What is Web Context and Request Lifecycle in a Spring Web Application](https://www.linkedin.com/pulse/what-web-context-request-lifecycle-spring-application-ali-pty1f/)
- [geeksforgeeks의 servlet-form-data](https://www.geeksforgeeks.org/servlet-form-data/)

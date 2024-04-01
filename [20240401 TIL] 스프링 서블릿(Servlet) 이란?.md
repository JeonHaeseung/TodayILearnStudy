## TIL - 스프링 서블릿(Servlet) 대해
---
Spring Boot로 코드를 짜다가 어느 날 든 생각. 도대체 `HttpServletRequest request`와 `HttpServletResponse response`가 무엇일까?  그동안 막연하게 `request`와 `response`일 것이라고 생각하고 있었지, 정작 이 중간에 있는 단어, `Servlet` 이 무엇인지에 대해서는 별로 관심이 없었다.
조사해 보니, Spring MVC는 중앙 `Servlet`인 `DispatcherServlet`를 중심으로 디자인되었다고 한다. `DispatcherServlet`는 사용자의 요청을 처리하기 위한 shared algorithm를 제공한다. 

`DispatcherServlet`은 Spring 기반 웹 애플리케이션의 **Front Controller** 역할을 한다. **Front Controller**는 내 웹사이트에 들어오는 모든 종류의 request를 처리하고 수락하는 역할을 한다. 즉, 일단 요청이 들어오면 이를 수락하고, 해당 요청을 처리할 올바른 컨트롤러로 보내주는 역할을 한다.

예를 들어서, `student.com` 이라는 웹사이트가 있고 클라이언트가 다음 `student.com/save`을 통해서 학생 데이터 저장을 요청한다고 가정하자. 그러면 먼저 프론트 컨트롤러인 가 `student.com/*` 경로로 들어오는 모든 요청을 처리한다. 이 요청이 프론트 컨트롤러에 대해서 수락되면 이 컨트롤러가 `/save` 작업에 대한 요청을 처리하고,  `Controller_1`에 할당한다. 그런 다음 클라이언트에 응답을 다시 반환해준다.

![DispatcherServlet](https://github.com/JeonHaeseung/SpringBootTILStudy/assets/89632139/ed1ff7f8-4a4a-481f-9ac3-e7fdfeb59ade)

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

## 레퍼런스
---
- [docs.spring.io의 mvc-servlet](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html)
- [geeksforgeeks의 what-is-dispatcher-servlet-in-spring](https://www.geeksforgeeks.org/what-is-dispatcher-servlet-in-spring/)

## QnA
---
(스터디 이후에 추가 예정)

## TIL - Spring Bean Life Cycle에 대해서
---
Spring에서 빈이 만들어지고, DI로 사용된다는 것은 알고 있었다. 그런데 전체적으로 Spring Bean Life Cycle에 대해서는 잘 모르는 것 같아서 이번에 새롭게 정리해보려고 한다. 간단하게 정리하자면, bean은 다음과 같은 라이프 사이클을 거친다.
![image](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/f5da182c-442c-4c5d-a1e3-585f3a63c69b)
`스프링 컨테이너 시작 -> 스프링 빈 초기화 -> 의존관계 주입(DI) -> init() -> 유틸리티 -> destory() -> 스프링 종료`라는 라이프 사이클을 거치는데, 더 디테일한 전체 flow는 다음과 같다.
![image](https://github.com/JeonHaeseung/TodayILearnStudy/assets/89632139/fbecfc42-2b76-40a9-bea8-70591c8f91c4)
1. 패키지 스캐닝이 있는 어노테이션(`@Bean`, `@Controller`, `@RestController`, `@Service`, `@Repository`, `@Component`)이나 XML 등으로 Bean Definition이 선언된다.
2. `BeanDefinitionReader`가 이 구성을 읽고 `BeanDefinition` 객체를 생성한다.
3. `BeanFactory`가 bean의 생성자를 호출해서 `FactoryBean` 객체를 생성한다.
4. DI를 통해서 의존성이 주입된다. 만약 특정 빈의 생성자에서 다른 빈에 대한 의존성이 있다면(예: `@Service`에서 `@Repository`를 필요로 하는 경우) 해당 빈이 먼저 생성된다.
5. `BeanPostProcessor`가 init() 메소드 전에 호출된다. 해당 프로세서는 초기화 전에 빈에 대한 조작 또는 수정이 가능하다.
6. `@PostConstruct`에 의해서 init() 메소드가 호출되고, 빈 초기화가 진행된다.
7. `BeanPostProcessor`가 다시 호출된다(프록시 등을 빈 주위에 만들고 싶을 경우에)
8. Application Context에 빈이 존재하고, 사용 또는 접근이 가능하다. 빈 인스턴스는 디폴트로 `singleton`으로 존재한다.
9. `@PreDestroy`에 의해서 destory() 메소드가 호출되고 빈이 종료된다.

여기서 `@PostConstruct`와 `@PreDestroy`를 알기 위해서, 간단한 bean 클래스에서 해당 어노테이션이 구성된 코드를 가져와보았다. 

``` java
package com.journaldev.spring.service;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

public class MyService {

	@PostConstruct
	public void init(){
		System.out.println("MyService init method called");
	}
	
	public MyService(){
		System.out.println("MyService no-args constructor called");
	}
	
	@PreDestroy
	public void destory(){
		System.out.println("MyService destroy method called");
	}
}
```

[저번 TIL](https://github.com/JeonHaeseung/TodayILearnStudy/blob/main/%5B20240401%20TIL%5D%20%EC%8A%A4%ED%94%84%EB%A7%81%20%EC%84%9C%EB%B8%94%EB%A6%BF(Servlet)%20%EC%9D%B4%EB%9E%80%3F.md)에서 `DispatcherServlet`을 공부할 때 `AnnotationConfigWebApplicationContext`라는 게 생성되는 것을 본 적이 있다.
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

이때 `AppConfig`라는 게 Context에 등록이 되었는데, 아래에서 볼 수 있듯이 이 코드가 Bean Definition을 생성하는 역할을 한다. 아주 예전에는 `@Controller`등 자동 컴포넌트 스캔을 하지 않고, 이 클래스 안에 빈으로 등록해야 하는 클래스를 전부 적어준 모양이다. 이 `AppConfig`을 `AnnotationConfigWebApplicationContext`에 등록해주고, 이 context가 다시 `DispatcherServlet`에 등록되기 때문에 `DispatcherServlet`은 적절한 controllor에 request를 처리할 수 있는 것이다. 
``` java
 @Configuration
 public class AppConfig {

     @Bean
     public MyBean myBean() {
         // instantiate, configure and return bean ...
     }
 }
```

## QnA
---
(스터디 이후 채워질 예정)

## 레퍼런스
---
- [geeksforgeeks의 bean-life-cycle-in-java-spring](https://www.geeksforgeeks.org/bean-life-cycle-in-java-spring/)
- [medium의 spring-bean-lifecycle-full-guide](https://medium.com/@TheTechDude/spring-bean-lifecycle-full-guide-f865966e89ce)
- [digitalocean의 spring-bean-life-cycle](https://www.digitalocean.com/community/tutorials/spring-bean-life-cycle)
- [docs.spring.io의 Annotation Interface Configuration](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Configuration.html)
- [tistory의 AppConfig](https://dear-by-dear.tistory.com/46)
- [zetcode의 Spring @Configuration tutorial](https://zetcode.com/spring/configuration/)

## TIL - @Scheduled 어노테이션에 대해
---
이메일 기능을 구현하면서 매일 일정한 시간에 메일을 보내줘야 하는 일이 생겼다. 찾아보니까 스프링 부트에 있는 `@Scheduled` 어노테이션을 사용하면 아주 간단하게 구현할 수 있다고 해서 해당 어노테이션을 사용해 구현해보았는데, 아주 간단하고 편리해서 이 어노테이션에 대해서 좀 더 알아보았다.

### Step1 - build.gradle 종속성
우선 이 기능은 Springboot Starter에 기본적으로 내장되어 있는 기능이다. 다른 라이브러리를 추가할 필요 없이 처음 프로젝트를 시작했을 때 필수적으로 포함되어 있을 Springboot Starter가 `build.gradle`에 잘 포함되어 있는지만 보면 된다. 
```java
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter'
	testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

### Step2 - @EnableScheduling 활성화
스케쥴링을 활성화하기 위해서는 `@EnableScheduling`를 1. `@SpringBootApplication`의 하위 패키지이자, 2. `@Component`가 있는 클래스(즉 `@Service` 있는 클래스도 가능)에 붙여주어야 한다. 아래와 같이 `@SpringBootApplication`가 있는 클래스에 같이 붙여주어도 된다.
```java
@SpringBootApplication
@EnableScheduling
public class BackendApplication {

	public static void main(String[] args) {
		SpringApplication.run(BackendApplication.class, args);
	}

}
```

나는 아래와 같이 실제로 스케쥴링 작업을 사용할 `EmailService` 클래스에 붙여주었다. (`@Service`가 `@Component`를 포함하고 있으므로, 해당 클래스에 붙여도 된다.)
```java
@Service
@EnableScheduling
public class EmailService {
    ...
}
```

### Step3 - @Scheduled로 스케줄링하기
이제 `@EnableScheduling`이 선언된 클래스 또는 하위 패키지의 클래스 안에 원하는 스케쥴링 메소드를 구현하고, 그 메소드 위에 `@Scheduled` 어노테이션을 붙인 후 원하는 옵션을 설정해주면 된다.
```java
@Service
@EnableScheduling
public class EmailService {
    private final MemberRepository memberRepository;
    private final AsyncEmailService asyncEmailService;

    // 매일 오전 자정에 스케쥴링
    @Scheduled(cron = "0 0 12 * * *")
    public void scheduleEmailCron(){
        log.info("스케쥴링 활성화");
        List<Member> activeMembers = memberRepository.findMembersByProfileIsEmailActive(true);

        for (Member member : activeMembers) {
            asyncEmailService.sendEmailNotice(member.getId(), member.getEmail());
        }
    }
}
```

이때 `@Scheduled`의 옵션으로는 다양한 것이 있다. 먼저 `메소드 호출~다음 메소드 호출 사이의 간격`을 정해주는 `fixedRate`이 있다. 단위는 ms이므로, 1,000으로 숫자를 지정해야 1초가 된다. 이때 이 옵션은 `@Scheduled`가 붙은 메소드의 실행 시간을 고려하지 않는다. 예를 들어서 해당 메소드가 10초는 걸리는 함수라고 해도 무조건 1초에 한 번씩 메소드를 호출한다.
```java
@Scheduled(fixedRate = 1000)
```

다음으로는 `메소드 종료~다음 메소드 호출 사이의 간격`을 정해주는 `fixedDelay`가 있다. 단위는 똑같이 ms이다. 이 옵션은 메소드가 끝나고 나서부터 다음 메소드의 간격을 설정하는 옵션이기 때문에 메소드의 실행 시간이 중요하다. 즉 해당 메소드가 10초 걸린다면, 첫 메소드 호출 후 다음 메소드가 호출되는 타이밍은 11초 뒤이다.
```java
@Scheduled(fixedDelay = 1000)
```

시간 간격을 정하는 게 아니라 원하는 시간/날짜마다 호출하고 싶다면 cron을 사용하면 된다. cron에는 6개의 숫자(또는 와일드카드)를 설정할 수 있는데, 순서대로 `초(0-59), 분(0-59), 시간(0-23), 일(1-31), 월(1-12), 요일(0-7)`이다. 이때 각 자리에 와일드카드 `*`나 `?`를 써서 모든 요일이나 시간 등을 나타내는 게 가능하다. 예를 들어서 아래의 `0 0 12 * * *`는 "매일 12:00"라는 뜻이다.
```java
// 초, 분, 시간, 일, 월, 요일
@Scheduled(cron = "0 0 12 * * *")
```

좀 더 다양한 예제는 다음과 같이 볼 수 있다.
```java
"0 0 * * * *" = 매일 매 시간 정각마다
"*/10 * * * * *" = 매 10초마다
"0 0 8-10 * * *" = 매일 8, 9, 10시마다(8-10 표현식은 "8시부터 10시까지"를 의미)
"0 0 8,10 * * *" = 매일 8, 10시마다(8,10 표현식은 "8시, 10시 둘 다"를 의미)
"0 0/30 8-10 * * *" = 매일 8:00, 8:30, 9:00, 9:30, 10:00마다(0/30 표현식은 "시작 시간부터 30분 간격으로"를 의미)
"0 0 9-17 * * MON-FRI" = 주중의 오전 9시~오후 5시 정각마다
"0 0 0 25 12 ?" = 크리스마스마다
```

## 레퍼런스
---

- [spring.io의 scheduling-tasks](https://spring.io/guides/gs/scheduling-tasks)
- [tistory의 @Scheduled 어노테이션 사용법](https://rooted.tistory.com/12)
- [stackoverflow의 spring-cron-expression](https://stackoverflow.com/questions/26147044/spring-cron-expression-for-every-day-101am)

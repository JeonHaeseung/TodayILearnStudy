## TIL - 스프링 @Autowired에 대해
---
스프링의 `@Autowired` 어노테이션은 dependency injection(DI)를 진행해주는 어노테이션이다. 이러한 방식의 DI를 annotations-driven Dependency Injection라고 부른다. 이는 bean(즉, dependency)를 자동으로 내 class 안으로 wire 가능하도록 하기 때문에, 수동으로 configuration할 필요가 없다. 사용자가 수동으로 injection 해줘야 했던 일들을 spring이 자동으로 해주기 때문이다.
스프링은 코드를 스캔해서 DI가 필요한 코드를 찾아낸 다음, application context에서 bean을 검색해서 종속성과 일치하는 bean을 찾아낸다. 이렇게 `@Autowired` 어노테이션을 사용해서 injection을 하는 방법에는 필드, setter, 생성자, collection, lombok 등 다양하게 존재한다.

1. 필드 인젝션

아래 코드는 `@Autowired` 어노테이션을 사용하는 가장 일반적인 케이스이다.  아래 코드를 스캔한 spring은 자동으로 `MemberRepository` bean을 찾아내서 종속성을 주입해준다. 이 방법은 간편하지만 필드 인젝션이므로 바꿀 수 없다는 점이 단점이다.
``` java
@Service
public class MemberService {
    @Autowired
    private MemberRepository memberRepository;
}
```

2. Setter 인젝션

Setter 인젝션은 필드 인젝션의 단점을 극복했다. 그러나 이 방식의 치명적인 단점은 누군가가 애플리케이션 실행 시점에 repo를 바꿀 수 있다는 점이다. 실제로는 스프링이 올라오는 타이밍에 이러한 bean 관련 세팅 및 조립이 전부 끝나기 때문에 바꿀 일이 없다는 것을 고려하면 이렇게 만들 필요가 없다.
``` java
@Service
public class MemberService {
    private MemberRepository memberRepository;

    @Autowired
    public void setMemberRepository(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }
}
```
하지만 이 방식은 optional dependencies와 같은 몇몇 예외적인 상황에 대해서는 유용하게 사용될 수도 있다. optional dependencies와 같은 경우에는 `@Autowired(required = false)` 옵션을 추가해 optional이라는 것을 명시할 수 있다.

3. 생성자 인젝션
   
이 경우에는 서버가 올라갈 때 스프링에서 인젝션을 담당한다. 생성자를 통한 인젝션은 처음에 세팅이 전부 끝나기 때문에 중간에 `set`해서 바꿀 수는 없다. 또한 테스트 케이스 같은 것을 작성할 때, 직접 주입을 해줘야 한다(`new MemberService(Mock())` 이런 식으로). 그래서 의존관계가 어떤지를 놓치지 않을 수 있다.
``` java
@Service
public class MemberService {
    private MemberRepository memberRepository;

    @Autowired
    public void MemberService(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }
}
```

4. collection 인젝션

여러 개의 bean을 한꺼번에 만들 수도 있다. `interface Plugin`의 여러 구현체가 있을 경우, 스프링은 이 구현체에 대한 bean을 모두 만든 다음 자동으로 `PluginManager`에 인젝션해준다.
``` java
@Component
public class PluginA implements Plugin {
    // Implementation of PluginA
}

@Component
public class PluginB implements Plugin {
    // Implementation of PluginB
}

@Component
public class PluginManager {
    @Autowired
    private List<Plugin> plugins;
}
```

5. 롬복 인젝션

롬복의 `@RequiredArgsConstructor` 어노테이션을 쓰면 `final`이 있는 argument의 생성자를 자동으로 만들어준다. `final`을 붙이면 반드시 생성자를 만들라고 요구하므로, 해당 어노테이션을 붙여서 생성자가 있는지 없는지 체크해주는 게 좋다.
``` java
@RequiredArgsConstructor //final이 있는 argument의 생성자를 자동으로 만들어준다.
...
private final MemberRepository memberRepository;
```

## QnA
---


## 레퍼런스
---
- [LinkedIn의 autowired-annotation](https://www.linkedin.com/pulse/autowired-annotation-spring-william-haywood/)
- [baeldung의 spring-autowire](https://www.baeldung.com/spring-autowire)
- [medium의 a-deep-dive-into-autowired-annotation](https://medium.com/@AlexanderObregon/a-deep-dive-into-autowired-annotation-in-the-spring-framework-2defc273ba59)


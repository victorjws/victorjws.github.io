# 좋은 객체 지향 프로그래밍
## 다형성 (Polymorphism)
### 역할과 구현을 분리
- 역할과 구현으로 구분하면 세상이 **단순**해지고, **유연**해지며 **변경도 편리**해진다.
- 장점:
    - client는 대상의 역할(interface)만 알면 된다.
    - client는 구현 대상의 내부 구조를 몰라도 된다.
    - client는 구현 대상의 내부 구조가 변경되어도 영향을 받지 않는다.
    - client는 구현 대상의 내부 구조가 변경되어도 영향을 받지 않는다.
    - client는 구현 대상 자체를 변경해도 영향을 받지 않는다.
- 자바 언어의 다형성을 활용
    - 역할: interface
    - 구현: interface를 구현한 class, 구현 객체
- 객체를 설계할 때 역할과 구현을 명확히 분리
- 객체 설계 시 역할(interface)을 먼저 부여하고, 그 역할을 수행하는 구현 객체 만들기

### 객체의 협력이라는 관계부터 생각
- 혼자 있는 객체는 없다.
- client: 요청, server: 응답
- 수 많은 객체 client와 객체 server는 서로 협력 관계를 가진다.

### java의 다형성
#### overriding

### 정리
- 실세계의 역할과 구현이라는 편리한 concept을 다형성을 통해 객체 세상으로 가져올 수 있음
- 유연하고, 변경이 용이
- 확장 가능한 설계
- client에 영향을 주지 않는 변경 가능
- interface를 안정적으로 잘 설계하는 것이 중요

### 한계
- 역할(interface) 자체가 변하면, client, server 모두에 큰 변경이 발생한다.
- 자동차를 비행기로 변경해야 한다면?
- 대본 자체가 변경된다면?
- USB interface가 변경된다면?
- interface를 안정적으로 잘 설계하는 것이 중요

### spring과 객체 지향
- 다형성이 가장 중요하다!
- spring은 다형성을 극대화해서 이용할 수 있게 도와준다.
- spring에서 이야기하는 제어의 역전(IoC), 의존관계 주입(DI)은 다형성을 활용해서 역할과 구현을 편리하게 다룰 수 있도록 지원한다.
- spring을 사용하면 마치 레고 블럭 조립하듯이! 공연 무대의 배우를 선택하듯이! 구현을 편리하게 변경할 수 있다.

## 좋은 객체 지향 설게의 5가지 원칙 (SOLID)
### SRP (Single responsibility principle, 단일 책임 원칙)
- 한 class는 하나의 책임만 가져야 한다.
- 하나의 책임이라는 것은 모호하다.
    - 클 수 있고, 작을 수 있다.
    - 문맥과 상황에 따라 다르다.
- **중요한 기준은 변경**이다. 변경이 있을 때 파급 효과가 적으면 단일 책임 원칙을 잘 따른 것
- 예) UI 변경, 객체의 생성과 사용을 분리

### OCP (Open/closed principle, 개방-폐쇄 원칙)
- software 요소는 **확장에는 열려**있으나 **변경에는 닫혀**있어야 한다
- 이런 거짓말 같은 말이? 확장을 하려면, 당연히 기존 코드를 변경?
- **다형성**을 활용해보자
- interface를 구현한 새로운 class를 하나 만들어서 새로운 기능을 구현
- 지금까지 배운 역할과 구현의 분리를 생각해보자
#### 문제점
```java
public class MemberService {
    // private MemberRepository memberRepository = new MemoryMemberRepository();
    private MemberRepository memberRepository = new JdbcMemberRepository();
}
```

- MemberService client가 구현 class를 직접 선택
    - MemberRepository m = new MemoryMemberRepository(); // 기존 코드
    - MemberRepository m = new JdbcMemberRepository(); // 변경 코드
- **구현 객체를 변경하려면 client code를 변경해야 한다.**
- **분명 다형성을 사용했지만 OCP 원칙을 지킬 수 없다.**
- 이 문제를 어떻게 해결해야 하나?
- 객체를 생성하고, 연관관계를 맺어주는 별도의 조립, 설정자가 필요하다.

### LSP (Liskov substitution principle, 리스코프 치환 원칙)
- program의 객체는 program의 정확성을 깨뜨리지 않으면서 하위 type의 instance로 바꿀 수 있어야 한다
- 다형성에서 하위 class는 interface 규약을 다 지켜야 한다는 것, 다형성을 지원하기 위한 원칙, interface를 구현한 구현체는 믿고 사용하려면, 이 원칙이 필요하다.
- 단순히 compile에 성공하는 것을 넘어서는 이야기
- 예) 자동차 interface의 accelerator는 앞으로 가라는 기능, 뒤로 가게 구현하면 LSP 위반, 느리더라도 앞으로 가야함

### ISP (Interface segregation principle, 인터페이스 분리 원칙)
- 특정 client를 위한 interface 여러 개가 범용 interface 하나보다 낫다
- 자동차 interface -> 운전 interface, 정비 interface로 분리
- 사용자 client -> 운전자 client, 정비사 client로 분리
- 분리하면 정비 interface 자체가 변해도 운전자 client에 영향을 주지 않음
- interface가 명확해지고, 대체 가능성이 높아진다

### DIP (Dependency inversion principle, 의존관계 역전 원칙)
- programmer는 '추상화에 의존해야지, 구체화에 의존하면 안된다.' 의존성 주입 원칙은 이 원칙을 따르는 방법 중 하나다.
- 쉽게 이야기해서 구현 class에 의존하지 말고, interface에 의존하라는 뜻
- 앞에서 이야기한 **역할(Role)에 의존하게 해야 한다는 것과 같다**. 객체 세상도 client가 interface에 의존해야 유연하게 구현체를 변경할 수 있다. 구현체에 의존하게 되면 변경이 아주 어려워진다.
- 그런데 OCP에서 설명한 MemberService는 interface에 의존하지만, 구현 class도 동시에 의존한다.
- MemberService client가 구현 class를 직접 선택
    - MemberRepository m = new MemoryMemberRepository();
- DIP 위반

### 정리
- 객체 지향의 핵심은 다형성
- 다형성 만으로는 쉽게 부품을 갈아 끼우듯이 개발할 수 없다
- 다형성 만으로는 구현 객체를 변경할 때 client code도 함께 변경된다.
- **다형성 만으로는 OCP, DIP를 지킬 수 없다**
- 뭔가 더 필요하다

# Spring
- spring은 다음 기술로 다형성 + OCP, DIP를 가능하게 지원
    - DI (Dependency Injection): 의존관계, 의존성 주입
    - DI container 제공
- **client code의 변경 없이 기능 확장**
- 쉽게 부품을 교체하듯이 개발

# 정리
- 모든 설계에 역할과 구현을 분리하자
- 자동차, 공연의 예를 떠올려보자
- application 설계도 공연을 설계 하듯이 배역만 만들어두고, 배우는 언제든지 **유연**하게 **변경**할 수 있도록 만드는 것이 좋은 객체 지향 설계다
- 이상적으로는 모든 설계에 interface를 부여하자
## 실무 고민
- 하지만 interface를 도입하면 추상화라는 비용이 발생한다
- 기능을 확장할 가능성이 없다면, 구체 class를 직접 사용하고, 향후 꼭 필요할 때 refactoring해서 interface를 도입하는 것도 방법이다 

# reference
## 책 추천
- 객체지향 책 추천: 객체지향의 사실과 오해
- 스프링 책 추천: **토비의 스프링**
- JPA 책 추천: 자바 ORM 표준 JPA 프로그래밍


# DI (Dependency Injection)
- field injection, setter injection, constructor injection 3가지 방법이 있다. 의존관계가 동적으로 변하는 경우는 거의 없으므로 constructor injection을 권장한다.
```java
// field injection
@Controller
public class MemberController {
    @Autowired private MemberService memberService;
}
```
```java
// setter injection
@Controller
public class MemberController {
    private MemberService memberService;

    @Autowired
    public void setMemberService(MemberService memberService) {
        this.memberService = memberService;
    }
}
```
public으로 호출 할 수 있어 중간에 바꿔치기가 가능하다
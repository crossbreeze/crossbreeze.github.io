---
layout: post
title: "field injection vs constructor injection with Java Spring's @Autowired"
date: 2014-04-08 19:36:44 +0900
comments: true
categories: java spring di test injection
---
Dependency injection은 스프링이나 EJB 같은 IoC container의 핵심 기술이고 이미 다들 사용하고 계시는걸로 알고 있는데 이번에 스프링이 제공하는 인젝션 방식에 대해 알아보고 어떻게 사용하는것이 좋을지 정리해봅니다.

<!-- more -->

Spring에서는 xml 설정 또는 @Autowired annotation을 이용한 의존관계 주입을 제공합니다.

```java
// Awesome! Spring injects CustomerRepository for me
@Autowired
CustomerRepository customerRepository;
```

List type 필드에 @Autowired을 사용할 경우 해당 type에 만족하는 모든 빈들을 주입해줍니다. 예를 들어 다음과 같이 Pusher interface를 구현하는 2개의 클래스 ApplePusher와 GooglePusher가 있다고 하면

```java
public interface Pusher {
     boolean push(String message);
}
```
```java
public class ApplePusher implements Pusher {
    @Override
    public boolean push(String message) {
        // send push notification to iOS device
    }
}

public class GooglePusher implements Pusher {
    @Override
    public boolean push(String message) {
        // send push notification to Android device
    }
}
```

```java
// pushers will contain each instance of ApplePusher and GooglePusher
@Autowired
List<Pusher> pushers;
```
위와 같은 방식으로 필드에 @Autowired annotation으로 의존관계를 주입하는 방식이 보통 DI를 이용하는 방식의 99% 정도를 차지하고 있습니다.

하지만 이런식의 필드 주입 방식은 제일 간단해 보이긴 하지만 나쁜 스타일이라고 여겨집니다.

> Field injections is evil… hides dependencies, instead of making them explicit.

또 class들끼리의 coupling이 높아지고 Spring framework에 대한 의존성이 높아지게 됩니다. 이로 인해서 유닛 테스팅이 힘들어진다는 크나큰 단점이 있습니다.

예를 들어서 위 pusher들을 사용하는 클래스를 정의해보고 이 클래스를 테스트 해보도록 하겠습니다.
```java
@Service
public class AdminService {
    @Autowired
    private List<Pusher> pushers;

    public doSomething() {
        // 여기서 무엇인가를 하고 pusher들을 사용해 푸시 메시지를 보내야한다고 가정합니다.

        for (Pusher pusher : pushers) {
            pusher.push("did something");
        }
    }
}
```
```java
public class AdminServiceTest {
    @Test
    public void doSomethingTest() {
        AdminService adminService = new AdminService();
        adminService.doSomething();

        Assert.assertEquals(adminService.didSomething, true);
    }
}
```
위와 같은 테스트 방식으로는 테스트 대상 class의 @Autowired를 사용한 필드들이 모두 null이므로 테스트가 돌아가지 않을 것입니다.

따라서 저희는 아래와 같이 테스트들을 Spring conext안에서 돌리고 있습니다.
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:/spring/spring-context.xml"})
public class AdminServiceTest extends AbstractJUnit4SpringContextTests {
	// same test cases
}
```
아주 간단한 유닛 테스팅임에도 불구하고 Spring 설정 파일을 읽어서 모든 bean 설정이 끝나야지만(경우에 따라서 오래 걸릴수도 있다는게 함정) 테스트가 시작되게됩니다. 이를 해결하기 위해 범위가 작게 적용된 테스트용 spring context를 만들어서 제공하거나 또는 Mockito library를 이용해서 Autowired filed injection을 흉내내어 테스트 할수 있습니다.(http://java.dzone.com/articles/use-mockito-mock-autowired) 하지만 둘다 좋은 방법이라고는 할 수 없습니다.

더 좋은 방법은 constructor injection(추천) 또는 setter injection(비추천)을 사용하는 것입니다. 어떤 객체가 올바로 동작하기 위해 반드시 의존관계가 필요한 경우라면 위와 같이 필드 주입 방식을 사용할 이유가 전혀 없습니다. 생성자 주입방식을 사용할 경우 올바른 의존관계를 넘어오지 않으면 compile에러가 나므로 위험요소를 미리 없앨수 있습니다. 또한 테스트를 작성할 경우에도 테스트 코드 자체에서 필요한 의존관계를 만들어서(직접 만들던지 아니면 mock을 해서 주던지) 넘겨줄수 있게 됩니다.

사용방법은 간단합니다. 생성자를 만들고 @Autowired annotation을 생성자 앞에 붙여주기만 하면 됩니다. 위에 AdminService를 생성자 주입을 사용해서 다시 만든다고 하면 다음과 같이 되겠죠

```java
@Service
public class AdminService {
    private List<Pusher> pushers;

    @Autowired
    public AdminService(List<Pusher> pushers) {
        // spring context안에서는 필드 주입 방식과 똑같이 알맞은 빈들이 생성되서 주입될 것입니다.
        this.pushers = pushers;
    }

    public doSomething() {
        // 여기서 무엇인가를 하고 pusher들을 사용해 푸시 메시지를 보내야한다고 가정합니다.

        for (Pusher pusher : pushers) {
            pusher.push("did something");
        }
    }
}
```
이제 테스트에서도 pusher을 직접 만들거나 Mockito등의 mock library를 사용해서 넘겨줄수 있으므로 원하는 방식으로 테스트 할수 있고 spring context안에서 돌리지 않아도 되므로 더 빠르게 테스트를 돌릴 수 있습니다.

옵셔널한 의존관계일 경우 null 값을 가지던가 아니면 생서자에서 올바른 기본값으로 설정해두는 방식으로 코드를 짜고 꼭 필요한 의존관계의 경우 생성자에서 인자로 받는 방법을 사용하는 것을 추천합니다. 또 생성자에서 받아야 하는 의존관계가 5개 이상되어서 지저분해 보인다하면 아마도 코드 리팩토링이 필요한 시기가 아닌가 점검해볼 수 있는 기회가 됩니다.

왜 필드 주입 방식이 나쁜 코드 스타일인지 설명하는 포스트

[http://www.petrikainulainen.net/software-development/design/why-i-changed-my-mind-about-field-injection/](http://www.petrikainulainen.net/software-development/design/why-i-changed-my-mind-about-field-injection/)

추가로 Constructor Injection vs. Setter Injection에 관한 부분은 아래 링크에서 더 알아보실수 있습니다.

[http://misko.hevery.com/2009/02/19/constructor-injection-vs-setter-injection/](http://misko.hevery.com/2009/02/19/constructor-injection-vs-setter-injection/)

[http://java.dzone.com/articles/repeat-after-me-setter](http://java.dzone.com/articles/repeat-after-me-setter)

[http://spring.io/blog/2007/07/11/setter-injection-versus-constructor-injection-and-the-use-of-required](http://spring.io/blog/2007/07/11/setter-injection-versus-constructor-injection-and-the-use-of-required)

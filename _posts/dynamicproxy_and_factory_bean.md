---
layout: post
title: dynamic proxy and factory bean
---
# 다이내믹 프록시와 팩토리 빈
## 프록시와 프록시 패턴, 데코레이터 패턴

이전에.. 부가기능 전부를 핵심 코드가 담긴 클래스에서 독립시켰다.
(그림 6-8참고)

> 문제는 이렇게 구성했더라도 클라이언트가 핵심기능을 가진 클래스를 직접 사용해 버리면 부가기능이 적용될 기회가 없다는 점이다.

부가기능은 마치 자신이 핵심 기능을 가진 클래스인 것처럼 꾸며서, 클라이언트가 자신을 거쳐서 핵심기능을 사용하도록 만들어야 한다.  대리인, 대리자와 같은 기능을 한다고 해서 **프록시** 라고 부른다.
(그림 6-9, 6-10)

**프록시는 사용 목적에 따라 두 가지로 구분할 수 있다.**
1. 클라이언트가 타깃에 접근하는 방법을 제어하기 위해서
2. 타깃에 부가적인 기능을 부여하기 위해서

### 데코레이터 패턴
타깃에 부가적인 기능을 런타임 시 다이내믹하게 보여주기 위해 프록시를 사용하는 패턴
[데코레이터 패턴 참고 블로그](https://jdm.kr/blog/78)

> 스프링에서는 DI를 이용해서 데코레이터 패턴을 구성할 수 있다.
```
<!--데코레이터-->  
<bean id="userService" class="com.dahye.user.service.UserServiceTx">  
 <property name="userService" ref="userServiceImpl" />  
 <property name="transactionManager" ref="transactionManager" />  
</bean>  
  
<!--타깃 DI를 이용한 다이나믹한 구성-->  
<bean id="userServiceImpl" class="com.dahye.user.service.UserServiceImpl">  
 <property name="userDao" ref="userDao" />  
 <property name="mailSender" ref="mailSender" />  
</bean>
```
- 데코레이터 패턴은 인터페이스를 통해 위임하는 방식이기 때문에 어느 데코레이터에서 타깃으로 연결될지 코드 레벨에선 미리 알 수 없다.
- 타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용한 방법이다.

### 프록시 패턴
프록시 패턴은 **접근 방법만 제어**할 뿐, 타깃의 기능을 확장하거나 추가하지 않는다.
[프록시패턴 참고 블로그](http://limkydev.tistory.com/79)

예) Collections의 unmodifiableCollection()
- 타깃의 기능 자체에는 관여하지 않으면서 접근하는 방법을 제어한다.
```
static class UnmodifiableCollection<E> implements Collection<E>, Serializable {  
    private static final long serialVersionUID = 1820017752578914078L;  
  
 final Collection<? extends E> c;  
  
  UnmodifiableCollection(Collection<? extends E> c) {  
        if (c==null)  
            throw new NullPointerException();  
 this.c = c;  
  }  
  // ...
    public boolean add(E e) {  
        throw new UnsupportedOperationException();  
  }  
    public boolean remove(Object o) {  
        throw new UnsupportedOperationException();  
  }  
  // ...
}
```

## 다이내믹 프록시
프록시를 만드는 일이 상당히 번거롭게 느껴진다. 매번 새로운 클래스를 정의해야 하고, 인터페이스를 구현해야 할 메소드가 많으면 모든 메소드를 일일히 구현해서 위임하는 코드를 넣어야 하기 때문이다.
> 프록시를 일일이 모든 인터페이스를 구현해서 클래스를 새로 정의하지 않고도 편리하게 사용할 방법은 없을까 ❔

프록시는 다음의 두가지 기능으로 구성된다.
- 타깃과 같은 메소드를 구현하고 있다가 메소드가 호출되면 타깃 오브젝트로 위임한다.
- 지정된 요청에 대해서는 부가기능을 수행한다.
```
public class UserServiceTx implements UserService {  
  
    UserService userService;  //타깃 오브젝트
  
    // 메소드 구현과 위임
    public void add(User user) {  
        userService.add(user);  
  }  
  
    // 부가기능 수행 + 위임
    public void upgradeGrades() {  
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());  
  
 try {  
            userService.upgradeGrades();  
  
  transactionManager.commit(status);  
  } catch (RuntimeException e) {  
            transactionManager.rollback(status);  
 throw e;  
  }  
    }  
}
```
❔ 번거로운 이유는
1. add () 메소드와 같이 .. 부가기능이 필요 없는 메소드도 구현해서 타깃으로 위임하는 코드를 일일이 만들어주어야 한다.
2. 부가기능 코드가 중복될 가능성이 많다. (트랜잭션 코드가 모든 메서드에 적용되는 경우)


### 리플렉션
번거로운 문제를 해결하는 데 유용한 것이 바로 **JDK의 다이내믹 프록시**다.
다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다.
> ❔ 리플렉션이란 
> 구체적인 클래스 타입을 알지 못해도, 그 클래스의 메소드, 타입, 변수들을 접근할 수 있도록 해주는 API
```
public class ReflectionTest {  
    @Test  
  public void invokeMethod() throws Exception {  
        String name = "spring";  
  
  //length()  
  assertThat(name.length(), is(6));  
  
  Method lengthMethod = String.class.getMethod("length");  
  assertThat((Integer)lengthMethod.invoke(name), is(6));  
  
  //charAt()  
  assertThat(name.charAt(0), is('s'));  
  
  Method charAtMethod = String.class.getMethod("charAt", int.class);  
  assertThat((Character)charAtMethod.invoke(name, 0), is('s'));  
  
  }  
}
```

### 프록시 클래스
다이내믹 프록시를 이용한 프록시를 만들어보자.
```
public interface Hello {  
    String sayHello(String name);  
  String sayHi(String name);  
  String sayThankYou(String name);  
}
```
```
public class HelloTarget implements Hello {  
    public String sayHello(String name) {  
        return "Hello " + name;  
  }  
  
    public String sayHi(String name) {  
        return "Hi " + name;  
  }  
  
    public String sayThankYou(String name) {  
        return "Thank You " + name;  
  }  
}
```
 프록시 
```
public class HelloUppercase implements Hello {  
  
    Hello hello;  
  
 public HelloUppercase(Hello hello) {  
        this.hello = hello;  
  }  
  
    public String sayHello(String name) {  
        return this.hello.sayHello(name).toUpperCase();  
  }  
  
    public String sayHi(String name) {  
        return this.hello.sayHi(name).toUpperCase();  
  }  
  
    public String sayThankYou(String name) {  
        return this.hello.sayThankYou(name).toUpperCase();  
  }  
}
```
```
@Test  
public void simpleProxyUppercase() {  
    Hello hello = new HelloUppercase(new HelloTarget());  
  assertThat(hello.sayHello("dahye"), is("HELLO DAHYE"));  
  assertThat(hello.sayHi("dahye"), is("HI DAHYE"));  
  assertThat(hello.sayThankYou("dahye"), is("THANK YOU DAHYE"));  
}
```

#### 문제점
- 인터페이스의 **모든 메소드를 구현해 위임**하도록 코드를 만들어야 한다.
- 부가기능인 리턴 값을 대문자로 바꾸는 기능이 **모든 메소드에 중복** 된다.

### 다이내믹 프록시 적용
클래스로 만든 프록시인 HelloUppercase를 다이내믹 프록시를 이용해 만들어보자.
> 다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트다.

(그림 6-13)
#### UppercaseHandler.java 
```
public class UppercaseHandler implements InvocationHandler {  
  
    Hello target;  
  
 public UppercaseHandler(Hello target) {  
        this.target = target;  
  }  
  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
        String ret = (String) method.invoke(target, args); // 타깃으로 위임
  
 return ret.toUpperCase();  
  }  
}
```
- 다이내믹 프록시가 클라이언트로 부터 받는 모든 요청은  invoke() 메소드로 전달된다.
- 다이내믹 프록시를 통해 요청이 전달되면 리플렉션 API를 이용해 타깃 오브젝트의 메소드를 호출한다.
- 타깃 인터페이스의 모든 메소드 요청이 하나의 메소드로 집중되기 때문에 **중복되는 기능을 효과적으로 제공할 수 있다.**


#### 다이내믹 프록시 생성 (Proxy 클래스의 newProxyInterface() 메소드 이용)
```
@Test  
public void simpleProxyWithHandler() {  
    Hello hello = (Hello) Proxy.newProxyInstance(  
	         getClass().getClassLoader(),  
			 new Class[] {Hello.class},  //구현할 인터페이스
			 new UppercaseHandler(new HelloTarget()));//부가 기능과 위임 코드를 담은 InvocationHandler
  
  assertThat(hello.sayHello("dahye"), is("HELLO DAHYE"));  
  assertThat(hello.sayHi("dahye"), is("HI DAHYE"));  
  assertThat(hello.sayThankYou("dahye"), is("THANK YOU DAHYE"));  
}
```
> 다이내믹 프록시 생성 방법을 적용했지만, 코드의 양도 줄어든 것 같지 않고, 코드 작성은 더욱 까다로워진 것 같다.

### 다이내믹 프록시의 확장
> 다이내믹 프록시 방식이 직접 정의한 프록시보다 훨씬 유연하고 많은 **장점**이 있다. **메소드가 추가될 때마다 프록시의 코드를 매번 추가하는 일이 없어진다.**
##### 스트링 외의 리턴 타입을 갖는 메소드가 추가된다면 ?
```
public class UppercaseHandler implements InvocationHandler {  
  
    Object target;  
  
  // 어떤 종류의 인터페이스를 구현한 타겟에도 적용 가능하도록 Object타입으로 수정  
  public UppercaseHandler(Object target) {  
        this.target = target;  
  }  
  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
        Object ret = method.invoke(target, args);  
  
  //String 인 경우, 특정 메소드 인 경우에만 대문자 변경하도록  
  if(ret instanceof String && method.getName().startsWith("say")) {  
            return ((String)ret).toUpperCase();  
  } else {  
            return ret;  
  }  
    }  
}
```
## 다이내믹 프록시를 이용한 트랜잭션 부가기능
UserServiceTx를 다이내믹 프록시 방식으로 변경해보자.
```
public class TransactionHandler implements InvocationHandler {  
  
    private Object target;  
 private PlatformTransactionManager transactionManager;  
 private String pattern;  
  
 ...
  
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {  
        if(method.getName().startsWith(pattern)) {  
            return invokeInTransaction(method, args);  
  } else {  
            return method.invoke(target, args);  
  }  
    }  
  
    public Object invokeInTransaction(Method method, Object[] args) throws Throwable {  
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());  
  
 try {  
		   //리플렉션 API 사용
            Object ret = method.invoke(target, args);  
  transactionManager.commit(status);  
 return ret;  
  } catch (InvocationTargetException e) {  
            transactionManager.rollback(status);  
 throw e.getTargetException();  
  }  
    }  
}
```

```
@Test  
public void upgradeAllOrNothing() throws Exception {  
   ...
  TransactionHandler txHandler = new TransactionHandler();  
  txHandler.setTarget(testUserService);  
  txHandler.setTransactionManager(transactionManager);  
  txHandler.setPattern("upgradeLevels");  
  // UserService 인터페이스 타입의 다이내믹 프록시 생성
  UserService txUserService = (UserService) Proxy.newProxyInstance(  
            getClass().getClassLoader(), new Class[] {UserService.class}, txHandler);  
  
  userDao.deleteAll();  
 for(User user : users) userDao.add(user);  
  
 try {  
        txUserService.upgradeLevels();  
  fail("TestUserServiceException expected");  
  } catch (TestUserServiceException e) {  
  
    }  
  
    checkGradeUpgrade(users.get(1), false);  
}
```

## 다이내믹 프록시를 위한 팩토리 빈
TransactionHandler와 다이내믹 프록시를 스프링의 DI를 통해 사용할 수 있도록 만들어보자.
다이내믹 프록시 오브젝트의 클래스가 어떤 것인지 알 수 없다.
```
public class TransactionHandler implements InvocationHandler {  
  
    private Object target;  
	private PlatformTransactionManager transactionManager;  
    private String pattern;  
  
}
```
스프링은 클래스 정보를 가지고 디폴트 생성자를 통해 오브젝트를 만드는 방법 외에도 빈을 만들 수 있는 여러 가지 방법을 제공한다.
1. 팩토리 빈을 이용한 빈 생성
**팩토리 빈이란❔** 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈
 **팩토리 빈을 만드는 방법에는 여러가지가 있는데, 가장 간단한 방법은 스프링의 FactoryBean이라는 인터페이스를 구현하는 것이다**
 
 ***FactoryBean.java***
```
public interface FactoryBean<T> {  
    T getObject() throws Exception;  
    Class<?> getObjectType();  
    boolean isSingleton();  
  
}
```
### factoryBean 학습 테스트
***생성자를 제공하지 않는 클래스***
```
public class Message {  
    String text;  
  
    private Message(String text) {  
        this.text = text;  
  }  
  
    public String getText() {  
        return text;  
  }  
  
    // 생성자 대신 static메소드 제공
    public static Message newMessage(String text) {  
        return new Message(text);  
  }  
}
```
- private 생성자를 가진 클래스는 빈으로 등록하는 일은 권장되지 않는다.
- 즉, 
```
<bean id="m" class="....Message>
```
와 같이 사용하면 안된다.

```
public class MessageFactoryBean implements FactoryBean<Message> {  
  
    String text;  
  
  // 오브젝트를 생성할 때 필요한 정보는 DI 받을 수 있도록
  // 주입된 정보는 오브젝트 생성 중에 사용
 public void setText(String text) {  
        this.text = text;  
  }  
  
  // 실제 빈으로 사용될 오브젝트를 직접 생성
    public Message getObject() throws Exception {  
        return Message.newMessage(text);  
  }  
  
    public Class<?> getObjectType() {  
        return Message.class;  
  }  
  
  // getObject 메서드가 싱글톤인지 알려준다.
    public boolean isSingleton() {  
        return false;  
  }  
}
```
- 스프링은 FactoryBean 인터페이스를 구현한 클래스가 빈의 크래스로 지정되면, 팩토리 빈 클래스의 오브젝트의 **getObject() 메소드**를 이용해 오브젝트를 가져오고, 이를 빈 오브젝트로 사용한다.

따라서, 다음과 같이 빈을 설정할 수 있다.
 ```
 <bean id="message" class="com.dahye.learningtest.factorybean.MessageFactoryBean">  
	 <property name="text" value="Factory bean" />  
</bean>
 ```
 - **다른 빈 설정과 다른 점은 message 오브젝트의 타입이 class 애트리뷰트에 정의된 MessageFactoryBean이 아니라 Message 차입이라는 것이다.** 
 - Message 빈의 타입은 getObjectType()의 메소드가 반환하는 타입으로 결정된다.
```
@Test  
public void getMessageFromFactoryBean() {  
    Object message = context.getBean("message");  
  assertThat(message, is(Message.class));  
  assertThat(((Message)message).getText(), is("Factory bean"));  
}  
  
  // 팩토리 빈 자체를 가져오는 테스트
@Test  
public void getFactoryBean() throws Exception {  
    Object factory = context.getBean("&message");  
  assertThat(factory, is(MessageFactoryBean.class));  
  
}
```

### 다이내믹 프록시를 만들어주는 팩토리 빈
Proxy의 newProxyInstance() 메소드를 통해서만 생성이 가능한 다이내믹 프록시 오브젝트는 일반적인 방법으로는 빈으로 등록할 수 없다.
대신 **팩토리 빈**을 사용하면 다이내믹 프록시 오브젝트를 스프링의 빈으로 만들어줄 수가 있다.
> 팩토리 빈을 사용해서 스프링 빈으로 등록해보자
```
public class TxProxyFactoryBean implements FactoryBean<Object> {  
  
    // 범용적으로 사용하기 위해 Object로 선언
    Object target;  
  PlatformTransactionManager transactionManager;  
  String pattern;  
  //다이내믹 프록시를 생성할 때 필요 
  //(UserService 외의 인터페이스를 가진 타깃에도 적용할 수 있음)  
  Class<?> serviceInterface; 
  
 public void setTarget(Object target) {  
        this.target = target;  
  }  
  
    public void setTransactionManager(PlatformTransactionManager transactionManager) {  
        this.transactionManager = transactionManager;  
  }  
  
    public void setPattern(String pattern) {  
        this.pattern = pattern;  
  }  
  
    public void setServiceInterface(Class<?> serviceInterface) {  
        this.serviceInterface = serviceInterface;  
  }  
  
  //DI 주입받은정보를 이용해서 TransactionHandler를 사용하는 다이내믹 프록시 생성
    public Object getObject() throws Exception {  
        TransactionHandler txHandler = new TransactionHandler();  
  txHandler.setTarget(target);  
  txHandler.setTransactionManager(transactionManager);  
  txHandler.setPattern(pattern);  
  
 return Proxy.newProxyInstance(getClass().getClassLoader(), new Class[] {serviceInterface}, txHandler);  
  }  
  
  // 팩토리 빈이 생성하는 오브젝트 타입은 DI받은 인터페이스 타입에 따라 달라진다.
  // 다양한 타입의 프록시 오브젝트 생성에 재사용 할 수 있다.
    public Class<?> getObjectType() {  
        return serviceInterface;  
  }  
  
    public boolean isSingleton() {  
        return false;  
  }  
}
```
***빈 주입***
```
<bean id="userService" class="com.dahye.user.service.TxProxyFactoryBean">  
 <property name="target" ref="userServiceImpl" />  
 <property name="transactionManager" ref="transactionManager" />  
 <property name="pattern" value="upgradeLevels" />  
 <property name="serviceInterface" value="com.dahye.user.service.UserService" />  
</bean>
```
***테스트 코드***
```
@Test  
public void upgradeAllOrNothing() throws Exception {  
    ...
    // 팩토리 빈 자체를 가져와야 한다.
  TxProxyFactoryBean txProxyFactoryBean =  
            context.getBean("&userService", TxProxyFactoryBean.class);  
  txProxyFactoryBean.setTarget(testUserService);  
  UserService txUserService = (UserService) txProxyFactoryBean.getObject();  
  
  userDao.deleteAll();  
 for(User user : users) userDao.add(user);  
  
 try {  
        txUserService.upgradeLevels();  
  fail("TestUserServiceException expected");  
  } catch (TestUserServiceException e) {  
  
    }  
  
    checkGradeUpgrade(users.get(1), false);  
}
```
**👍🏻 해당 코드는 계속 재사용할 수 있다. 트랜잭션 부가기능이 필요한 빈이 추가될 때마다 빈 설정만 추가해주면 된다**

## 프록시 팩토리 빈 방식의 장점과 한계
### 프록시 팩토리 빈의 재사용
TransactionHandler를 이용하는 다이내믹 프록시를 생성해주는 TxProxyFactoryBean은 코드의 수정 없이도 다양한 클래스에 적용할 수 있다.
```
<bean id="coreServiceTarget" class="complex.module.CoreServiceImpl>
	<property name="coreDao" ref="coreDao" />
</bean>
```
***CoreService에 대한 트랜잭션 프록시 팩토리 빈***
```
<bean id="coreService" class="com.dahye.user.service.TxProxyFactoryBean">  
 <property name="target" ref="coreServiceTarget" />  
 <property name="transactionManager" ref="transactionManager" />  
 <property name="pattern" value="" /> <!--모든 메소드에 대해 트랜잭션 적용--> 
 <property name="serviceInterface" value="com.dahye.user.service.CoreService" />  
</bean>
```
### 프록시 팩토리 빈 방식의 장점
프록시 팩토리는 앞의 두 문제를 해결해준다.
- 타깃 인터페이스를 구현하는 클래스를 일일이 만드는 **번거로움을 제거**할 수 있다.
- 하나의 핸들러 메소드를 구현하는 것만으로도 수많은 메소드에 부가기능을 부여해줄 수 있어서 부가기능 코드의 **중복문제도 사라진다**.
- DI 설정만으로도 다양한 타깃 오브젝트에 적용할 수 있다.

### 프록시 팩토리 빈의 한계
- 하나의 클래스 안에 존재하는 여러 개의 메소드에 부가기능을 한 번에 제공하는 일은 쉬웠지만, **여러 개의 클래스에 공통적인 부가기능을 제공하는 일은 불가능하다.** (프록시 빈의 설정이 중복되는 것을 막을 수 없다)
- 하나의 타깃에 여러 개의 부가기능을 적용할 때도 문제이다. 설정 파일이 급격하게 복잡해 질 수 있다.
- TransactionHandler 오브젝트가 프록시 팩토리 빈 개수만큼 만들어진다.
> 모든 타깃에 적용 가능한 싱글톤 빈으로 만들어서 적용할 필요성이 느껴진다 !!

(참고) 토비의 스프링 3.1

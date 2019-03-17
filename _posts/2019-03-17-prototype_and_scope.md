## 1.3 프로토타입과 스코프
> 어떤 경우에 싱글톤 방식이 아닌 빈을 사용해야 할지 알아보자 !

### 스프링의 빈은 싱글톤으로 만들어진다.
= 애플리케이션 컨텍스트마다 빈의 오브젝트는 한 개만 만들어진다는 뜻이다.

### 싱글톤으로 설정된 빈은 DI(의존주입) 이든 DL(조회)이든 상관없이 매번 같은 오브젝트가 사용된다.
```
@Test
    public void singletonScope() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(SingletonBean.class, SingletonClientBean.class);
        Set<SingletonBean> beans = new HashSet<SingletonBean>();

        beans.add(ac.getBean(SingletonBean.class));
        beans.add(ac.getBean(SingletonBean.class));
        assertThat(beans.size(), is(1));

        beans.add(ac.getBean(SingletonClientBean.class).bean1);
        beans.add(ac.getBean(SingletonClientBean.class).bean2);
        assertThat(beans.size(), is(1));
    }

    static class SingletonBean {}
    static class SingletonClientBean{
        @Autowired
        SingletonBean bean1;

        @Autowired
        SingletonBean bean2;
    }
```

(싱글톤이 아닌 빈) ***프로토타입 스코프***

```
@Test
    public void prototypeScope() {
        ApplicationContext ac = new AnnotationConfigApplicationContext(PrototypeBean.class, PrototypeClientBean.class);
        Set<PrototypeBean> bean = new HashSet<PrototypeBean>();

        bean.add(ac.getBean(PrototypeBean.class));
        assertThat(bean.size(), is(1));

        bean.add(ac.getBean(PrototypeBean.class));
        assertThat(bean.size(), is(2));

        bean.add(ac.getBean(PrototypeClientBean.class).bean1);
        assertThat(bean.size(), is(3));
        bean.add(ac.getBean(PrototypeClientBean.class).bean2);
        assertThat(bean.size(), is(4));
    }

    @Scope("prototype")
    static class PrototypeBean {}

    static class PrototypeClientBean {
        @Autowired PrototypeBean bean1;
        @Autowired PrototypeBean bean2;
    }
```

▶ DL(의존객체 조회)이든 DI(의존성 주입)이든 상관없이 매번 새로운 오브젝트 생성

### 프로토타입 빈의 생명주기와 종속성
**IOC의 기본 원칙**은 오브젝트를 코드가 아니라 컨테이너가 관리한다는 것이다.

**프로토타입 빈**은 IOC 기본 원칙을 따르지 않는다. 요청이 있을 때마다 컨테이너가 생성하고 초기화하고 DI까지 해주기도 하지만 일단 빈을 제공하고 나면 컨테이너는 더 이상 빈 오브젝트를 관리하지 않는다.

한번 DL이나 DI를 통해 컨테이너 밖으로 전달된 후의 오브젝트는 **오브젝트를 가져간 코드나 DI로 주입받은 다른 빈이** 컨테이너가 제공한 빈 오브젝트를 관리하게 된다.

프로토타입 빈이 싱글톤이라면, 이 빈에 주입된 프로토타입 빈도 역시 싱글톤 생명주기를 따라서 컨테이너가 종료될 때까지 유지될 것이다.

### 프로토타입 빈은 도대체 어떤 경우에 사용할까 ?
DI를 해야하는 경우.

(매번 새롭게 만들어지는 오브젝트가 컨테이너 내의 빈을 사용해야 하는 경우가 있을 수 있다. DI를 이용하려면 사용하는 오브젝트는 컨테이너가 관리하는 빈이어야 한다.)

### (예제) 콜센터에서 고객의 A/S 신청을 받아서 접수하는 기능을 만든다고 정의

```
// a/s 신청 폼 클래스
public class ServiceRequest {
    String customerNo;
    String productNo;
    String description;
    ...
}
```

ServiceRequest는 매번 신청 받을 때마다 새롭게 생성, 서비스 계층으로 전달해 줘야 한다.

```
// 웹 컨트롤러
public void serviceRequestFormSubmit(HttpServletRequest request) {
    ServiceRequest serviceRequest = new ServiceRequest();
    serviceRequest.setCustomerNo(request.getParameter("custNo"));

    ...
    this.serviceRequestService.addNewServiceRequest(serviceRequest);
    ...
}
```

```
// 서비스 계층
public void addNewServiceRequest(ServiceRequest serviceRequest) {
    Customer customer = this.customerDao.findCustomerByNo(serviceRequest.getCustomerNo);
    ...
    this.serviceRequestDao.add(serviceRequest, customer);

    //접수가 완료된 고객에게 메일 알림을 준다고 가정
    this.emailService.sendEmail(customer.getEmail(), "A/S 접수가 정상으로 처리되었습니다.");
}
```

ServiceRequest 폼 오브젝트 사용 방식 →
![_config.yml]({{ site.baseurl }}/images/2019-03-17/prototype1.png)

▶ 문제 :: 폼의 고객정보 입력 방법이 모든 계층의 코드와 강하게 결합되어 있다는 점. (쉽게 말하자면, 현재 코드는 Customer의 no 값으로 정보를 가져오도록 설정되어 있으므로, id를 폼에서 전달하려고 변경하게 되면 코드가 전반적으로 수정되어야 한다 !)

▶ 전형적인 **데이터 중심의 아키텍처**가 만들어내는 구조이다.

### 모델 관점으로 코드를 변경해 보자 (더 오브젝트 중심의 구조, 좀 더 객체지향적으로 !)
모델 관점으로 보자면, 폼에서 어떻게 입력받는지에 따라 달라지는 (customerNo이나 customerId 같은 값) 값에 의존하고 있으면 안된다.

```
// 수정된 a/s 신청 폼 클래스
public class ServiceRequest {
    Customer customer; //customerNo 값 대신 Customer 오브젝트 자체를 갖고 있게 한다.
    String productNo;
    String description;
    ...

    @Autowired CustomerDao customerDao;

    public void setCustomerByCustomerNo(String customerNo) {
        this.customer = customerDao.findCustomerByNo(customerNo);
    }
}
```

SendRequest가 CustomerDao 를 DI받아서 사용할 수 있도록 변경한다. (SendRequest가 Customer 오브젝트의 정보를 가져오는 작업을 직접 처리하기 위해)

```
// 수정된 서비스 계층
public void addNewServiceRequest(ServiceRequest serviceRequest) {
    this.serviceRequestDao.add(serviceRequest);
    this.emailService.sendEmail(serviceRequest.getCustomer().getEmail(), "A/S 접수가 정상으로 처리되었습니다.");
}
```

고객번호로 고객을 찾아오는 번거로운 작업을 생략할 수 있게 되었다. (SendRequest가 Customer 오브젝트를 가지고 있으니까 ~)

▶ ServiceRequestService는 ServiceRequest의 Customer 객체가 어떻게 만들어졌는지에 대해서는 전혀 신경쓰지 않아도 된다.

**결론** )) customerId를 받아서 처리하도록 변경할 때는 아래와 같은 메서드만 추가해주면 수정은 끝난다.

```
public class ServiceRequest {
    ...
    public void setCustomerByCustomerId(String customerId) {
        this.customer = customerDao.findCustomerById(custocustomerIdmerNo);
    }
}
```

▶ 폼에서 고객 정보를 입력받는 방법을 어떻게 변경하든 ServiceRequest를 사용하는 서비스 계층이나 DAO의 코드는 전혀 영향받지 않게 된다.

ServiceRequest 가 UserDao를 DI하기 위해서는 결국,, **컨테이너에 오브젝트 생성을 맡겨야한다.**

우리의 ServiceRequest는 호출할 때마다 **매번 새로운 오브젝트**가 만들어지도록 해야하기 때문에, 이 때 바로 **프로토타입 스코프 빈**이 필요한 때이다.

### 프로토타입 빈으로 선언해보자

**어노테이션**
```
@Component
@Scope("prototype")
public class ServiceRequest {
    ...
```

**XML으로 빈 등록**
```
<bean id ="serviceRequest" class = "...ServiceRequest" scope = "prototype">
```

**프로토타입 빈을 가져오는 코드**

```
// 웹 컨트롤러
@Autowired
ApplicationContext context;

public void serviceRequestFormSubmit(HttpServletRequest request) {
    ServiceRequest serviceRequest = context.getBean(ServiceRequest.class);
    serviceReqeust.setCustomerByCustomerNo(request.getParameter("custNo"));
    ...
    this.serviceRequestService.addNewServiceRequest(serviceRequest);
    ...
}
```

**EmailService에 대해서도 생각해보자**

ServiceRequest가 자유롭게 DI를 받을 수 있는 빈이 되었으므로 EmailService를 ServiceRequest에 두도록 변경해보자

```
public class ServiceRequest {
    Customer customer;
    @Autowired EmailService emailService;

    ...
    public void notifyServiceRequestRegistration() {
        if(this.customer.serviceNotificationMethod == NotificationMethod.EMAIL) {
            this.emailService.sendEmail(customer.getEmail(), "A/S 접수가 정상으로 처리되었습니다.");
        }
    }
    ...
}
```

```
// 서비스 계층
public void addNewServiceRequest(ServiceRequest serviceRequest) {
    this.serviceRequestDao.add(serviceRequest);
    serviceRequest.notifyServiceRequestRegistration();
}
```
프로토타입 빈 ServiceRequest를 적용한 구조
![_config.yml]({{ site.baseurl }}/images/2019-03-17/prototype2.png)


> 매번 새롭게 오브젝트를 만들면서도 DI도 함께 적용하려고 할 때 사용할 수 있는 게 바로 프로토타입 빈이다.
필수는 아니지만, 좀 더 오브젝트 중심적이고 유연한 확장을 고려한다면 프로토타입 빈을 이용하는 편이 낫다.

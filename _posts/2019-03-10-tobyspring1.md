## 관심사의 분리
관심사의 분리를 객체지향에 적용해보면, 관심이 같은 것끼리는 하나의 객체 안으로 또는 친한 객체로 모이게 하고, 관심이 다른 것은 가능한 한 따로 떨어져서 서로 영향을 주지 않도록 분리하는 것
=> 관심과 중복의 제거 !!
## #1 중복 코드 메소드 추출
메서드가 2000개쯤 된다고 생각해보자. 해당 부분이 수정된다면 모든 2000개의 메서드를 모두 수정해야한다.
한 가지 관심에 대한 변경이 일어날 경우 그 관심이 집중되는 코드만 수정하면 되도록 구현하자.
### # before
```
public void add(User user) throws ClassNotFoundException, SQLException {
        Class.forName("oracle.jdbc.driver.OracleDriver");
        Connection c = DriverManager.getConnection("jdbc:oracle:thin:@ip:orcl", "id", "pwd");

        PreparedStatement ps = c.prepareStatement(
                "insert into users(id, name, password) values(?,?,?)"
        );
        ps.setString(1, user.getId());
        ps.setString(2, user.getName());
        ps.setString(3, user.getPassword());

        ps.executeUpdate();

        ps.close();
        c.close();
    }
    
    public User get(String id) throws ClassNotFoundException, SQLException {
        Class.forName("oracle.jdbc.driver.OracleDriver");
        Connection c = DriverManager.getConnection("jdbc:oracle:thin:@ip:1521:orcl", "id", "pwd");

        PreparedStatement ps = c.prepareStatement(
                "select * from users where id = ?"
        );
        ps.setString(1, id);

        ResultSet rs = ps.executeQuery();
        rs.next();

        User user = new User();
        user.setId(rs.getString("id"));
        user.setName(rs.getString("name"));
        user.setPassword(rs.getString("password"));

        rs.close();
        ps.close();
        c.close();

        return user;
    }
```
위의 코드는 총 세가지의 관심사항을 가지고 있다.
- DB 연결을 위한 커넥션을 가져오는 부분
- DB에 보낼 SQL 문장을 작성하여 DB와 통신하는 부분
- 사용한 리소스를 되돌려주는 부분

### # After. 커넥션을 가져오는 부분을 독립적인 메소드로 분리해보자.
```
public void add(User user) throws ClassNotFoundException, SQLException {
        Connection c = getConnection();
        ...
    }

    public User get(String id) throws ClassNotFoundException, SQLException {
        Connection c = getConnection();
        ...
    }

    //DB 커넥션의 변화가 있다면, 해당 메서드만 수정하면 됨.
    private Connection getConnection() throws ClassNotFoundException, SQLException {
        Class.forName("oracle.jdbc.driver.OracleDriver");
        Connection c = DriverManager.getConnection("jdbc:oracle:thin:@172.30.1.24:1521:orcl", "dahye", "dahye");

        return c;
    }
```

한가지의 관심 , 즉, DB 커넥션의 변화가 생긴다면 getConnection()메서드만 수정하면 된다.
이러한 메서드의 분리과정도 **refactoring(메소드 추출 기법)** 이라고 한다.


### #2 상속을 통한 확장
DB 커넥션 연결이라는 관심을 상속을 통해 서브클래스로 분리해버리는 것
#### # 만약에 커넥션 정보를 다양하게 쓴다고 정의해 본다면, getConnection()메서드를 추상메서드로 변경하여 사용할 수 있다.
```
public abstract Connection getConnection() throws ClassNotFoundException, SQLException;
```
▶ 서브 클래스에서 해당 클래스를 상속하여 getConnection()을 재정의하여 사용할 수 있게 된다.
하지만 !! 아직도 단점은 존재한다!!
- UserDao를 상속해야할 클래스가 이미 다른 클래스를 상속하고 있다면? :: java는 다중상속을 지원하지 않는다 !
- UserDao를 상속하고 있는 서브 클래스는 후에 다른 목적으로 상속을 적용하기 힘들다 :: 확장성에 불리
- 슈퍼클래스 내부의 변경이 있을 때, 모든 서브클래스를 함께 수정하거나 다시 개발해야할 수 있다.
- dao에 한정된 메소드이기 때문에, dao 클래스가 늘어나면 getConnection을 구현한 코드가 매 클래스마다 중복되어 나타나는 심각한 문제 발생

> 오히려, '상속'을 사용해서 생긴 단점이 더욱 보인다.


### #3 새로운 클래스로 변경
관심사를 독립시키면서 동시에 손쉽게 확장 할 수 있는 방법을 알아보자.
#### # DB커넥션을 위한 독립적인 클래스
```
public class SimpleConnectionMaker {
    public Connection makeNewConnection() throws ClassNotFoundException, SQLException {
        Class.forName("oracle.jdbc.driver.OracleDriver");
        Connection c = DriverManager.getConnection("jdbc:oracle:thin:@ip:1521:orcl", "id", "pwd");

         return c;
    }
}
```


#### # 사용 예
```
public class UserDao4 {
    private SimpleConnectionMaker simpleConnectionMaker;

    public UserDao4() {
        //한번만 만들어 인스턴스 변수에 저장해두고 메서드에서 사용하도록
        simpleConnectionMaker = new SimpleConnectionMaker();
    }
    public void add(User user) throws ClassNotFoundException, SQLException {
        Connection c = simpleConnectionMaker.makeNewConnection();
        ...
    }

    public User get(String id) throws ClassNotFoundException, SQLException {
        Connection c = simpleConnectionMaker.makeNewConnection();
        ...
    }
}
```
SimpleConnectionMaker라는 클래스에 DB커넥션이라는 기능을 종속적으로 만들어버린다.
### 참고. 템플릿 메소드 패턴
슈퍼클래스에 기본적인 로직의 흐름을 만들고, 그 기능의 일부를 추상 메소드나 오버라이딩이 가능한 protected 메소드 등으로 반든 뒤 서브클래스에는 이런 메소드를 필요에 맞게 구현해서 사용하도록 하는 방법의 디자인 패턴

최근 신규 프로젝트를 참여하게 되면서 대량의 데이터를 다루게 되고, 쿼리에 대한 성능에 신경쓰게 되었습니다. 기존에는 단순한 DML만 사용했었어서, 기본적인 개념일 수 있습니다.

## Union 과 Union all
- Union 을 쓰게된 이유

> Union은 두 개의 집합을 하나로 묶는 역할을 한다. Union 키워드 다음에 DISTINCT 또는 ALL이라는 키워드를 추가해서 중복 레코드에 대한 처리 방법을 선택할 수 있다. **UNION키워드 뒤에 아무것도 명시하지 않으면 기본적으로 DISTINCT가 적용된다.**

UNION 키워드만 쓰는 경우가 많을 것 같습니다. 
union만 쓰게되면 union distinct와 동일하게 적용되는데 mysql 서버에서 내부적으로 두 개의 집합에서 중복된 레코드를 제거한 후(두 집합의 레코드 가운데 중복된 레코드 중 하나는 버림) 합집합을 사용자에게 반환하게 됩니다.

- 사례(중복이 발생할 수 없는 것이 확실한 경우엔 ?)

만약 두 집합에서 중복된 결과가 있을 수 없다는 것이 보장된다면 UNION ALL 연산자를 이용해 MySQL 서버가 불필요하게 중복 제거 작업을 하지 않고 빨리 처리할 수 있게 됩니다.
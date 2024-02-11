---
title: "[postgresql]select 결과 아예 없을 때 Mybatis interceptor로 객체 초기화하는 방법"
date: 2024-02-11 09:34:00 +0900
categories:
  - Spring
  - Mybatis
tags:
  - mybatis
math: true
mermaid: true
image: /assets/img/post_previewImg/mybatis-logo.png
---


프로젝트를 수행 중 select 결과가 없는 경우 화면에서 나는 `undefined` error를 잡아야하는 상황이 발생했다.(당시 react로 화면을 구성하였음)

## 시도 1) `COALESCE`,  `NULLIF` 사용 

맨 처음 생각한 것은 `COALESCE`나 `NULLIF`를 사용해서 결과가 없을 때 null을 출력하도록 하는 방안이였다.

하지만 `COALESCE`와 `NULLIF`는 단일행 함수이다.
단일행 함수(Single-Row Function)란 각 행(Row)에 개별적으로 작동하여 각각의 행에 대한 조작 결과를 리턴하는 함수이다.

즉, 결과가 한 행이라도 있어야 해당 함수가 작동된다는 의미이다.
따라서 select 결과로 아무것도 조회되지 않는 경우에는 적용이 불가했다.

<br>

---
## 시도 2) 각 VO 객체 초기값 생성 및 각 select 조회 메소드 마다 null일 경우 객체 생성

한마디로 노가다이다....

모든 VO객체에 초기값을 생성하고 모든 select 조회 문마다 if조건문을 걸어서 조회 결과가 없을 경우 초기값이 설정된 VO객체로 return 하는 방안이였다.

당장의 문제들을 해결 할 수 있었지만, 프로젝트 전체에 적용하기에는 select가 정말 많았고 적용하면서도 이게 맞나 의문이 드는 방안이였다.
<span style="color: #808080">(결국 다 적용은 했었지만 현타가 왔었다...)</span>

<br>

---

## 해결방안) Mybatis interceptor 사용하기

계속 더 나은 방안을 모색하던 중 Mybatis에서 interceptor를 제공하는 것을 알게 되었다.

Mybatis에서 제공하는 interceptor는 매핑된 문장의 실행 중 특정 시점에서 호출을 가로챌 수 있도록 해준다.

즉, 우리가 특정시점만 잘 적용하면 해당 시점에 원하는 작동과 결과값을 구현할 수 있다.

<span style="color: #808080">Mybatis 3.5.1버전 기준입니다.</span>

### 1. interceptor 클래스 특정하기
일단 Mybatis의 interceptor를 사용하려면 `mybatis-config.xml`에 자신이 사용할 interceptor 클래스를 걸어줘야한다.

```xml
<!-- mybatis-config.xml -->
<plugins>
  <plugin interceptor="org.mybatis.example.EmptyResultInterceptor">
  </plugin>
</plugins>
```

### 2.  Mybatis interceptor 적용하기

아래 예시는 mybatis 공식 문서에 나와있는 예시 코드이다.

```java
// ExamplePlugin.java
@Intercepts({@Signature(
  type= Executor.class,
  method = "update",
  args = {MappedStatement.class,Object.class})})
public class ExamplePlugin implements Interceptor {
  private Properties properties = new Properties();

  @Override
  public Object intercept(Invocation invocation) throws Throwable {
    // 쿼리 실행 전에 필요한 작업
    Object returnObject = invocation.proceed();
    // 쿼리 실행 후에 필요한 작업
    return returnObject;
  }

  @Override
  public void setProperties(Properties properties) {
    this.properties = properties;
  }
}
```

interceptor 클래스를 만드려면 `interceptor` 인터페이스 클래스를 구현해야한다.
`@Intercepts`와 `@Signature`어노테이션을 붙여 Interceptor에서 가로채려고 하는 메서드의 시그니처를 `@Signature`어노테이션에 지정하면 된다.
- Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
- ParameterHandler (getParameterObject, setParameters)
- ResultSetHandler (handleResultSets, handleOutputParameters)
- StatementHandler (prepare, parameterize, batch, update, query)

#### 문제 1)`NoSuchMethodException`예외 발생

```java
@Intercepts({
    @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class})
})
public class EmptyResultInterceptor implements Interceptor {
    // ...
}

```

select 쿼리마다 결과가 없을 때 작동을 시킬거여서 
위에 처럼 `Executor`인터페이스의 `query`메서드로 SQL쿼리를 실행하고 그 결과를 반환하도록 하였다. 
`MappedStatement`와 `Object`를 인자로 받도록 작성을 하니 `NoSuchMethodException`예외가 발생하였다. 

이는 Interceptor가 가로채려고 하는 메서드의 시그니처가 실제 메서드의 시그니처와 일치하지 않을 때 발생하는 예외이다.

> `@Signature`어노테이션에서 method와 args 속성을 사용할 때는 실제 메서드의 시그니처와 완벽하게 일치해야 에러가 나지 않습니다. 즉, 메서드 이름뿐만 아니라 인자의 타입과 순서까지 정확하게 지정해야합니다. 
>  <span style="color: #808080">메서드의 시그니처는 각 버전에 맞는[mybatis docs]을 참고해주세요.</span>

Mybatis 3.5.1의 `Executor`인터페이스의 `query`메소드는 `MappedStatement`, `parameter`, `rowBounds`, `resultHandler` 총 4개의 인자를 받는다.

```java
<E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;
```


<br>

##### [사용 예시]
intercept 메서드 내에서 Invocation 객체의 getArgs 메서드를 사용하여 query 메서드의 인자들을 가져올 수 있다.
```java
Object[] args = invocation.getArgs();
MappedStatement ms = (MappedStatement) args[0];
Object parameter = args[1];
RowBounds rowBounds = (RowBounds) args[2];
ResultHandler resultHandler = (ResultHandler) args[3];
```
 - `MappedStatement`
	 - SQL 쿼리에 대한 정보를 담고 있는 MyBatis의 기본 객체이다. 
	 - SQL 문장, 실행할 SQL 명령의 유형(SELECT, INSERT, UPDATE, DELETE), 입력 파라미터의 클래스, 결과 매핑할 클래스 등의 정보를 포함하고 있다.
	 - 예를 들어, `SELECT * FROM users WHERE id = #{id}` SQL 쿼리에 대한 `MappedStatement`는 'users'테이블에서 'id'컬럼을 기준으로 데이터를 가져오는 SELECT명령을 나타낸다.
- `Object parameter` 
	- SQL 쿼리에 바인딩할 파라미터 값
	- 주로 DAO의 메소드에 넘겨주는 파라미터 객체를 의미합니다.
	- `UserMapper.getUserById(int id)`메소드를 호출할 때, id 파라미터가 parameter가 된다.
- `RowBounds` 
	- 결과로 반환할 행의 범위를 지정하는 데 사용된다.
	- `offset`과 `limit`을 설정하여 특정 범위의 결과만 가져올 수 있다.
	- 예를 들어, `RowBounds(10,5)`는 11번째 행부터 시작하여 총 5개의 행을 가져오라는 의미이다.
- `ResultHandler` 
	- 쿼리 결과를 처리하는 사용자 정의 핸들러이다.
	- 결과 세트의 각 행을 특정 객체로 변환하거나, 특정 조건을 만족하는 행만 선택하는 등의 작업을 할 수 있다.

<br>

#### 문제 2)`NullPointerException` 예외 발생

  
Interceptor Class 생성 시 오버라이드 되는 메소드에서 주의할 것은 아래 와 같이 `Proceed` 와  `Object.wrap`을 꼭 선언해주어야 한다. 
Null을 리턴하면 바로 `NullPointException`으로 예외를 발생시킨다.

```java
	public class EmptyResultInterceptor implements Interceptor { 
	@Override 
	public Object intercept(Invocation invocation) 
											throws Throwable { 
		// 필요한 로직 작성 후 
		// 아래와 같이 invocation.proceed()로 다음 동작으로 진행시킨다. 
		return invocation.proceed(); 
	} 
	@Override 
	public Object plugin(Object target) { 
		// 아래와 같이 plugin에 등록한다. 
		return Plugin.wrap(target, this); 
	}
}
```

<br>

#### 쿼리 결과가 없을 경우 빈 리스트([]) 반환

> MyBatis는 쿼리의 결과가 없을 경우 null을 반환하는 대신 비어있는 리스트를([]) 반환한다.

`MappedStatement`객체를 통해 select문인 경우에만 처리하고 결과가 없을 경우에만 작동하도록 작성

```java
@Override
public Object intercept(Invocation invocation) throws Throwable {
    MappedStatement mappedStatement = (MappedStatement) invocation.getArgs()[0];
    Object parameter = invocation.getArgs()[1];

    // select 문인 경우에만 처리
    if (mappedStatement.getSqlCommandType().name().equals("SELECT")) {
        Object result = invocation.proceed();

        // 결과가 비어있는 리스트인 경우
        if (result instanceof List && ((List) result).isEmpty()) {
             // 결과가 없을 때 처리할 내용
        }

        return result;
    }

    return invocation.proceed();
}
```

#### 다른 예외가 발생해도 객체 초기화되게 적용하기

resultType 객체를 함수를 초기화하는 함수를 따로 만들고 
어떠한 예외(`PSQLException` 등)가 발생해도 객체를 초기화 하여 화면에 `undefined` 에러 방지

```java
@Override
    public Object intercept(Invocation invocation) throws Throwable {
        MappedStatement mappedStatement = (MappedStatement) invocation.getArgs()[0];
        Object parameter = invocation.getArgs()[1];
        Object result;
        LocalDateTime now = LocalDateTime.now();
        String date = now.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
        String time = now.format(DateTimeFormatter.ofPattern("HH:mm:ss"));

        try {
            // select 문인 경우에만 처리
            if (mappedStatement.getSqlCommandType().name().equals("SELECT")) {
                try {
                    result = invocation.proceed();
                } catch (Exception e) {
                    System.out.println("에러출력 : " + e);
                    result = initializeResult(mappedStatement, date, time);
                }

                // 결과가 비어있는 리스트인 경우
                if (result instanceof List && ((List) result).isEmpty()) {
                    result = initializeResult(mappedStatement, date, time);
                }
            } else {
                result = invocation.proceed();
            }
        } catch (Exception e) {
            System.out.println("에러출력 : " + e);
            result = initializeResult(mappedStatement, date, time);
        }

        return result;
    }

// resultType 객체 초기화 함수
    private Object initializeResult(MappedStatement mappedStatement, String date, String time) throws Exception {
        Class<?> resultType = mappedStatement.getResultMaps().get(0).getType();
        System.out.println("select 조회 결과 없음");
        // 결과 타입의 인스턴스 생성
        Object instance;
        // resultType이 Map인 경우 따로 처리
        if (Map.class.isAssignableFrom(resultType)) {
            instance = new HashMap<>();
        } else {
            instance = resultType.newInstance();
        }

        // 필드 초기화
        for (Field field : resultType.getDeclaredFields()) {
        	if (field.getName().equals("serialVersionUID")) {
       	        continue;
       	    }
            field.setAccessible(true);
            // 날짜 관련 필드는 오늘 날짜로 적용
            if (field.getName().equals("msurDt")
            		|| field.getName().equals("regDtm")
            		|| field.getName().equals("applyDt")
            		|| field.getName().equals("regDt")
            		|| field.getName().equals("measDtm")) {
                field.set(instance, date + " " + time);
            } else  if (field.getName().equals("msurDate")) {
                field.set(instance, date);
            } else  if (field.getName().equals("msurTime")) {
                field.set(instance, time);
            } else if (field.getType() == String.class) {
                field.set(instance, "NULL");
            } else if (field.getType() == int.class || field.getType() == Integer.class) {
                field.set(instance, 0);
            } else if (field.getType() == double.class || field.getType() == Double.class) {
                field.set(instance, 0.0);
            } else if (field.getType() == long.class || field.getType() == Long.class) {
                field.set(instance, 0L);
            }
        }

        // 초기화된 인스턴스를 결과에 추가
        List<Object> result = new ArrayList<>();
        result.add(instance);
        System.out.println("필드 초기화 완료 : " + result);

        return result;
    }
       
```


이렇게 어떠한 예외가 발생돼도 resultType 객체를 초기화하여 화면에 undefined 에러가 발생하는 것을 방지했다.

나같은 경우는 특수한 경우이고 이러한 방법은 추천하지는 않는다.

그래도 MyBatis의 interceptor를 잘 활용하면 원하는 시점에 원하는 작업을 할 수 있으니 알고 있으면 잘 써먹을 수 있을 것 같다.


<br/><br/><br/><br/><br/>

참고
- [단일행 함수, 다중행 함수](https://sewonzzang.tistory.com/49) 
- [mybatis docs](https://mybatis.org/mybatis-3/configuration.html#plugins)
- [mybatis-3 docs](https://mybatis.org/mybatis-3/apidocs/org/apache/ibatis/executor/Executor.html)
- [Mybatis interceptor 활용하기](https://kindloveit.tistory.com/88)
- [뤼튼](https://www.wrtn.ai/)

<br/><br/>

피드백은 언제든 감사하게 받겠습니다.




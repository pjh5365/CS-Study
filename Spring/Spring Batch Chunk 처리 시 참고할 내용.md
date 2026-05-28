## 1. Chunk Size란?

Spring Batch에서 `chunk(size)`는 Reader가 한 번에 읽는 개수가 아니라, **몇 개의 item을 모아서 처리하고 커밋할지 결정하는 단위**이다.

```java
.<Input, Output>chunk(100, transactionManager)
````

의 의미는 다음과 같다.

```text
Reader가 item을 1건씩 반환
→ 100건이 모이면
→ Processor 처리
→ Writer에 100건 전달
→ Commit
```

즉 Reader는 기본적으로 한 번 호출될 때 item 1건을 반환한다.

---

## 2. Reader의 Paging Size와 Chunk Size 차이

`pageSize`는 DB나 외부 API에서 **한 번에 가져오는 데이터 개수**이고,
`chunkSize`는 Spring Batch가 **Writer와 Commit을 수행하는 단위**이다.

```text
pageSize = 조회 단위
chunkSize = 처리 / 저장 / 커밋 단위
```

예를 들어:

```text
chunkSize = 100
pageSize = 10
```

이면 DB 또는 API를 10건씩 10번 조회해서 총 100건이 모였을 때 Writer가 호출된다.

```text
10건 조회
10건 조회
...
총 100건 모임
→ Writer 1번 호출
→ Commit 1번
```

반대로:

```text
chunkSize = 10
pageSize = 100
```

이면 100건을 한 번에 조회해 Reader 내부 버퍼에 들고 있고, Writer는 10건씩 10번 호출된다.

---

## 3. pageSize를 1로 설정해도 되는가?

동작은 가능하지만 성능상 좋지 않다.

```text
chunkSize = 100
pageSize = 1
```

이면 100건을 처리하기 위해 DB/API 조회가 100번 발생한다.

따라서 네트워크 왕복, SQL 실행 비용, API 호출 비용이 크게 증가한다.

보통은 관리와 성능 예측을 쉽게 하기 위해 아래처럼 맞추는 경우가 많다.

```java
chunkSize = 100
pageSize = 100
```

---

## 4. Chunk Size는 버퍼 크기처럼 볼 수 있는가?

어느 정도 맞다.

다만 정확히는 다음처럼 구분하는 게 좋다.

```text
Reader 내부 버퍼 = pageSize 단위로 조회한 데이터
Spring Batch Chunk 버퍼 = chunkSize 단위로 모은 처리 대상
```

Reader는 데이터를 가져와 내부 버퍼에 저장하고, `read()`가 호출될 때마다 하나씩 반환한다.

Spring Batch는 Reader가 반환한 item을 chunkSize만큼 모은 뒤 Writer에 전달한다.

---

## 5. Tasklet의 for > while 구조를 Chunk로 바꾸는 방법

기존 Tasklet 구조는 보통 다음과 같다.

```text
for 대상 목록
    while 페이지 조회
        API 호출
        데이터 가공
        DB 저장
```

Chunk로 바꿀 때는 반복문의 제어권을 Spring Batch에 넘긴다.

```text
for 대상 목록
while 페이지 조회
→ Reader 내부 상태로 관리

데이터 가공
→ Processor

DB 저장
→ Writer
```

즉, Reader는 다음 상태를 필드로 들고 있어야 한다.

```text
현재 대상 index
현재 page 또는 currentId
현재 API 응답 데이터 buffer
```

---

## 6. Reader에서 같은 API가 반복 호출되는 원인

Reader의 `read()`는 chunkSize만큼 계속 호출된다.

따라서 `read()`가 호출될 때마다 API를 바로 호출하면 같은 API가 여러 번 호출될 수 있다.

잘못된 구조:

```java
@Override
public Item read() {
    List<Item> items = apiClient.call();
    return items.get(0);
}
```

이 구조는 `read()`가 호출될 때마다 API를 호출하므로 같은 데이터가 반복된다.

올바른 구조:

```java
@Override
public Item read() {

    if (iterator.hasNext()) {
        return iterator.next();
    }

    List<Item> items = apiClient.call();

    if (items.isEmpty()) {
        return null;
    }

    iterator = items.iterator();

    return iterator.next();
}
```

핵심은 API 결과를 Iterator나 Queue에 저장해두고, 남은 데이터가 있을 때는 API를 다시 호출하지 않는 것이다.

---

## 7. Reader에서 null 반환의 의미

Reader에서 `null`을 반환하면 Spring Batch는 전체 데이터가 끝났다고 판단하고 Step을 종료한다.

따라서 `null`은 다음 경우에만 반환해야 한다.

```text
모든 대상 처리 완료
모든 페이지 조회 완료
더 이상 처리할 데이터 없음
```

반대로 아래 상황에서는 null을 반환하면 안 된다.

```text
현재 페이지가 끝남
현재 학교/대상이 끝남
현재 chunk가 끝남
```

이 경우에는 다음 페이지나 다음 대상으로 넘어가야 한다.

---

## 8. @StepScope와 Reader 상태 관리

`@StepScope` 자체가 문제는 아니다.

다만 Reader 상태가 매번 초기화되거나, Reader를 직접 `new`로 생성하면 같은 대상이 반복될 수 있다.

안전한 방식은 다음과 같다.

```java
@Component
@StepScope
public class ApiPagingItemReader implements ItemStreamReader<Item> {
    private int targetIndex;
    private Long currentId;
    private Iterator<Item> iterator = Collections.emptyIterator();

    @Override
    public void open(ExecutionContext executionContext) {
        this.targetIndex = executionContext.getInt("targetIndex", 0);
        this.currentId = executionContext.containsKey("currentId")
                ? executionContext.getLong("currentId")
                : 0L;
    }

    @Override
    public void update(ExecutionContext executionContext) {
        executionContext.putInt("targetIndex", targetIndex);
        executionContext.putLong("currentId", currentId);
    }
}
```

재시작이 필요 없다면 단순 필드 상태만으로도 가능하지만, 재시작 가능성을 고려하면 `ItemStreamReader`를 사용하는 것이 좋다.

---

## 9. JobParameter와 ExecutionContext 차이

JobParameter는 Job 실행 시 외부에서 전달되는 읽기 전용 값이다.

실행 중 값을 추가하거나 변경하는 용도로는 적합하지 않다.

실행 중 값을 저장하고 다른 Step에서 사용하려면 ExecutionContext를 사용해야 한다.

```text
JobParameter
= 실행 시작 시 전달되는 읽기 전용 파라미터

JobExecutionContext
= Job 전체에서 공유 가능한 실행 상태 저장소

StepExecutionContext
= 현재 Step 내부에서 사용하는 실행 상태 저장소
```

Job 전체에서 공유하려면:

```java
stepExecution.getJobExecution()
    .getExecutionContext()
    .put("key", value);
```

꺼낼 때는:

```java
@Value("#{jobExecutionContext['key']}")
private String value;
```

---

## 10. Writer에서 Insert 처리

JPA의 `saveAll()`이나 MyBatis에서 for문으로 insert를 호출하면 기본적으로는 한 건씩 insert가 수행된다.

대량 저장 성능을 높이려면 다음 방식 중 하나를 사용한다.

### MyBatis Multi Insert

```xml
<insert id="insertUsers">
    INSERT INTO users (
        id,
        name
    )
    VALUES
    <foreach collection="users" item="user" separator=",">
        (
            #{user.id},
            #{user.name}
        )
    </foreach>
</insert>
```

Oracle에서는 보통 `INSERT ALL` 방식을 사용한다.

```xml
<insert id="insertUsers">
    INSERT ALL
    <foreach collection="users" item="user">
        INTO users (id, name)
        VALUES (#{user.id}, #{user.name})
    </foreach>
    SELECT 1 FROM DUAL
</insert>
```

### JdbcBatchItemWriter

```java
@Bean
public JdbcBatchItemWriter<User> userWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<User>()
            .dataSource(dataSource)
            .sql("""
                INSERT INTO users (
                    id,
                    name
                )
                VALUES (
                    :id,
                    :name
                )
            """)
            .beanMapped()
            .build();
}
```

---

## 11. 최종 정리

```text
chunkSize
= Writer와 Commit 단위

pageSize
= Reader가 DB/API에서 가져오는 단위

Reader
= 데이터를 1건씩 반환

Processor
= 1건씩 가공

Writer
= chunk 단위로 저장

ExecutionContext
= 실행 중 상태 저장

JobParameter
= 실행 시작 시 전달되는 읽기 전용 값
```

Tasklet의 복잡한 반복문을 Chunk로 바꿀 때 가장 중요한 것은 다음이다.

```text
반복문을 없애는 것이 아니라,
반복 상태를 Reader가 관리하게 만드는 것
```

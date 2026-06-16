## 1. Chunk 기본 동작

Spring Batch의 Chunk 구조는 데이터를 일정 단위로 모아서 처리한 뒤 Writer를 실행하는 방식이다.

예를 들어 다음과 같이 설정하면,

```java
.chunk(1000, transactionManager)
```

Spring Batch는 내부적으로 다음 흐름으로 동작한다.

```text
read() 호출 → 1건 반환
read() 호출 → 1건 반환
read() 호출 → 1건 반환
...
1000건이 모이면
processor 처리 결과를 모아
writer() 1회 실행
```

즉, Chunk 구조는 단순히 “Reader가 1000건을 한 번에 조회한다”는 의미가 아니라, **Reader가 반환한 데이터를 최대 1000건까지 모은 뒤 Writer를 실행한다**는 의미에 가깝다.

---

## 2. Reader 동작 방식

Reader는 데이터를 한 건씩 Spring Batch에 전달하는 역할을 한다.

```java
@Override
public Item read() throws Exception {
    return item;
}
```

Spring Batch는 Chunk Size만큼 데이터를 모으기 위해 `read()`를 반복 호출한다.

```text
read() 1회 호출 = item 1건 반환
```

Reader가 더 이상 반환할 데이터가 없으면 `null`을 반환한다.

```java
return null;
```

Reader가 `null`을 반환하면 Spring Batch는 더 이상 읽을 데이터가 없다고 판단한다.  
단, 이미 읽어 둔 데이터가 있다면 해당 데이터는 버려지지 않고 Processor와 Writer로 전달된다.

예를 들어 Chunk Size가 1000이고 실제 데이터가 700건이라면 다음처럼 동작한다.

```text
read() 700회 수행
read()에서 null 반환
processor 700건 처리
writer 700건 처리
step 종료
```

---

## 3. Processor 동작 방식

Processor는 Reader가 반환한 데이터를 한 건씩 받아 처리한다.

```java
@Override
public Output process(Input item) throws Exception {
    return output;
}
```

Processor에서는 보통 다음과 같은 작업을 수행한다.

```text
데이터 검증
데이터 변환
계산 로직 수행
저장용 객체로 매핑
```

Processor가 `null`을 반환하면 해당 item은 Writer로 전달되지 않는다.

```java
@Override
public Output process(Input item) {
    if (조건) {
        return null;
    }

    return output;
}
```

즉, Processor의 `null`은 “이 데이터는 저장 대상에서 제외한다”는 의미로 사용할 수 있다.

---

## 4. Writer 동작 방식

Writer는 Processor를 통과한 데이터가 Chunk 단위로 모였을 때 실행된다.

```java
@Override
public void write(Chunk<? extends Item> chunk) throws Exception {
    List<? extends Item> items = chunk.getItems();
    save(items);
}
```

Writer는 한 건씩 호출되는 것이 아니라, Chunk 단위로 한 번 호출된다.

예를 들어 Chunk Size가 1000이면 다음처럼 동작한다.

```text
Reader 1000건 반환
Processor 1000건 처리
Writer 1회 실행
```

따라서 DB 저장, 외부 완료 요청, 후처리 로직 등은 보통 Writer에서 일괄 처리한다.

---

## 5. Chunk Size의 의미

Chunk Size는 “Reader가 한 번에 조회하는 개수”가 아니라, **Writer가 실행되는 기준 건수**이다.

```java
.chunk(1000, transactionManager)
```

위 설정의 의미는 다음과 같다.

```text
최대 1000건 단위로 데이터를 모은다.
1000건이 모이면 Writer를 실행한다.
Writer 실행 후 트랜잭션을 커밋한다.
```

데이터가 1000건보다 적어도 Reader가 `null`을 반환하면, 그동안 모인 데이터만큼 Writer가 실행된다.

```text
총 데이터 500건
→ read() 500회
→ read() null 반환
→ writer 500건 처리
```

---

## 6. Iterator를 사용하는 이유

외부 API나 DB 조회 결과가 List 형태로 반환되는 경우가 많다.

```java
List<Item> resultList = getData();
```

하지만 Spring Batch Reader는 한 번에 한 건씩 반환해야 한다.

그래서 List를 Iterator로 변환한 뒤, `read()` 호출마다 하나씩 꺼내 반환한다.

```java
private Iterator<Item> currentIterator = Collections.emptyIterator();

@Override
public Item read() throws Exception {

    if (currentIterator.hasNext()) {
        return currentIterator.next();
    }

    List<Item> resultList = getData();

    if (resultList == null || resultList.isEmpty()) {
        return null;
    }

    currentIterator = resultList.iterator();

    return currentIterator.next();
}
```

즉, Iterator는 “한 번 조회한 List 데이터를 Spring Batch에 한 건씩 넘겨주기 위한 장치”라고 볼 수 있다.

---

## 7. `return read()`와 직접 반환의 차이

다음과 같은 방식도 동작은 가능하다.

```java
currentIterator = resultList.iterator();
return read();
```

하지만 이 방식은 `read()` 메서드 안에서 다시 `read()`를 호출하는 재귀 구조다.

더 명확한 방식은 Iterator에서 item을 직접 꺼내 반환하는 것이다.

```java
currentIterator = resultList.iterator();

Item item = currentIterator.next();

return item;
```

두 방식의 차이는 다음과 같다.

```text
return read()
- read() 메서드에 다시 진입한다.
- 흐름 추적이 어려워질 수 있다.
- 조건이 많아지면 디버깅이 복잡해질 수 있다.

직접 반환
- 현재 흐름에서 바로 item을 반환한다.
- 상태 변경 위치가 명확하다.
- 디버깅이 쉽다.
```

따라서 Reader에서는 `return read()`보다 직접 item을 꺼내 반환하는 방식이 더 명확하다.

---

## 정리

Spring Batch Chunk 구조의 핵심은 다음과 같다.

```text
Reader는 데이터를 한 건씩 반환한다.
Processor는 데이터를 한 건씩 처리한다.
Writer는 Chunk 단위로 실행된다.
Chunk Size는 Writer 실행 기준 건수다.
Iterator는 조회된 List를 한 건씩 반환하기 위해 사용한다.
return read()보다 item을 직접 반환하는 방식이 더 명확하다.
```
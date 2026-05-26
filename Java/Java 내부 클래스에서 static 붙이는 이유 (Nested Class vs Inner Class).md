
자바에서 내부 클래스를 작성하다 보면 이런 형태를 자주 보게 된다.

```java
class Outer {

    private class Inner {

    }
}
```

근데 어떤 경우엔 `static`이 붙고:

```java
class Outer {

    private static class Inner {

    }
}
```

어떤 경우엔 안 붙는다.

그리고 `main()` 메서드에서 non-static 내부 클래스를 생성하려 하면 에러가 나는 경우도 있다.

이번 글에서는:

- static 내부 클래스와 non-static 내부 클래스 차이
    
- 왜 main()에서 에러가 나는지
    
- 내부적으로 어떻게 동작하는지
    

까지 정리해본다.

---

# 1. non-static 내부 클래스 (Inner Class)

가장 기본적인 내부 클래스 형태.

```java
class Outer {

    private int value = 10;

    class Inner {
        void print() {
            System.out.println(value);
        }
    }
}
```

## 특징

### Outer 객체에 종속된다

`Inner`는 독립적으로 존재할 수 없다.

반드시 어떤 `Outer 객체`에 소속되어야 한다.

즉 생성 방식이:

```java
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
```

이렇게 된다.

---

## 왜 Outer 객체가 필요할까?

`Inner`는 바깥 클래스의 인스턴스 변수에 접근 가능하다.

```java
System.out.println(value);
```

이게 가능한 이유는 내부적으로 `Inner`가 숨겨진 형태로 Outer 객체 참조를 들고 있기 때문이다.

개념적으로 보면 대충 이런 느낌이다.

```java
class Inner {
    Outer this$0;
}
```

즉:

> "나는 어느 Outer 객체에 속해 있는지 알고 있어야 함"

이라는 구조다.

---

# 2. static 내부 클래스 (Static Nested Class)

이번엔 static을 붙여보자.

```java
class Outer {

    static class Inner {

    }
}
```

이 경우는 완전히 다르다.

## 특징

### Outer 객체와 연결되지 않는다

생성도 그냥 가능하다.

```java
Outer.Inner inner = new Outer.Inner();
```

Outer 객체가 필요 없다.

---

## Outer 인스턴스 변수 접근 불가

다음은 에러다.

```java
class Outer {

    int value = 10;

    static class Inner {
        void print() {
            System.out.println(value); // 에러
        }
    }
}
```

왜냐하면 static 내부 클래스는 Outer 객체 참조를 가지고 있지 않기 때문이다.

접근 가능한 건 static 멤버뿐이다.

```java
static int value = 10;
```

---

# 3. main()에서 non-static 내부 클래스가 에러 나는 이유

다음 코드를 보자.

```java
class Outer {

    class Inner {

    }

    public static void main(String[] args) {
        Inner inner = new Inner(); // 에러
    }
}
```

처음 보면 이상하다.

> "main()이 Outer 안에 있는데 왜 Inner 생성이 안 되지?"

---

# 핵심 이유

## main()은 static 메서드다

즉 JVM은 프로그램 시작 시 객체 없이 이렇게 호출한다.

```java
Outer.main(args);
```

중요한 건:

> 아직 `new Outer()`가 일어나지 않았을 수도 있다는 점.

---

## 클래스와 객체는 다르다

많이 헷갈리는 부분인데:

### 클래스

```text
설계도
```

### 객체

```text
실제 생성된 인스턴스
```

`main()`은:

```text
Outer 클래스에 속한 static 메서드
```

일 뿐이지,

```text
Outer 객체 안에서 실행되는 메서드
```

가 아니다.

---

# 그래서 왜 에러가 날까?

non-static 내부 클래스는 반드시 Outer 객체에 종속된다.

즉 이런 형태가 필요하다.

```java
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
```

하지만 main()은 static 영역이라:

- Outer 객체가 자동으로 존재하지 않음
    
- this 사용 불가능
    
- 인스턴스 멤버 접근 불가능
    

따라서:

```java
new Inner()
```

만으로는 생성할 수 없는 것이다.

---

# 4. static 내부 클래스는 왜 main()에서 생성 가능할까?

```java
class Outer {

    static class Inner {

    }

    public static void main(String[] args) {
        Inner inner = new Inner();
    }
}
```

이건 가능하다.

왜냐하면 static 내부 클래스는 Outer 객체와 연결되지 않은 독립 클래스이기 때문이다.

즉 사실상:

```java
Outer.Inner
```

라는 이름만 가진 일반 클래스에 가까운 구조다.

---

# 5. 실무에서는 보통 어떻게 사용할까?

기준은 하나다.

## Outer 인스턴스 상태를 사용하는가?

### 사용한다

→ non-static inner class

```java
class Game {

    private int score;

    class Player {
        void addScore() {
            score++;
        }
    }
}
```

---

### 사용하지 않는다

→ static nested class 추천

```java
class User {

    static class Builder {

    }
}
```

Builder는 User 객체 상태와 직접 연결될 필요가 없기 때문.

---

# 6. 실무에서 static을 많이 권장하는 이유

non-static 내부 클래스는 숨겨진 Outer 참조를 가진다.

그래서 메모리 누수 원인이 될 수 있다.

특히 안드로이드에서 유명한 문제다.

```java
class MainActivity {

    private class MyHandler extends Handler {

    }
}
```

이 경우 Handler가 Activity 참조를 계속 잡고 있어 GC가 안 될 수 있다.

그래서 보통:

```java
private static class MyHandler
```

형태를 권장한다.

---

# 정리

|구분|non-static inner class|static nested class|
|---|---|---|
|Outer 객체 참조|있음|없음|
|Outer 인스턴스 멤버 접근|가능|불가능|
|생성 방식|outer.new Inner()|new Outer.Inner()|
|static 영역에서 생성|불가능|가능|
|메모리 부담|더 큼|더 적음|

---

# 한 줄 요약

## non-static inner class

> "Outer 객체에 소속된 클래스"

## static nested class

> "Outer 안에 정의만 된 독립 클래스"

그리고 `main()`은:

> Outer 객체 내부가 아니라 static 영역에서 실행되기 때문에  
> non-static 내부 클래스를 바로 생성할 수 없는 것이다.
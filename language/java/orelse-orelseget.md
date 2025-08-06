# orElse, orElseGet 차이

\
Java 8에서 도입된 Optional 클래스는 null 값을 가질 수 있는 객체를 감싸는 래퍼 클래스다. 이는 null을 직접 다루는 것보다 안전하고 명확한 코드를 작성할 수 있도록 도와준다.\
\


#### Optional의 주요 메소드들은 다음과 같다 <a href="#optional" id="optional"></a>

* **of(T value)** : non-null 값으로 Optional 인스턴스를 생성한다. 만약 null 값이 전달되면 NullPointerException이 발생한다.\
  \

* **ofNullable(T value)** : null 값이 허용되는 Optional 인스턴스를 생성한다. 만약 null 값이 전달되면 빈 Optional 객체가 반환된다.\
  \

* **get()** : Optional 객체가 감싸고 있는 값을 반환한다. 만약 해당 객체가 비어 있다면 NoSuchElementException이 발생한다.\
  \

* **isPresent()** : Optional 객체에 값이 존재하는지 여부를 확인하는데 사용된다.\
  \

* **ifPresent(Consumer\<? super T> consumer)** : 값이 존재할 경우 주어진 함수를 실행하고, 그렇지 않으면 아무런 작업도 수행하지 않는다.\
  \

* **orElse(T other)** : 존재하는 값을 반환하거나, 없을 경우에는 기본값을 반환한다.\
  \


**예시**

```java
String name = "John Doe";
// of 메소드 사용
Optional<String> opt = Optional.of(name);
System.out.println(opt.get()); // John Doe

String unknownName = null;
// ofNullable 메소드 사용
Optional<String> optUnknown = Optional.ofNullable(unknownName);
System.out.println(optUnknown.orElse("Default Name")); // Default Name
```

\
\
\


### orElse 와 orElseGet의 차이? <a href="#orelse-orelseget" id="orelse-orelseget"></a>

Java의 Optional 클래스에는 orElse 와 orElseGet 메서드가 존재한다.\
둘 다 Optional 을 통해 가져온 값이 null 일 때는 해당 값을 반환하라는 메소드이다. 그럼 무슨 차이가 있을까?

```java
    /**
     * If a value is present, returns the value, otherwise returns
     * {@code other}.
     *
     * @param other the value to be returned, if no value is present.
     *        May be {@code null}.
     * @return the value, if present, otherwise {@code other}
     */
    public T orElse(T other) {
        return value != null ? value : other;
    }

    /**
     * If a value is present, returns the value, otherwise returns the result
     * produced by the supplying function.
     *
     * @param supplier the supplying function that produces a value to be returned
     * @return the value, if present, otherwise the result produced by the
     *         supplying function
     * @throws NullPointerException if no value is present and the supplying
     *         function is {@code null}
     */
    public T orElseGet(Supplier<? extends T> supplier) {
        return value != null ? value : supplier.get();
    }
```

* orElse() 의 경우에는 T 타입의 other 을 그대로 반환해주는 역할을 하고, orElseGet() 의 경우에는 Supplier 의 인터페이스를 통해 그 인터페이스의 결과 를 반환한다고 볼 수 있다.\
  \


**이게 무슨차이일까?**

* orElse(T other) : 이 메서드는 Optional 객체가 값으로 null을 가지고 있든 아니든 항상 인자로 전달된 메서드/연산을 수행한다.
* orElseGet(Supplier\<? extends T> other) : 이 메서드는 Optional 객체가 값으로 null을 가지고 있는 경우에만 인자로 전달된 Supplier 함수를 실행한다.\
  \
  \
  \


#### 해당 내용을 확인하기 위한 테스트코드 예시 <a href="#undefined" id="undefined"></a>

```java
@Test
@DisplayName("notNull테스트")
void name() {
    String name = "Jack";
    String elseName = Optional.ofNullable(name).orElse(anyName());
    System.out.println(elseName);

    String elseGetName = Optional.ofNullable(name).orElseGet(this::anyName);
    System.out.println(elseGetName);
}

private String anyName() {
    System.out.println("function called");
    return "anyName";
}
```

위 테스트 코드의 결과값은 다음과 같다.

> function called\
> Jack\
> Jack

* orElse일때는 null값이 아닌데도 불구하고 anyName함수의 'function called'가 콘솔에 찍힌 것을 볼 수 있다.

null일 때만 값을 return한다는 것은 같다. 둘다 Jack을 리턴 한 것이 보일것이다.\
하지만, 둘의 차이점은 orElse함수는 null값이 아닌데도 불구하고 무조건 전달된 메서드를 실행하고, orElseGet은 null값이 아닐 경우 함수를 실행 조차도 하지 않는것이다.\
\


```java
    public T orElseGet(Supplier<? extends T> supplier) {
        return value != null ? value : supplier.get();
    }
```

메소드를 다시 보면, other 를 바로 실행하는 것이 아니다. method 를 전달하고, value 가 null 이면 other.get() 즉, 메소드를 실행하는 것이다.

\
\


**따라서 연산 비용이 큰 작업들은 가능하면 orElseGet() 내부의 Supplier 함수로 제공하는 것이 좋다. 그렇게 하면 해당 연산은 실제로 필요한 경우에만 수행되게 될 것이기 때문이다.**

\

# Checked, Unchecked Exception



**Java에서는 두 가지 주요한 예외 유형을 구분한다. Checked Exception과 Unchecked Exception이다.**\
**이 둘의 차이점은 컴파일 시간에 검사가 이루어지느냐, 아니면 실행 시간에 발생하느냐에 따라 다르다.**

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

## Checked Exception

> Checked Exception은 컴파일러가 예외를 처리하도록 강제하는 예외 유형이다. 이는 코드 작성 중에 발생할 수 있는 문제들을 나타내며, 주로 외부 시스템과의 상호작용에서 발생하는 문제들을 포함한다.
>
> 예를 들어, 파일 I/O 작업, 데이터베이스 연결 등에서 종종 Checked Exception을 볼 수 있다. IOException과 SQLException은 대표적인 Checked Exception이다.
>
> Checked Exception이 발생하는 메소드를 사용할 때는 반드시 try-catch 문으로 해당 예외를 처리하거나 throws 키워드를 사용해 해당 예외를 호출자에게 전달해야 한다.

\
\


## **Unchecked Exception**

> 반면 Unchecked Exception은 컴파일러가 강제로 처리하지 않는 RuntimeException 클래스와 그 하위 클래스들의 인스턴스이다. 이들은 프로그램의 버그와 관련된 문제들을 나타내기 때문에 개발자가 미리 확인하고 수정할 수 있다.
>
> NullPointerException, ArrayIndexOutOfBoundsException 등이 Unchecked Exceptions의 대표적인 사례이다.
>
> Unchecked Exceptions는 주로 프로그래밍 오류로 인해 발생하기 때문에 개발 과정에서 충분한 테스트와 코드 검증으로 방지할 수 있다. 따라서 Java 컴파일러는 이런 종류의 예외처리를 강제하지 않는다.

![](https://velog.velcdn.com/images/jds7979/post/52a9ca2d-198a-4c41-9c25-eee1c91f12ab/image.png)

* **checked Exception과 unchecked Exception의 트랜잭션 처리 다른점**

checked exception 발생시 java에서는 default로 roll-back을 하지않고, unchecked exception의 경우에는 기본적으로 roll-back이 되는것은 맞다.

하지만 transaction의 종류는 여러가지이고, spring boot에서는 roll-back의 여부를 정의 할 수 있기때문에, 어디까지나 java의 default 처리 방식이라는거지 roll-back여부가 절대적 이라는 것은 아니다.

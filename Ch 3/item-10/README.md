# Item 10 - equals는 일반 규약을 지켜 재정의하라

## Intro

equals 메서드는 재정의하기 쉬워 보이지만 곳곳에 함정이 도사리고 있어서 자칫하면 끔찍한 결과를 초래한다. 문제를 회피하는 가장 쉬운 길은 아예 재정의하지 않는 것이다. 그냥 두면 그 클래스의 인스턴스는 오직 자기 자신과만 같게 된다. 그러니 다음에서 열거한 상황 중 하나에 해당한다면 재정의하지 않는 것이 우선이다.

- 각 인스턴스가 본질적으로 고유하다. -> 동등한 다른 인스턴스가 개념적으로 없다.
- 인스턴스의 논리적 동치성(logical equality)를 검사할 일이 없다. -> 등등성을 비교할 일이 없다.
- 상위 클래스에서 재정의한 equals가 하위 클래스에도 딱 들어맞는다. -> equals를 호출하면 상위 클래스에 잘 정의된 equals가 호출된다.
- 클래스가 private이거나 package-private( = default)이고 equals 메서드를 호출할 일이 없다.

---

## 객체의 동일성(identity)과 동등성(equality)

두 객체가 동일한 것은 두 객체의 참조값이 같은 것이다. 두 객체가 동등한 것은 두 객체의 상태나 값이 같은 것이다.

Java에서 동일성은 `==` 연산자로 비교할 수 있다. Java에서 동등성은 `equals` 메서드로 비교할 수 있다. [equals](<https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Object.html#equals(java.lang.Object)>) 메서드는 [Object](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/Object.html) 클래스에 정의되어 있고 모든 클래스는 상속한 클래스가 없으면 Object 클래스를 상속하므로 기본적으로 사용은 가능하다. Object 클래스의 equals는 동일성 기반으로 boolean 타입을 반환하도록 구현되어 있다.

---

## equals 메서드 재정의 시 일반 규약

equals 메서드를 재정의할 경우 다음 5가지 일반 규약을 따라 재정의해야한다.

- 반사성(reflexivity) - null이 아닌 모든 참조 값 `x`에 대해, `x.equals(x)`는 `true`다.
- 대칭성(symmetry) - null이 아닌 모든 참조 값 `x`, `y`에 대해, `x.equals(y)`가 `true`면 `y.equals(x)`도 `true`다.
- 추이성(transivity) - null이 아닌 모든 참조 값 `x`, `y`, `z`에 대해, `x.equals(y)`가 `true`이고 `y.equals(z)`도 `true`면 `x.equals(z)`도 `true`다.
- 일관성(consistency) - null이 아닌 모든 참조 값 `x`, `y`에 대해, `x.equals(y)`를 반복해서 호출하면 항상 `true`를 반환하거나 항상 `false`를 반환한다.
- null-아님 - null이 아닌 모든 참조 값 `x`에 대해, `x.equals(null)`은 `false`다.

반사성, 대칭성, 추이성은 수학의 동치 관계에서 따온 것으로 보이고 일관성, null-아님은 프로그래밍의 특성을 반영한 것으로 보인다.

---

## Outro

equals의 일반 규약을 모두 잘 지키고 정상 동작하는지 테스트하는 과정은 번거로우면서 까다롭다. 동등성 비교가 꼭 필요한 것이 아니면 equals 메서드를 재정의하지 말고 재정의할 경우 IDE(or Lombok, AutoValue)의 도움을 받고 비즈니스 특성상 동등성의 의미가 다를 경우 위 규약에 맞춰 개발하자.

---

## Ref

- [F-Lab: 자바의 동등성과 동일성 이해하기](https://f-lab.kr/insight/understanding-equality-and-identity-in-java?gad_source=1&gad_campaignid=22368870602&gbraid=0AAAAACGgUFfyIXNGFoznObrlZ2RyGy8HJ&gclid=CjwKCAjw89jGBhB0EiwA2o1On_qT6PqjHczAnFpFeKhuxVlJo6-OUbm4LC4K1rdO9zAtrp9bHMjZ2RoCYYgQAvD_BwE) -> 동일성을 메모리 위치로 설명했는데 참조값이 좀 더 정확한거 같아서 본문에는 참조값으로 썼다.
- [wikipedia - 동치 관계](https://ko.wikipedia.org/wiki/%EB%8F%99%EC%B9%98_%EA%B4%80%EA%B3%84)

---

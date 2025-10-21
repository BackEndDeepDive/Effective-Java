# Item 15 - 클래스와 멤버의 접근 권한을 최소화하라

## Intro

어설프게 설계된 컴포넌트와 잘 설계된 컴포넌트의 가장 큰 차이는 바로 클래스 내부 데이터와 내부 구현 정보를 외부 컴포넌트로부터 얼마나 잘 숨겼는냐다. 잘 설계된 컴포넌트는 모든 내부 구현을 완벽히 숨겨, 구현과 API를 깔끔히 분리한다. 오직 API를 통해서만 다른 컴포넌트와 소통하며 서로의 내부 동작 방식에는 전혀 개의치 않는다. 정보 은닉, 혹은 캡슐화라고 하는 이 개념은 소프트웨어 설계의 근간이 되는 원리다.

---

## 접근 제어자(Access Modifier)

Java에는 `public`, `protected`, `default(package-private)`, `private` 4가지 접근 제어자가 있다. 이 접근 제어자를 활용하면 필요한 정보만 공개하며 불필요한 정보는 숨기는 캡슐화를 구현할 수 있다.

가장 바깥 클래스인 톱레벨 클래스(인터페이스)에는 `public`, `default` 접근 제어자를 쓸 수 있으며, `public`은 모든 곳에서, `default`는 해당 패키지 내에서만 접근할 수 있다는 의미다. 패키지 외부에서 쓸 이유가 없다면 `public`으로 선언해 클라이언트가 변경에 대응하도록 구현하지 말자.

멤버(필드, 메서드, 중첩 클래스, 중첩 인터페이스)에는 `public`, `protected`, `default`, `private` 4가지 접근 제어자를 모두 쓸 수 있으며 `public`은 모든 곳에서, `protected`는 `default`의 접근 범위를 포함하며, 이 멤버를 선언한 클래스의 하위 클래스에서도, `default`는 멤버가 소속된 패키지 안의 모든 클래스에서, `private`은 멤버를 선언한 톱레벨 클래스에서만 접근할 수 있다는 의미다.

---

## 주의사항

`public` 클래스의 필드를 `public`으로 선언할 경우 클라이언트가 변경에 대응해야하는 공개 API가 되며, 일반적으로 멀티스레드 상황에서 안전하지 않고, `final` 키워드를 통해 불변으로 만들어도 내부 변경의 허점이 발생할 수 있다.

동물 인터페이스와 개, 고양이 클래스

```java
public interface Animal {

    void bark();
}

public class Dog implements Animal {
    String name;

    public Dog(String name) {
        this.name = name;
    }

    @Override
    public void bark() {
        System.out.println(name + " barked");
    }
}

public class Cat implements Animal {
    String name;

    public Cat(String name) {
        this.name = name;
    }

    @Override
    public void bark() {
        System.out.println(name + " barked");
    }
}
```

배열 참조는 변경 불가능해도 내부 원소의 참조는 변경 가능

```java
public class Test1 {

    public static final Animal[] animals = {new Dog("dog1"), new Dog("dog2")};
}

public class Test1Main {

    public static void main(String[] args) {
        Animal[] animals = Test1.animals;
        animals[0].bark();  // dog1 barked

        // Test1.animals 배열은 final 키워드로 새로운 참조 할당 방지
//        Test1.animals = new Animal[2];

        // 배열 내부 원소는 상관 X
        animals[0] = new Cat("cat1");
        animals[0].bark();  // cat1 barked
    }
}
```

배열의 접근 제어자를 `private`으로 접근 불가능하게 해도 원소를 외부에서 주입받으면 캡슐화가 깨질 수 있음

```java
public class Test2 {

    private final Animal[] animals;

    public Test2(Animal... animals) {
        this.animals = new Animal[animals.length];
        for (int i = 0; i < animals.length; i++) {
            this.animals[i] = animals[i];
        }
    }

    public void func() {
        for (Animal animal : animals) {
            animal.bark();
        }
    }
}

public class Test2Main {

    public static void main(String[] args) {
        Dog dog1 = new Dog("dog1");
        Dog dog2 = new Dog("dog2");

        Test2 test2 = new Test2(dog1, dog2);
        test2.func();
        // dog1 barked
        // dog2 barked

        // 멤버 배열을 private으로 선언해도 외부에서 객체를 주입하는 형태면 객체 값이 변경될 수 있다.
        dog1.name = "dog3";
        test2.func();
        // dog3 barked
        // dog2 barked
    }
}
```

---

## 모듈(module)

Java 9부터는 패키지들의 묶음이라는 모듈이 도입되어 접근 제어 및 캡슐화를 더 세밀하게 제어할 수 있다고 한다. 아직은 JDK 수준을 제외하면 실제 활용이 크지는 않은 것 같아서 자세한 조사는 생략했다.

---

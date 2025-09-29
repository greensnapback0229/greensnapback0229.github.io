---
layout: post
title: Comparable vs Comparator
date: 2024-04-16 00:00
category: [TIL]
author: "greensnapback0229"
tags: [JAVA]
summary:
---

## Intro

자바의 컬렉션 자료형을 사용할때 객체를 컬렉션에 넣으면 정렬을 하기위해 "기준"이 필요합니다.
primitive 자료형과 다르게 객체의 **어느 필드를 참조하여 어떻게 정렬**할지 모르기 때문입니다.

자바에서는 객체의 정렬 기준을 직접 커스텀할 수 있도록 2개의 interface를 제공하고 내부 메서드를 직접 개발자가 구현하는 방식으로 제공하고 있습니다.
크게 2가지 방식을 제공하는데 `Comparator` / `Comparable` 2가지 interface를 활용하여 구현할 수 있습니다.

각 interface의 사용법과 차이점을 알아보겠습니다.

### 사용할 class

앞으로 상품의 코드, 가격, 이름을 가지고 설명하겠습니다.

```java
public class Main {
    public static class Item {
        private int code;       //상품 코드
        private int price;      //상품 가격
        private String name;    //상품명

        public Item(int code, int price, String name){
            this.code = code;
            this.price = price;
            this.name = name;
        }

        @Override
        public String toString() {
            return String.format("{ code=%d,\tprice=%d,\tname=%s }", this.code, this.price, this.name);
        }

    }

    public static void main(String[] args) throws Exception{


    }
}
```

### 원초적인 compare의 동작

우리가 compare함수를 구현할때 다음을 기준으로 분기처리 하여 비교합니다.

| 기준                    | 반환 |
| ----------------------- | ---- |
| 비교 대상보다 더 클때   | 음수 |
| 비교 대상과 같을때      | 0    |
| 비교 대상보다 더 작을때 | 양수 |

이게 생각보다 처음에 헷갈리는데 지금 잘 봐두고 가면 좋습니다.
비교 대상 즉, **_기준이 되는 수와 비교하여_** 큰지 작은지로 생각하면 좋습니다.

**!! 비교값 반환시 주의할 점**

비교값을 반환하기 위해서 흔히 2가지 방법을 생각하게 됩니다.
1번

```java

int num;

public compareTo(int num){
  return this.num - num;
}
```

2번

```java
public compareTo(int num){
  if(this.num < num  ) return -1;
  else if(this.num == num) return 0;
  else return 1;
}
```

1번의 경우를 떠올리고 "아 클린하다"라고 생각하셨다면 조금 더 생각해야 합니다.
위 경우에는 수의 크기가 클 경우 overflow가 발생할 가능성이 있습니다.
예를 들어, this.num이 `Integer.MAX_VALUE`(= 2,147,483,647)이고, num이 -1이라면 다음과 같은 결과가 나옵니다.

```java
this.num - num = 2,147,483,647 - (-1) = 2,147,483,648
```

2번은 당연히 추가적인 사칙연산이 적용되지 않기 때문에 안전합니다.
하지만.. 코드가 그리 맘에 들진 않습니다.

가장 좋은 방법이라고 생각되는 것은 위 내용을 머리에 잘 넣어두고 Java7부터 지원하는 클래스 자료형의 `compare()` 메서드를 사용하는것 입니다.  
왼쪽에 비교 대상의 값을 넣는다는 것만 명심하면 될거 같습니다.

```java
Integer.compare(this.num, num);
```

## Comparable

정의한 객체의 Class에 직접 implements하여 사용할 수 있는 방법입니다.  
`toCompare()` 메서드를 override하여 구현해주면 됩니다.  
이렇게 되면 구현한 클래스의 기본 정렬 기준이 되어서 정렬시 `sort()` 메서드가 가장 먼저 참고하는 정렬 기준으로 작동합니다.

Comparable의 생김새는 공식문서를 참고하자 > <a href="https://docs.oracle.com/javase/8/docs/api/java/lang/Comparable.html#method.summary"> Comparable에 대한 공식문서 </a>

Comparable을 사용하면 객체의 학교에서

> Comparable을 사용한

```java
public class Main {
    public static class Item implements Comparable<Item> {

        // field, constructor, toString() 생략

        @Override
        public int compareTo(Item other){

            return Integer.compare(this.price, other.price);
        }
    }

    public static void main(String[] args) throws Exception{
        ArrayList<Item> items = new ArrayList<Item>();

        items.add(new Item(3, 1000, "pencil"));
        items.add(new Item(1, 500, "eraser"));
        items.add(new Item(0, 1000, "bottle"));

        // items.sort(null);        // -> 방법1
        Collections.sort(items);    // -> 방법2

        for(Item i : items){
            System.out.println(i.toString());
        }
    }
}
```

> 출력 결과

```plain
{ code=1,       price=500,      name=eraser }
{ code=3,       price=1000,     name=pencil }
{ code=0,       price=1000,     name=bottle }
```

출력 결과를 보면 price를 기준으로 정렬된 것을 볼 수 있습니다.
**방법1**에서 null을 넣은 이유는 기본 정렬을 참고하라는 의미로 null을 넣어서 정렬을 진행했습니다.

## Comparator

Comparator는 메서드로 두개의 객체를 비교하는 기준을 제공합니다.

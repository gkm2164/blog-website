---
title: "LISP 엔진 만들기 - Lexer - 1"
categories:
  - Lengine
tags:
  - Scala
  - Coding
--- 

Lexer는 LISP 소스코드가 주어졌을 때, 이들의 형태소를 분석하는 작업을 합니다. 가장 간단하게, Python으로 LISP을 처리하는 코드로 손쉽게 해결할 수는 있습니다.

- [Python Lisp parser](https://norvig.com/lispy.html)

해당 사이트를 들어가보면 괄호안의 Token들에 대해서 따로 뭔가 하는 작업은 없고, 그냥 배열로 나열하는 작업을 수행합니다. 가장 간단한 아이디어에 해당하는 코드가 있는데요, 아래를 살펴봅시다.

```python
# python tokenize
def tokenize(chars: str) -> list:
    "Convert a string of characters into a list of tokens."
    return chars.replace('(', ' ( ').replace(')', ' ) ').split()
```

여기에서 보면, **괄호 여닿는 곳에 일부러 스페이스를 하나씩 넣어** space로 split을 하는 것을 볼 수 있습니다. 이런 코드를 실행하면 다음과 같은 구문의 처리가 가능해집니다.

```python 
tokenize("(println (+ 3 5))") # == ['(', 'println', '(', '+', '3', '5', ')', ')']
```

최소 이정도만 되어도 식별이 잘 된다고 말할 수는 있습니다. 이 방식의 문제점은... 아무래도 문자열 구문안의 괄호조차 저렇게 처리된다는 문제점이 있습니다.

```python
tokenize("(println (concat \"3\" \"(a token which should be identified as string)\")")
# == ['(', 'println', '(', 'concat', '"3"', '"(', ...]
```

이 문제 때문에, 주어진 토큰이 문자열일 때, 이것을 문자열로 인식할 수 있는 렉서를 만들어야 합니다.

일단, 함수 구성을 해봅시다. 문자열이 주어지면, 토큰단위로 잘린 문자열의 리스트를 되돌려주는 함수여야 합니다.

```scala
def tokenize(code: String): List[String]
```

| <span style="color: red;">Warning</span>: 이것이 완성된 형태는 아닙니다. 점차 개선할 것이지만, 초기 컨셉을 잡는 정도에서는 매우 유용합니다. |

기본적으로, 우선 python tokenize와 같이 동작하는 코드를 만들어야 합니다. 이는 심플하게, 스페이스 단위, 괄호 단위로 잘 끊는 토큰 리스트로 만들어야 합니다. 꼬리 재귀 함수로 짜볼 것인데, code를 list of character로 만든 다음, 패턴 매칭을 통해 ```buf: StringBuilder```에 축적하고, 스페이스를 만나거나 괄호를 만나면 flush하는 방식으로 작성할 예정입니다.

```scala
def tokenize(code: String): List[String] = {
  @tailrec
  def loop(acc: Vector[String], buf: StringBuilder, remains: List[Char]): List[String] = remains match {
    case Nil => acc.toList
    case ' ' :: tail if buf.isEmpty => loop(acc, buf, tail)
    case ' ' :: tail => loop(acc :+ buf.toString, new StringBuilder(), tail)
    case '(' :: tail if buf.isEmpty => loop(acc :+ "("), buf, tail)
    case '(' :: tail => loop(acc :+ buf.toString, "("), new StringBuilder(), tail)
    case ')' :: tail if buf.isEmpty => loop(acc :+ ")", buf, tail)
    case ')' :: tail => loop(acc :+ buf.toString :+ ")", new StringBuilder(), tail)
    case ch :: tail => loop(acc, buf.append(ch), tail)
  }

  loop(Nil, new StringBuilder(), code.toList)
}
```

위 코드가 꽤나 복잡해보이지만, 패턴 매칭 하는 구문에 잘 집중해보면, 현재 주어진 character와 buffer의 상태에 따라 다음 액션을 결정합니다. 스페이스를 만나거나 괄호를 만났을 때, buffer가 비어있을 경우에는 그냥 아무것도 하지 않고 넘어가고, buffer가 비어있지 않다면 해당 값들을 넣어주는 역할을 합니다.

이제 앞서 지적했던 문자열 안의 괄호를 만났을 때를 살펴봅시다. 따옴표를 만났다면, 다음 따옴표가 등장할 때까지 하나의 토큰으로 인식해야합니다.

이 로직을 추가 한 tokenize의 모양은 다음과 같습니다.

```scala
def tokenize(code: String): List[String] = {
  @tailrec
  def loop(acc: Vector[String], buf: StringBuilder, remains: List[Char]): List[String] = remains match {
    case Nil => acc.toList
    case ' ' :: tail if buf.isEmpty => loop(acc, buf, tail)
    case ' ' :: tail => loop(acc :+ buf.toString, new StringBuilder(), tail)
    case '(' :: tail if buf.isEmpty => loop(acc :+ "("), buf, tail)
    case '(' :: tail => loop(acc :+ buf.toString, "("), new StringBuilder(), tail)
    case ')' :: tail if buf.isEmpty => loop(acc :+ ")", buf, tail)
    case ')' :: tail => loop(acc :+ buf.toString :+ ")", new StringBuilder(), tail)
    case '"' :: tail => 
      val (str, remains) = takeString(new StringBuilder(), tail)
      loop(acc :+ str, new StringBuilder, tail)
    case ch :: tail => loop(acc, buf.append(ch), tail)
  }

  @tailrec
  def takeString(acc: StringBuilder, remains: List[Char]): (String, List[Char]) = remains match {
    case Nil => (acc.toString, Nil)
    case '"' :: tail => (acc.append('"').toString, tail)
    case ch :: tail => takeString(acc.append(ch), tail)
  }

  loop(Nil, new StringBuilder(), code.toList)
}
```

 중요한 것은, 따옴표가 문자 그대로의 따옴표로 쓰였을 경우에 대한 처리가 필요합니다. 말그대로 escaping을 말하는 것인데요, 이것 또한 패턴매칭으로 표현할 수 있습니다. 그리고, 문자열 인식 함수는 마지막에 문자열로 취하고 남은 토큰을 되돌려줘야 합니다. takeString에 escape란 변수를 추가하여, 이에 따라 반응하게 해봅시다. 그리고 escape character는 ```\``` 로 합시다.
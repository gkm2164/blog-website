---
title: "Scala - Trie with Dotty"
categories:
  - Scala
tags:
  - Scala
  - Functional Programming Language
  - Coding
  - Dotty
  - Scala 3
--- 

이번 포스팅은 Scala 3가 곧 출시 예정인데, 궁금하기도 해서 써본 사항을 공유해보려 합니다.

# 개요

> Scala로 Trie를 쉽게 만들어볼 수 있지 않을까?

이 포스팅은 위의 질문에서 시작되었습니다. Trie를 만드는 포스팅은 여러가지가 있습니다. Scala의 `map`, `groupBy`, `mapValues` 명령어들을 활용하면 쉽사리 구현할 수 있지 않을까 하는 호기심에서 접근하다가 *Union type*이라는 것을 생각해보게 되어 결국 이 글 까지 오게 된 것 같습니다.

# Trie?

Trie는 문자열을 효과적으로 저장할 수 있는 자료구조 중 하나이며, 검색에서 많이 쓰이는 구조입니다. 간략하게 설명하자면, 여러 문자열들이 있을 때, 이들의 Common prefix들은 하나의 노드로 관리하고, 차이가 나는 지점부터 Branch가 시작되는 재미있는 구조입니다.

한편, Trie는 어떻게 보면 Recursive Tree 구조를 갖고 있습니다. 각 노드는 어떻게 보면 Map 타입입니다. Map에 아무것도 없다면, 그것은 종단 노드로 봐야합니다.

# Scala 2.xx로 Trie 구현

코드는 매우 심플합니다.

```scala
// Would not be compiled
def toTrie(words: List[String]): Map[Char, ???] = 
  words.filterNot(_.isEmpty)
       .groupBy(_.head) // (1)
       .view
       .mapValues(ws => toTrie(ws.map(_.tail))) // (2)
```

설명을 하자면, 주어진 문자열의 첫번쨰 글자로 group을 하면, 첫머리 글자를 key로 갖는 dictionary가 나옵니다. 이 작업을 (2)에서 재귀적으로 나머지 문자열 들에 대해서도 수행하는 작업입니다.

보면, 재귀적인 Map의 구조를 취하고 있습니다. 문제는, 이 Map이 재귀적인 구조를 갖다보니, 자기 자신의 타입 또한 재귀적으로 처리가 어렵다는 점에 있습니다. 그래서, 위 코드를 컴파일 되게 하려면, ??? 에 Any를 넣어야 합니다.

```scala
// Would be compiled
def toTrie(words: List[String]): Map[Char, Any] = 
  words.groupBy(_.head)
       .view
       .mapValues(ws => toTrie(ws.map(_.tail).filterNot(_.isEmpty)))
```

사실 Any가 들어가면 만사 해결이 이루어지긴 하지만, 좋은 해결책이 아닙니다. 동적 타입이 아닌 언어에 대해서, 어떻게 하는게 좋을까 생각해보다가, 결국 아래의 문제를 해결하는 것이라는 결론이 생기더군요.

```scala
type Trie = Map[Char, Trie]
```

개념적으로는, Trie는 재귀적 자료구조로, 문자를 키로 갖고, 하위로 자기 자신의 타입을 갖는 타입이지만, 이걸 컴파일 하려고 하면, 자기 자신을 참조하는 타입이라며 거절합니다.

하지만 Scala 3는 어느정도 이걸 해결할 수 있게 되었습니다.

# Scala 3로 Trie 구현

이라고 제목은 썼지만, 사실 간단합니다.

```scala
type Trie[A] = A match {
  case Nothing => Nothing
  case A => Map[A, Trie[A]] // 사실상 아무 타입..
}
```

이는 Scala 3에 새로 추가된 문법인데, 타입 파라미터를 가지고 그 타입에 따라 어떻게 타입을 반환할지를 결정합니다. 어쩌면 컴파일러가 이제 매우 바빠진 문법인 것 같습니다. Meta programming을 지원한다더니, 이런것이 있었네요...

이제 위의 구문과 함께 아래와 같이 코드를 작성할 수 있습니다.

```scala
// Would be compiled
def toTrie(names: List[String]): Trie[Char] = // (1)
  names.filterNot(_.isEmpty)
       .groupBy(_.head)
       .view
       .mapValues(x => toTrie(x.map(_.tail)))
       .toMap
```

앞서 선언했던 `Trie[A]` 타입에 `Trie[Char]`를 넣고 수행한 결과입니다. 

다시 위 타입 선언 코드를 살펴봅시다.

```scala
type Trie[A] = A match {
  case Nothing => Nothing
  case A => Map[A, Trie[A]] // 사실상 아무 타입..
}
```
사실상 Nothing은 어떤 효과도 주진 않지만, 이렇게 함으로써 컴파일 타임에 타입이 결정되어버리는 현상을 막는 의도인 것 같습니다.

우선 위의 사항을 종합해서, 코드를 만들어보면 아래와 같습니다.

```scala
object Main {
  type Trie[A] = A match {
    case Nothing => Nothing
    case A => Map[A, Trie[A]]
  }

  def toTrie(names: List[String]): Trie[Char] =
    names.filterNot(_.isEmpty)
         .groupBy(_.head)
         .view
         .mapValues(x => toTrie(x.map(_.tail)))
         .toMap

  def main(args: Array[String]): Unit = {
    val words = List("hello", "world", "wing", "word", "holy", "herald")
    val trie = toTrie(words)
    println(trie)
  }
}
```

이상을 만든 후, 실행한다면 아래와 같은 결과를 얻습니다.

```
Map(h -> Map(e -> Map(l -> Map(l -> Map(o -> Map())), r -> Map(a -> Map(l -> Map(d -> Map())))), o -> Map(l -> Map(y -> Map()))), w -> Map(i -> Map(n -> Map(g -> Map())), o -> Map(r -> Map(l -> Map(d -> Map()), d -> Map()))))
```

Trie가 만들어진 것이 맞는지 좀 더 면밀히 보기 위해 출력을 아래와 같이 정리해봤습니다.

```
Map(h -> Map(e -> Map(l -> Map(l -> Map(o -> Map())),
                      r -> Map(a -> Map(l -> Map(d -> Map())))), //(1)
         o -> Map(l -> Map(y -> Map()))), // (2)
    w -> Map(i -> Map(n -> Map(g -> Map())), // (3)
              o -> Map(r -> Map(l -> Map(d -> Map()), // (4)
                            d -> Map())))) // (5)
```

우선, (1), (2), (3), (4), (5)를 보면, 가지가 필요한 시점에 잘 쳐진 것으로 보아, Trie가 잘 만들어진 것을 볼 수 있고,
자연스럽게 마지막 Map()이 들어감으로써, 전체적인 구조를 잘 유지하고 있는 것을 알 수 있습니다.

한편, 이렇게 Trie를 만들었으면 이제 쓸 수 있어야 하는데, insert, search등이 되면 좋겠다는 생각이 드네요. 그럼, Scala 3에서 소개하는 method extension기능을 사용해보도록 합시다.

# Scala 3 method extension

초기 발표했을 당시, 제가 봤던 기능은 이러했습니다.

```scala
// Not now...
def (self: String) hello(name: String): String =
  s"$self, $name"

println("Hello".hello("world"))
=> Hello, world
```

하지만, 최근에 여러 이유를 근거로 extension으로 바꾼 것 같습니다. 한편, 기존에 Scala 2에서는 method extension을 하려면 복잡한 implicit 선언을 통해 해왔습니다.

예를 들자면, 위를 구현한다 치면, 아래와 같았죠.

```scala
implicit class StringExt(self: String) {
  def hello(name: String): String = s"$self, $name"
}
```

한편, Scala 3에서는 그냥 extension이란 키워드를 도입했는데, 큰 차이는 못느끼겠습니다.

```scala
extension (self: String) {
  def hello(name: String): String = s"$self, $name"
}
```

여러가지 장점이 있겠습니다만, 제 기준에서는 그저 키워드만 바꿨다는 느낌? 여튼, 위의 Trie를 조금 확장해봅시다.

우선은, search가 되게 만들어볼 예정입니다.

```scala
extension (self: Trie[Char]) {
  def search(keyword: List[Char]): Boolean = keyword match {
    case Nil => true
    case h :: t => self match {
      case map: Map[Char, Trie[Char]] if map.contains(h) => map(h).search(t) // (1)
      case _ => false
    }
  }
}
```

일단, `Trie[Char]` 타입이 항상 Map으로 존재한다는 보장이 맨 위에서 일단 없었습니다. 그렇기 때문에, 현재의 노드가 map타입인지 확인해줄 필요는 있을 것 같네요. 다만, 맨 위에서 언급했듯이, Trie 타입은 재귀적인 자료구조입니다. 그래서 Sub-tree 역시 Trie 타입이기 때문에, (1)과 같이 주어진 노드에서, 여기에서 구현한 search 함수를 바로 호출할 수 있는 장점이 있습니다.

이번엔 insert를 구현해보도록 합시다.

```scala
extension (self: Trie[Char]) {
  def search(word: List[Char]): Boolean = ...

  def insert(keyword: List[Char]): Trie[Char] = keyword match {
    case Nil => self
    case h :: t => self match {
      case map: Map[Char, Trie[Char]] if map.contains(h) =>
        map.updated(h, map(h).insert(t)) // (1)
      case map: Map[Char, Trie[Char]] =>
        map.updated(h, toTrie(List(t.mkString("")))) // (2)
    }
  }
}
```

insert의 경우에도, (1)과 같이 재귀적인 처리가 가능합니다. 앞에서 설명했듯, Trie가 재귀적인 구조이므로, `map(head).insert(tail)`를 수행하면, 다음 노드가 존재하는지를 확인하고, 없다면 (2)를 실행하는 방식으로 진행할 예정이므로, 손쉽게 구현이 가능합니다.

다음으로 추가를 했으니 지워보기도 해봅시다.

```scala
extension (self: Trie[Char]) {
  def search(word: List[Char]): Boolean = ...

  def insert(keyword: List[Char]): Trie[Char] = ...

  def isEmpty: Boolean = self == Map()

  def delete(keyword: List[Char]): Trie[Char] = keyword match {
    case Nil => self
    case h :: t => self match {
      case map: Map[Char, Trie[Char]] if map.contains(h) && map(h).isEmpty => map.removed(h) // (1)
      case map: Map[Char, Trie[Char]] if map.contains(h) =>
        val deleted = map(h).delete(t)
        if (deleted.isEmpty) {
          map.removed(h) // (2-1)
        } else {
          map.updated(h, deleted) // (2-2)
        }
      case map: Map[Char, Trie[Char]] => // (3)
        println("doing nothing")
        self
    }
  }
}
```

(1)은 다음 노드가 비어있다면, 그냥 지우게 됩니다. 한편, (3)은 존재하지 않는 word이므로 그냥 아무것도 하지 않습니다.

(2-1)에서는, 지운 노드가 비어있다면 현재 노드 또한 재귀적으로 지우는 명령입니다.
(2-2)는 그렇게 지운 노드가 여전히 값을 갖고 있다면 살려둡니다.

이상의 extension을 정리해보면 아래와 같습니다.

```scala
extension (self: Trie[Char]) {
  def search(keyword: List[Char]): Boolean = keyword match {
    case Nil => true
    case h :: t => self match {
      case map: Map[Char, Trie[Char]] if map.contains(h) => map(h).search(t)
      case _ => false
    }
  }

  def insert(keyword: List[Char]): Trie[Char] = keyword match {
    case Nil => self
    case h :: t => self match {
      case map: Map[Char, Trie[Char]] if map.contains(h) =>
        map.updated(h, map(h).insert(t))
      case map: Map[Char, Trie[Char]] =>
        map.updated(h, toTrie(List(t.mkString(""))))
    }
  }

  def delete(keyword: List[Char]): Trie[Char] = keyword match {
    case Nil => self
    case h :: t => self match {
      case map: Map[Char, Trie[Char]] if map.contains(h) && map(h).isEmpty => map.removed(h) // (1)
      case map: Map[Char, Trie[Char]] if map.contains(h) =>
        val deleted = map(h).delete(t)
        if (deleted.isEmpty) {
          map.removed(h) // (2-1)
        } else {
          map.updated(h, deleted) // (2-2)
        }
      case map: Map[Char, Trie[Char]] => // (3)
        println("doing nothing")
        self
    }
  }
}
```

이제 위의 extension을 기준으로, interactive한 프로그램을 만들어보겠습니다.

사용자로부터 단어를 읽어와서,
(1) word가 trie안에 있다면 "Found word"를 출력
(2) word가 trie안에 없다면, 오류("Not found word, try next time")와 함께 Trie에 추가하는 작업

위의 간단한 기능을 만들어봅시다.

```scala
def main(args: Array[String]): Unit = {
  val words = List()
  val trie = toTrie(words)
  var nextTrie = trie
  while (true) {
    print("> ")
    StdIn.readLine() match {
      case str if nextTrie.search(str.toList) =>
        println(s"Found word: $str")
      case str => println("No word found, try next time")
        nextTrie = nextTrie.insert(str.toList)
    }
    
    System.out.flush()
  }
}
```

이제 몇가지 작업을 해본다면, 아래와 같이 동작합니다.

```scala
> hello
No word found, try next time
> hello
Found word
> herald
No word found, try next time
> herald
Found word
```

이렇게 만들었지만, trie가 보이지 않아 답답하군요.. 그래서 출력하는 구문들을 넣어봅시다.
print라는 명령을 치면 trie를 출력하도록 만들어봅시다.

```scala
def main(args: Array[String]): Unit = {
  // ...
  while (true) {
    print("> ")
    StdIn.readLine() match {
      case "print" => println(nextTrie)
      case str if nextTrie.search(str.toList) => ...
      
      case str => ...
    }
    
    System.out.flush()
  }
}
```

```
> print
Map()
> hello
No word found, try next time
> print
Map(h -> Map(e -> Map(l -> Map(l -> Map(o -> Map())))))
> hello
Found word
> herald
No word found, try next time
> print
Map(h -> Map(e -> Map(l -> Map(l -> Map(o -> Map())),
                  r -> Map(a -> Map(l -> Map(d -> Map()))))))
> herald
Found word
```

이제 지우는 명령을 추가해보도록 합시다. 지우는 명령은 어떻게 추가할까 고민해봤지만, 결국 "del xx"라고 하면 지울 수 있도록 만들어보고 싶어졌습니다.

우선 regex를 정의하고 패턴매칭에서 활용할 수 있도록 해봅시다..

```
val deleteCommand = "del (.+)".r

while (true) {
  print("> ")
  StdIn.readLine() match {
    // ...
    case deleteCommand(str) => nextTrie = nextTrie.delete(str.toList)
    // ...
  }
```

이렇게 만든 후, 다시 실행해봅시다.

```
> print
Map()
> hello
No word found, try next time
> herald
No word found, try next time
> print
Map(h -> Map(e -> Map(l -> Map(l -> Map(o -> Map())),
                  r -> Map(a -> Map(l -> Map(d -> Map()))))))
> del hello
> print
Map(h -> Map(e -> Map(r -> Map(a -> Map(l -> Map(d -> Map()))))))
```

이와 같이 search, insert, delete가 가능한 Trie를 만들어봤습니다.

소스 코드는 [여기](https://gist.github.com/gkm2164/8e5ddf670cc43abb745863586c53bf74)에서 공유하고 있습니다.

# 결론

한편, Scala 3에서 좀 더 세련되게 만드는 방법도 있을 것 같습니다만, 우선, 현재의 포스팅은 조금 그래도 Type safe하게 해봤다는 점과 extension 키워드를 써봤다는데 의의가 있다고 볼 수 있을 것 같습니다.

모쪼록, 알고리즘 문제 풀어볼 때마다 Trie만들기 참 귀찮다 생각했었는데, 이제 이렇게 쉽게 만들어 볼 수 있어서 다행인 듯 합니다.

IO모나드를 써서 코드를 조금 더 예쁘게 다듬고 싶지만... 일단은 Trie가 목적이니, 여기에서 마무리 하도록 하겠습니다.(ㅎㅎ..)
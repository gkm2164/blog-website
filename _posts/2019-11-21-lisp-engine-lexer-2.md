---
title: LISP 엔진 만들기 - Lexer - 2
categories:
- Lengine
tags:
- Scala
- Token
date: 2019-11-21 00:00 +0000
---
이제 어휘의 명칭을 붙여봅시다. 여러가지 접근 방법이 있겠지만, 여기에서는 [LISP 엔진 만들기 - Lexer - 1](https://blog.gyeongmin.co/lengine/2019-11-20/lisp-engine-lexer-1/)에서 만든 코드로 생성된 String List의 문자열들을 매칭하여 토큰 타입을 결정하는 방법을 씁니다.

# 예제

이해를 돕기 위해, 다음 코드를 한번 생각해봅시다.

```lisp
(fn sum (a b) (+ a b))
```

이상의 코드는 Lengine의 tokenizer를 거쳐 다음과 같이 문자열의 리스트로 바뀝니다.

```scala
List("(", "fn", "sum", "(", "a", "b", ")", "(", "+", "a", "b", ")", ")")
```

그리고 배열 값은 다시 Lexer를 거쳐 다음과 같은 토큰으로 식별됩니다.

```scala
List(LeftPar, LispFn, EagerSymbol("sum"),
              LeftPar, EagerSymbol("a"), EagerSymbol("b"), RightPar,
              LeftPar, EagerSymbol("+"), EagerSymbol("a"), EagerSymbol("b"), RightPar,
     RightPar)
```

자세한 타입의 정의는 [표](#token-tables)를 참조해주세요.

# 구현

이 부분은 스칼라의 pattern match를 거치면 생각보다 구현하기 쉽습니다. 문자열의 리스트가 주어졌을 때, 이들을 keyword들의 패턴매칭을 통해 해당 토큰으로 태깅하는 것입니다.
이번 코드에서는 예제 코드만 처리 가능한 렉서를 만들어보도록 하겠습니다.

<a id="lisp-token"></a>

```scala
object LispToken {
  private val SymbolRegex: Regex = """([a-zA-Z\-+/*%<>=?][a-zA-Z0-9\-+/*%<>=?]*\*?)""".r

  def apply(token: String): LispToken = token match {
    case "" => LispNop
    case "fn" => LispFn
    case "(" => LeftPar
    case ")" => RightPar
    case SymbolRegex(name) => EagerSymbol(name)
    case _ => ???
  }
}
```

이제 위의 코드가 동작하려면 아래와 같이 사용할 수 있습니다.

```scala
val tokens = Tokenizer.tokenize("(fn sum (a b) (+ a b))").map(LispToken.apply)
println(tokens)

/*
List(LeftPar, LispFn, EagerSymbol("sum"),
              LeftPar, EagerSymbol("a"), EagerSymbol("b"), RightPar,
              LeftPar, EagerSymbol("+"), EagerSymbol("a"), EagerSymbol("b"), RightPar,
     RightPar)
*/
```

앞서 제시한 [```LispToken.apply```](#lisp-token) 코드는 앞으로 뭔가 추가될 때마다 확장해야하니 참고 바랍니다.

# 비고 

아래는 lengine에서 식별하는 LISP token의 리스트입니다.

<a id="token-tables"></a>

| Token Value        | Token name                 | Token type name                 |
|:------------------:|:--------------------------:|:-------------------------------:|
| ```#C(```          | Complex Number Parenthesis | ```ComplxNPar```                |
| ```(```            | Left Parenthesis           | ```LeftPar```                   |
| ```'(```           | List start parenthesis     | ```ListStartPar```              |
| ```)```            | Right parenthesis          | ```RightPar```                  |
| ```{```            | Left brace                 | ```LeftBrace```                 |
| ```}```            | Right brace                | ```RightBrace```                |
| ```[```            | Left bracket               | ```LeftBracket```               |
| ```]```            | Right bracket              | ```RightBracket```              |
| ```nil```          | Nil list                   | ```LispNil```                   |
| ```loop```         | loop keyword               | ```LispLoop```                  |
| ```import```       | import keyword             | ```LispImport```                |
| ```let```          | let keyword                | ```LispLet```                   |
| ```def```          | def keyword                | ```LispDef```                   |
| ```fn```           | fn keyword                 | ```LispFn```                    |
| ```lambda```       | lambda keyword             | ```LispLambda```                |
| ```do```           | do keyword                 | ```LispDo```                    |
| ```for```          | for keyword                | ```LispFor```                   |
| ```ìn```           | in keyword                 | ```LispIn```                    |
| ```eager_symbol``` | Eager Symbol               | ```EagerSymbol(name: String)``` |
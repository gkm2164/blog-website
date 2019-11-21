---
title: "LISP 엔진 만들기 - Lexer - 2"
categories:
  - Lengine
tags:
  - Scala
  - Token
---

이제 어휘의 명칭을 붙여봅시다. 여러가지 접근 방법이 있겠지만, 여기에서는 [LISP 엔진 만들기 - Lexer - 1](https://blog.gyeongmin.co/lengine/2019-11-20/lisp-engine-lexer-1/)에서 만든 코드로 생성된 String List의 문자열들을 매칭하여 토큰 타입을 결정하는 방법을 쓸 것이며, Lengine에서는 다음과 같은 자료 구조를 따릅니다.

```scala
trait LispToken

case object LispNop extends LispToken

// #C(
case object CmplxNPar extends LispToken

// (
case object LeftPar extends LispToken

// nil
case object LispNil extends LispToken

// '(
case object ListStartPar extends LispToken

// )
case object RightPar extends LispToken

// {
case object LeftBrace extends LispToken

// }
case object RightBrace extends LispToken

// [
case object LeftBracket extends LispToken

// ]
case object RightBracket extends LispToken

// loop
case object LispLoop extends LispToken

// import
case object LispImport extends LispToken

// let
case object LispLet extends LispToken

// def
case object LispDef extends LispToken

// fn
case object LispFn extends LispToken

// lambda
case object LispLambda extends LispToken

// do
case object LispDo extends LispToken

// for
case object LispFor extends LispToken

// in
case object LispIn extends LispToken
```
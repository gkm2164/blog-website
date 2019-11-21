---
title: "LISP 엔진 만들기 - Lexer - 1"
categories:
  - Lengine
tags:
  - Scala
  - Token
---

이제 어휘를 붙여봅시다. 여러가지 접근 방법이 있겠지만, Lengine에서는 다음과 같은 자료 구조를 따릅니다.

```scala
sealed trait LispToken

case object LeftParenthesis extends LispToken
case object RightParenthesis extends LispToken
case class StringToken(str: String) extends LispToken

```
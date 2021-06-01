---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "Aecor Intro [번역] - Event Sourcing with Scala"
subtitle: ""
summary: ""
authors: [Youngseo Choi]
tags: [Functional Programming, Scala, Event Sourcing]
categories: [Programming]
date: 2021-05-21T23:52:45+09:00
lastmod: 2021-05-21T23:52:45+09:00
featured: false
draft: false
toc: true

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
이 글은 Vladimir Pavkin의 [Aecor - Purely functional event sourcing in Scala. Introduction](https://pavkin.ru/aecor-intro/)을 번역한 글입니다.

---

지금부터 [Aecor](http://aecor.io/)에 대한 시리즈를 시작해보려 한다. Aecor는 이벤트 소스를 사용하는 어플리케이션을 만들기 위해 Scala + 순수함수형 프로그래밍을 사용한 라이브러리이다. 나의 장대한 계획은 툴에 대한 종합적인 설명 뿐만 아니라 다음까지 포함한다.

- 일반적인 [이벤트 소싱](https://martinfowler.com/eaaDev/EventSourcing.html)에 관한 논의와 Aecor가 어떻게 이벤트 소싱을 다루는지
- 수면 아래에서 Aecor가 어떻게 작동하는지 설명
- 거기다 실제로 작동하는 애플리케이션까지!

## Introduction

Aecor는 2년 정도 전에 [Denis Mikhaylov](https://github.com/notxcain)([@notxcain](https://twitter.com/notxcain))이 전체 구조를 제작하였다. 나는 이 프로젝트를 초기부터 지켜보고 있었고, 최근에 Denis의 팀인  [Evotor](https://evotor.ru/)에 합류할 수 있는 기회를 얻었다. 이 곳에서는 이미 여러 Aecor 기반의 서비스가 실제로 운영되고 있었다.

높은 수준의 FP 프로젝트가 실제로 운영되는 것을 볼 수 있다는 것은 정말 신나는 일이었다. 나의 시선에서 그 곳은 FP로 만들어진 멋진 프로그램 더미에서 신나게 노는 곳으로 보였다. 물론 실제로는 철저하게 테스트된 솔루션과 잘 설계된 깔끔하고 결합가능한 인터페이스로 이루어져 있었다.

Aecor는 언제나 Scala를 통해 순수 함수형 프로그래밍을 사용하는 업계의 최신 기술을 적용하였다. (그다지 놀라운 사실은 아니다.) Aecor의 코드를 읽다보면 [cats](https://github.com/typelevel/cats), [cats-effect](https://github.com/typelevel/cats-effect), [fs2](http://fs2.io/) 그리고 다른 [Typelevel](https://typelevel.org/) 라이브러리들을 개성적이고 강력하게 활용했다는 것을 알 수 있을 것이다. 또한 Tagless Final 패턴에 대해서도 이야기 할 것인데 Aecor가 정말 흥미롭게 활용하고 있으니 주목하기 바란다.

이 모든 강력한 도구들은 언제나 나를 매료시켰던 **이벤트 소싱**을 제공하기 위함이다. 이 기술에 대한 많은 자료들이 있으며 때로는 현재 상황에 걸맞지 않는 힘을 제공하곤 한다. 확실한 것은 [모든 상황에 이벤트 소싱을 적용해선 안된다](https://youtu.be/LDW0QWie21s?t=1257)는 것이다. 하지만 이벤트 소싱을 잘 적용할 수 있는 사례가 있다면 Aecor가 당신을 위해 어려운 일들을 해결해 줄 것이다.

난 몇 년간 [취미로써](https://github.com/vpavkin/i-have-money), [일로써](http://evolutiongaming.com/) Scala를 이용한 이벤트 소싱과 함께했다. 비록 내 자신을 전문가라 부르기는 어려우나 Denis가 Aecor에 집어넣은 지식과 노력의 진가를 인정한다.

이제 Aecor는 실제 세상에 조금씩 드러나고 있다. 이벤트 소싱이 적용된 실제로 작동하는 애플리케이션을 운영하는 팀 안에서 나 역시 매일 많은 배움을 얻고 있다. 지금이야말로 이 시리즈를 연재하기 가장 좋은 타이밍이라고 생각한다.

## What Aecor gives you

이 시리즈는 Aecor가 개발자에게 부여하는 능력에 대해 이야기 할 것이므로 간단하게 언급하고 넘어가겠다.

이벤트 소싱에서 가장 신나는 부분은 *행동(behavior)*을 정의하는 것이다. 나는 소프트웨어를 디자인은 행동을 정의하는 것에서 시작해야 한다고 생각한다. 데이터베이스 스키마 대신 행동에 집중하는 것은 [도메인 주도 설계](https://en.wikipedia.org/wiki/Domain-driven_design)에서 역시 뿌리가 되는 부분이다. Aecor도 이 원리에 기반하고 있다.

Aecor는 다양한 방식의 이벤트 소스 행동들을 정의하기 위해 MTL 스타일 타입클래스들을 제공한다. [Part1](https://pavkin.ru/aecor-part-1)과 [Part2](https://pavkin.ru/aecor-part-2/)에서 더 자세한 내용을 설명할 것이다.

다음으로 당신은 어떤 방법으로든 정의한 행동을 *실행*하고 싶을 것이다. 이벤트 소싱의 확장성(Scalability)은 견고하면서 고립된 작은 섬을 구축하는 능력에서 나온다. 간단히 말해 임의의 엔티티에서 동시 커맨드 프로세싱(Concurrent command processing)이 일어나지 않는다는 것을 보장함을 의미한다. 이는 *단일 라이터 원칙(Single Writer Principle)*로도 알려져 있다. 그리고 분산 시스템에서는 이 원칙이 지켜진다는 *합의*가 필요하다.

만약 이런 합의가 필요한 상황에서 Scala만 사용해서 해결하는 방법은 [Akka-cluster](https://doc.akka.io/docs/akka/2.5/cluster-usage.html)이다. Akka-cluster의 [샤딩](https://doc.akka.io/docs/akka/2.5/cluster-sharding.html) 모듈은 확장성이 높은 이벤트 소싱 시스템에 완벽히 들어맞는 해법이다. Aecor는 akka-cluster 기반에서 우리가 정의한 행동을 실행할 수 있게 한다. [Part3](https://pavkin.ru/aecor-part-3/)에서 더 자세하게 알아볼 것이다. 추가적으로...

- Aecor가 어떻게 우리가 짠 순수한 함수형, 타입 코드를 그다지 순수하지도 완벽하게 타입이 적용되지 않은 Akka 액터 코드와 분리시키는지 알아본다.
- Akka의 이벤트 소싱 솔루션인 [Akka-persistence](https://doc.akka.io/docs/akka/2.5/persistence.html)와 비교해서 Aecor가 갖는 장점이 무엇인지 알아볼 것이다.
- 단일 라이터를 구현하기 위해 여러 가지 대안들을 살펴볼 것이다. 특히 현재 진행 중인 Kafka 기반 런타임 [R&D](https://github.com/notxcain/aecor/pull/49)를 자세히 알아볼 것이다. (Kafka 영역으로 원칙 합의 의무를 넘겼다.)

Part [4a](https://pavkin.ru/aecor-part-4a/)와 [4b](https://pavkin.ru/aecor-part-4b/)에서는 Aecor를 이용해 [CQRS](https://martinfowler.com/bliki/CQRS.html)를 구현하기 위한 단계적 과정을 알아볼 것이다. CQRS가 이벤트 소싱과 잘 어울린다는 것은 잘 알려진 사실이다. 따라서 이벤트 소싱이라는 도구 상자에 CQRS 드라이버가 없다는 것은 이상한 일이다.

[Part5](https://pavkin.ru/aecor-part-5/)는 Aecor와 직접적으로 관련되어있다. 이벤트 소싱된 엔티티와 시스템의 다른 파트를 조율할 수 있는 매우 강력한 툴인 [프로세스 매니저](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html) 패턴에 대해 알아볼 것이다. 이는 Aecor 기반 앱에 잘 적용될 수 있기 때문에 이를 위한 챕터를 따로 구성하였다.

이 시간을 통해 Aecor를 이용한 솔루션 구성 방법에 대한 모든 것을 알 수 있을 것이다. 따라서 이 기회가 수면 아래에서 Aecor가 무엇을 하는지 아는 좋은 기회가 될 것이다. [Part6](https://pavkin.ru/aecor-part-6)에서는 Aecor의 부품들을 하나씩 쪼개서 어떻게 작동하고 어떤 디자인 의사결정이 있었는지 알아볼 것이다.

## What we're going to build

일반적인 이벤트 소싱 예시로 입금, 출금 또는 전자상거래 등이 있다. 대신 우리는 간단한 콘서트 티켓 예약 시스템을 만들어 볼 것이다. 우리는 몇 가지 까다로운 비즈니스 규칙을 적용시켜 볼 것이다. 물론 이는 예약 시스템을 만들기 위한 가이드가 아니다. 임의로 결정한 요구사항은 실제 도메인 전문가가 보기에 이상한 점도 있을 것이다. 하지만 이 예시가 Aecor를 이용해 너무 간단하지 않은 앱을 설명한다는 이 시리즈의 목적에 잘 맞기 때문에 이 점은 양해해주기 바란다.

[github repo](https://github.com/vpavkin/ticket-booking-aecor)에서 완성된 솔루션을 확인할 수 있다. readme의 지시를 읽고 프로젝트를 실행해보고 갖고 놀면 된다. (또는 부숴봐도 된다!)

## Installing Aecor

시작하기 전에 Aecor를 설치하고 빌드하는 방법을 아래 첨부한다. 특정 모듈을 사용하는 방법은 나중에 이야기 할 것이다.

```scala
val aecorVersion = "0.18.0"
 
libraryDependencies ++= Seq(
  "io.aecor" %% "core" % aecorVersion,
  "io.aecor" %% "schedule" % aecorVersion,
  "io.aecor" %% "akka-cluster-runtime" % aecorVersion,
  "io.aecor" %% "distributed-processing" % aecorVersion,
  "io.aecor" %% "boopickle-wire-protocol" % aecorVersion,
  "io.aecor" %% "test-kit" % aecorVersion % Test
)
```

또한 [partial-unification](https://www.scala-lang.org/news/2.12.0/#partial-unification-for-type-constructor-inference) 플래그를 켜는 것을 잊지 말아야 한다.

여기까지 Aecor 소개가 끝났다. [Part1](https://pavkin.ru/aecor-part-1)에서는 예약 시스템에 들어갈 행동을 정의해볼 것이다.

---

여기까지 Aecor 인트로였습니다. 번역은 계속 하고 있으나 내용도 어렵고 알맞은 한글을 찾기가 참 어렵다고 느낍니다. 읽어주셔서 감사합니다.

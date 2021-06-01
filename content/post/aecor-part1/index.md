---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "Aecor Part1 - Scala 이벤트 소싱하기!"
subtitle: ""
summary: ""
authors: [Youngseo Choi]
tags: [Functional Programming, Scala, Event Sourcing]
categories: [Programming]
date: 2021-05-22T00:38:38+09:00
lastmod: 2021-05-22T00:38:38+09:00
featured: false
draft: true
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

이 글은 Vladimir Pavkin의 [Aecor - Purely functional event sourcing in Scala. Part 1](https://pavkin.ru/aecor-part-1/)을 번역한 글입니다.

---

이 글은 [Aecor](http://aecor.io/) 시리즈로 연재하고 있는 글이다. Aecor는 이벤트 소싱이 적용된 어플리케이션을 Scala를 통해 순수 함수적으로 구현하는 라이브러리이다. 아직 Aecor에 대해 들어본 적이 없다면 먼저 [Introduction](https://pavkin.ru/aecor-intro)을 읽고 오길 바란다.

이 글에서는 일반적인 엔티티의 행동(behavior)들에는 무엇이 있는지 알아보고 Aecor를 이용해 어떻게 이벤트 소싱이 적용된 행동을 만드는지 알아볼 것이다. 또한 실제 디자인 사례를 짚어가면서 몇 가지 질문들에 대해 답해볼 것이다. 꽤나 긴 글이니 커피 한 잔과 함께 하길 바란다.

## Part 1. Defining entity behavior

우리가 [introduction](https://pavkin.ru/aecor-intro)에서 이야기 했듯이 지금부터 티켓 예약 시스템을 만들어 볼 것이다.

티켓을 예약하는 행위는 우리의 가상 비즈니스에서 메인 도메인이 된다. 여러가지 복잡성과 고유의 일시성을 포함하여 "예약하기(Booking)"가 이벤트 소싱되기에 적절한 엔티티 후보이다.

하지만 왜 굳이 "예약하기"인가? 어떻게 이런 결정을 내리게 되는가? 이에 대해 좀 더 자세히 이야기해보겠다.

## Picking entities

이벤트 소싱은 [도메인 주도 설계](https://en.wikipedia.org/wiki/Domain-driven_design)와 매우 잘 어울려서 작동한다. 그리고 **엔티티(Entity**)라는 용어는 도메인 주도 설계에서 등장한 것이다. 잘 알려진 "[파란책](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)"에서 엔티티는 다음으로 정의된다.

- 식**별되는 정보(distinguished identity)**을 가진 것, 이를 통해 엔티티의 서로 다른 두 인스턴스를 구별할 수 있다. 설령 서로 같은 속성(attribute)들을 가지고 있더라도 구별할 수 있다.
- 비즈니스 규칙에 따른 특정 형태의 **생명 주기(lifecycle)**를 갖는다.

티켓 예약 컨텍스트 안에서 쉽게 여러 엔티티들을 정의할 수 있는 것을 생각해보면 예약하기가 가장 먼저 떠오른다.

- 클라이언트가 항상 참조할 수 있도록 유일한 식별자(identifier)를 가져야 한다. (identity)
- 예약 프로세스가 진행됨에 따라 몇 개의 구별된 단계를 거친다. (lifecycle)

따라서 우리는 *예약하기*를 우리의 첫 번째 엔티티로 선정하였다. 여기까지는 봤을 때는 이벤트 소싱과 큰 관련이 있지는 않다. 전통적인 도메인 주도 설계 엔티티는 기본적인 [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) 저장소로 구현된다. 반명 이벤트 소싱 시스템 내부의 값을 보면 엔티티들은 일관된 경계(Boundary)와 잘 어우러진다.

엔티티(또는 엔티티가 합쳐진 에그리게잇(Aggregate))에 이벤트 소싱을 적용하면 보통 세분성과(granularity)(확장성을 부여한다) 일관성(consistency) 사이에 가장 최적화된 트레이드 오프를 제공한다. 이는 대부분의 비즈니스 규칙(invariant, [참고 링크](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/microservice-ddd-cqrs-patterns/domain-model-layer-validations))이 하나의 일관된 경계 내부에서 확인될 수 있음을 의미한다.

## Behavior interface

따라서 우리는 "예약하기" 엔티티에 이벤트 소싱을 적용하기로 결정하였다. 이제 몇 가지 행동을 정의해볼 차례이다.

먼저 우리의 엔티티에 Tagless Final algebra를 적용시킨다. 사실 이는 Aecor 또는 그 외 라이브러리들이 필요 없는 작업이다. 적용한 코드는 다음과 같다.

{{< gist tomatophobia 99883d49734e80538376af5596a0183d >}}

```scala
trait Booking[F[_]] {

  def place(client: ClientId, concert: ConcertId, seats: NonEmptyList[Seat]): F[Unit]
  
  def confirm(tickets: NonEmptyList[Ticket]): F[Unit]
  def deny(reason: String): F[Unit]
  
  def cancel(reason: String): F[Unit]
  
  def receivePayment(paymentId: PaymentId): F[Unit]

  def status: F[BookingStatus]
  def tickets: F[Option[NonEmptyList[Ticket]]]
}
```

> 만약 당신이 Tagless Final에 익숙하지 않다면 웹에 좋은 포스트들이 많으니 참고하기 바란다. 난 [@LukaJacobowitz](https://twitter.com/LukaJacobowitz)의 [이 글](https://typelevel.org/blog/2017/12/27/optimizing-final-tagless.html)을 추천한다. 또는 나의 [Writing a simple Telegram bot with tagless final, http4s and fs2](https://pavkin.ru/writing-a-simple-telegram-bot-with-tagless-final-http4s-and-fs2/)를 참고하기 바란다.

그렇다면 "예약하기"가 할 수 있는 것들은 무엇이 있는지 짚어보자

- 우리는 **콘서트**에서 **하나 이상**의 **자리**를 **클라이언트**를 대신해 예약을 **실행(place)**할 수 있다.
- 예약은 승인(confirm)될 수 있다. 이는 자리 확정되고 가격이 결정된다는 것을 의미한다. 우리는 여기서 명시적인 승인 과정을 정의했다. 왜냐하면 실제 데이터 관리, 좌석 예약은 다른 시스템에서 완료될 것이기 때문이다. 승인은 다른 시스템과 비동기적으로 협동할 때 사용될 것이다. 그 외부 시스템에서 가격 또한 관리할 것이다. 따라서 예약이 승인되면 좌석은 **티켓**으로 ****변환된다. - 우리 케이스에서 티켓은 (좌석, 가격)으로 생각할 수 있다.
- 만약 뭔가 잘못된다면 (자리가 이미 예약됨) 예약은 **사유**와 함께 **거절**된다.
- 고객은 언제든지 예약을 **취소**할 수 있다.
- 예약금 받기(receive payment)는 당연히 "예약하기" 생명주기에서 중요한 액션이다.
- 우리 엔티티는 상태(status), 티켓(선택사항이다. 예약이 승인될 때까지 가격이 존재하지 않기 때문이다.)을 통해 내부 상태를 노출시킨다.

몇 줄 되지 않는 코드지만 꽤 많은 고민과 노력을 요한다. 그리고 이 시점에서 생겼을 몇 가지 질문에 대해 답해보겠다.

> 이 algebra는 내부 상태를 갖고 그것을 다루는 것처럼 보이는데 왜 그런 것인가?

사실이다. 그 이유는

- 우리는 **행동에 집중**하기 때문이다. 연료가 되는 내부 상태는 부차적인 요소이다. 우리는 행동 algebra를 내부 상태에 연결시키고 싶지 않았다.
- 만약 다른 컴포넌트가 이 행동을 호출할 때 "예약하기"의 내부 상태를 오염시켜서는 안될 것이다.

나는 이 상황을 다음과 같이 생각한다. "예약하기" algebra의 인스턴스는 *특정한 예약하기 엔티티 인스턴스*와 그것의 현재 상태를 나타낸다. 그리고 *메소드*는 네가 인스턴스를 통해 실행할 수 있는 *액션*이다.

> 왜 `F[Unit]`이 여러 곳에서 보이는가? 에러는 왜 없는가? 예를 들어 거절된 부킹에 돈을 낼 수 없는 것은 어떻게 표현되는가?

합리적인 의문이다. 여기서 `Unit`이 나타내는 것은 "Ack"(acknowledgement) 응답의 일종이다. 이는 액션이 성공했음을 의미한다. "예약하기"는 아마 내부 상태를 바꿀 것이지만 우린 신경쓰지 않는다. 이런 상황에서 `Unit`을 반환하는 것은 Tagless Final에서 매우 흔한 상황이다.

에러에 대해서는 MTL에 좋은 전통이 있다. 우리는 에러를 우리의 이펙트(effect) `F`를 핸들링하는 곳에 위임할 것이다.

그나저나 대부분의 Tagless Final algebras가 그렇듯 아마 `F`는 결국 `IO` 또는 `Task`가 될 것이다. 스포일러: 우리가 이벤트 소싱된 행동을 다룰 때는 그렇게 되지 않을 것이다.

> 만약 이 액션들이 예약만을 위한 것이라면 왜 `place`가 여기 있는가? 새로운 예약을 `create` 하는 것이 더 합당하지 않나?

재미있는 질문이다. 만약 전통적인 [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete)였다면 엔티티 인스턴스를 생성하는 것은 그 엔티티가 갖는 (혹은 갖지 않는) 여러 로직들과 분리되어 있을 것이다. 하지만 우린 완전히 "행동(behavior)" 중심 세상으로 왔기 때문에 반드시 엔티티를 존재하게 하는 어떠한 종류의 "비즈니스 액션"이 있어야 한다. 우리의 상황에 대입해보면 `place`가 그 역할을 한다. 이 액션은 우리 도메인의 필수적인 동사이고, 엔티티 라이프사이클의 일부이다. 그래서 엔티티 algebra에 속하게 된다.

## Behavior actions, MTL-style

이제 슬슬 앞으로 나갈 준비가 된 것 같다. 마침내 Aecor의 타입클래스들을 하나씩 둘러볼 차례다. 하나씩 살펴보자.

메인이 되는 녀석은 `MonadAction`이다.

{{< gist tomatophobia efb38ba71942b89b80b27b36d061f316 >}}

```scala
trait MonadAction[F[_], S, E] extends Monad[F] {
  def read: F[S]
  def append(es: E, other: E*): F[Unit]
  def reset: F[Unit]
}
```

이는 액션들을 구성하기 위핸 기본적인 빌딩 블락들을 제공한다. Aecor의 액션은 어떻게 엔티티가 입력되는 커맨드들에 반응하는지 기술한다. 이는 커맨드 *핸들러 컨셉*과 매우 유사하다. 타입 시그니쳐를 통해 액션에 대해 몇 가지 알 수 있는 점이 있다.

- 액션은 `S` 타입을 갖는 상태에 의존한다. 우리는 `read`를 통해 상태를 조회할 수 있다.
- 액션은 커맨드에 반응하여 `E` 타입을 갖는 이벤트들을 생산(또는 추가, `append`) 한다.
- 액션을 호출한 caller에게 어떠한 종류의 결과를 반환한다.

따라서 임의의 이펙트 `F`에 대해 우리는 위에서 정의한 것들을 이용하여 액션들을 기술할 수 있다. (`MonadAction`의 인스턴스를 가질 것이다.)

또한 에러들도 필요하다. 커맨드 핸들링 관점에서 에러는 현재 엔티티의 상태에서 커맨드가 실행될 수 없음을 의미한다. 예를 들어 거절된 예약에 대해서는 돈을 내는 것이 불가능하다. 이 상황에서 `receivePayment` 커맨드는 *거부(reject)*될 것이다. 그리고 액션의 결과는 *거부(rejection)*가 된다.

Aecor는 에러를 사용할 수 있는 `MonadAction`보다 더 강력한 버전을 제공한다. 이를 `MonadActionReject`라고 부른다.

{{< gist tomatophobia 12d80ea59d4af031531381bbae3478e3 >}}

```scala
trait MonadActionReject[F[_], S, E, R] extends MonadAction[F, S, E] {
  def reject[A](r: R): F[A]
}
```

`MonadActionReject`와 `MonadAction`의 관계는 `MonadError`가 `Monad`와 엮여있는 관계와 같은 원리이다. 보통 당신의 엔티티들은 거부가 필요하겠지만 전혀 사용하지 않는 경우도 있다. 이런 상황에서 `MonadAction`을 유용하게 사용할 수 있다.

우리가 사용할 액션들을 구현하기 전에 이벤트 소싱된 예약에서 사용할 `S`, `E`, `R` 타입에 대해 이해하고 넘어가자.

### Events

이벤트 소싱을 구현하는 것은 전통적인 상태 기반 접근에 비해 본질적으로 난이도가 높다. 그 이유는 *상태* 대신에 *이벤트*들을 사용하기 때문이다. (또한 우리의 경우 거부도 고려해야 한다.)

도메인에서 적절한 이벤트를 발굴하는 것은 중요한 토픽 중 하나이다. 우리가 이미 도메인 전문가들과 함께 [이벤트스토밍](https://www.eventstorming.com/) 과정을 거쳐 다음 이벤트들이 나왔다고 가정하겠다.

{{< gist tomatophobia f6b18adc5dc454127e87a9c102b962cd >}}

```scala
sealed trait BookingEvent extends Product with Serializable

case class BookingPlaced(clientId: ClientId, concertId: ConcertId, seats: NonEmptyList[Seat]) extends BookingEvent
case class BookingConfirmed(tickets: NonEmptyList[Ticket]) extends BookingEvent
case class BookingDenied(reason: String) extends BookingEvent
case class BookingCancelled(reason: String) extends BookingEvent
case class BookingPaid(paymentId: PaymentId) extends BookingEvent
case object BookingSettled extends BookingEvent
```

> `BookingPaid`와 `BookingSettled`는 서로 다른 이벤트이다. 왜냐하면 어떤 예약은 무료여서 돈을 내지 않고 완료될 수 있기 때문이다.

우리가 의존성을 갖는 것이 없다는 것(no-dependency mode)에 주목하기 바란다. 이 이벤트들은 완전히 임의적이며 라이브러리에 의존성이 없다. [marker trait](https://link.springer.com/chapter/10.1007/978-3-319-02192-8_6) 또는 비슷한 개발 기술이 들어가지 않았다. (Maximum composition)

또한 우리는 아무런 식별 정보(identity information) 또는 메타데이터(예: 타임스탬프)를 넣지 않았다. Aecor는 비즈니스 관련(business-releated) 데이터와 메타데이터를 분리하는 방법을 제공하여 이벤트들을 깔끔하게 만들 수 있다. 어떻게 이벤트들이 메타데이터를 포함하여 확장시키는지 뒤에서 더 자세히 알아볼 것이다. 식별 정보에 대해서는 곧 이야기할 것이다.

### State

다음으로 우리의 엔티티 내부에 상태를 관리해야 한다. 이 시점에서 우리는 데이터베이스 스키마를 의식하는 함정에 빠지면 안된다. 우리가 상태를 사용하는 이유는 데이터베이스 테이블에 잘 호환되는 좋은 쿼리를 만들기 위함이 아니다. 상태 또한 도메인 모델이기 때문에 다음을 만족해야 한다.

- 읽기 쉬워야 하고 유비쿼터스 언어(ubiquitous languate)를 사용해야 한다.
- 커맨드와 이벤트 핸들링을 표현할 수 있을만큼 표현력이 충분히 풍부해야 한다.
- 엔티티 라이프사이클 전체를 지원해야 한다.

우리는 다음 상태들을 우리의 엔티티에 사용할 것이다.

{{< gist tomatophobia 8bc5b3b75b2eae8aaee39df622c2857d >}}

```scala
// State itself
case class BookingState(clientId: ClientId,
                        concertId: ConcertId,
                        seats: NonEmptyList[Seat],
                        tickets: Option[NonEmptyList[Ticket]],
                        status: BookingStatus,
                        paymentId: Option[PaymentId])

// data definitions that are used in BookingState

case class Money(amount: BigDecimal) extends AnyVal

case class ClientId(value: String) extends AnyVal
case class ConcertId(value: String) extends AnyVal

case class Row(num: Int) extends AnyVal
case class SeatNumber(num: Int) extends AnyVal

case class Seat(row: Row, number: SeatNumber)

case class Ticket(seat: Seat, price: Money)

case class PaymentId(value: String) extends AnyVal

sealed trait BookingStatus extends EnumEntry

object BookingStatus extends Enum[BookingStatus] with CirceEnum[BookingStatus] {
  case object AwaitingConfirmation extends BookingStatus
  case object Confirmed extends BookingStatus
  case object Denied extends BookingStatus
  case object Canceled extends BookingStatus
  case object Settled extends BookingStatus

  def values: immutable.IndexedSeq[BookingStatus] = findValues
}
```

> `tickets`은 Option을 사용하였다. 왜냐하면 전체 "예약하기" 과정에서 좌석의 가격이 없을 수도 있기 때문이다. (티켓은 confirmation 과정에서 확정된다.) 더 타입-안전(type-safe)한 방법은 `tickets` 필드에 Option을 사용하지 않고 티켓을 내부에 삽입하는 것이다. 여기서는 간단한 구현을 위해 상태 표현에 Option을 사용하였다.

다시 강조하면 우리가 구현한 상태는 어떠한 라이브러리에도 의존하지 않는다.

### Identity in state and events

이 쯤에서 이런 질문이 나올 수 있다.

> 식별성(identity)에 대해 많이 언급했는데 `bookingId`는 도대체 어디있는 것이냐?

이것은 Denis Mikhaylov에게 들은 꽤 괜찮은 아이디어인데 *엔티티는 커맨드를 핸들링할 때 식별 정보를 필요로 해서는 안된다*. 아마 네가 커맨드를 올바른 엔티티 인스턴스에 *라우트*하기 위해서는 반드시 어떠한 종류의 식별자를 필요로 할 것이다. 하지만 그 비즈니스 로직 이후에는 *일반적으로 식별자를 신경쓰지 않는다*.

게다가 만약 선택된 식별자가 여전히 비즈니스 로직에서 필요하다면 대부분 그것을 2개로 분리할 수 있을 것이다: 순수한 식별을 위한 부분과 커맨드 핸들러가 동작하기 위해 필요로 하는 부분이다. 그렇다면 당신은 전자를 이벤트와 상태 바깥으로 옮기고 후자만을 남겨야 한다.

이 아이디어를 실제로 구현하고 매우 멋지다는 것을 깨달았다. 관심사의 분리는 항상 옳다. 그래서 질문에 답변하면 뒤에서 `bookingId`를 보게 될 것이다. 하지만 그것은 우리의 행동(behavior)과는 큰 관련이 없다.

### Rejections

거부(rejection)에 많은 시간을 투자하지 않을 것이다. 간단한 열거형(enum)으로 충분할 것이다. 하지만 누구도 네가 사용할 거부에 몇 가지 데이터를 추가해 확장하는 것을 말리지 않는다. 다음은 우리가 사용할 예약하기 커맨드 거부들이다.

{{< gist tomatophobia 9c3269bab81ad96cd27432a99698902b >}}

```scala
sealed trait BookingCommandRejection

case object BookingAlreadyExists extends BookingCommandRejection
case object BookingNotFound extends BookingCommandRejection
case object TooManySeats extends BookingCommandRejection
case object DuplicateSeats extends BookingCommandRejection
case object BookingIsNotConfirmed extends BookingCommandRejection
case object BookingIsAlreadyCanceled extends BookingCommandRejection
case object BookingIsAlreadyConfirmed extends BookingCommandRejection
case object BookingIsAlreadySettled extends BookingCommandRejection
case object BookingIsDenied extends BookingCommandRejection
```

### Implementing Actions

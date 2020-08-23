---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "함수형 디자인 소개 [번역]"
subtitle: ""
summary: ""
authors: []
tags: [Functional Programming, Scala]
categories: [Functional Programming]
date: 2020-08-22T18:41:23+09:00
lastmod: 2020-08-22T18:41:23+09:00
featured: false
draft: false

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

이 글은 John A De Goes의 [An Introduction to Functional Design](https://degoes.net/articles/functional-design)을 읽고 번역 및 요약한 글입니다. 발번역이지만 도움이 되길 바랍니다.

## An Introduction to Functional Design

함수형 프로그래밍은 복잡하고 이론적이여야 한다고 생각히기 십상이다. 그러나 오히려 함수형 프로그래밍은 간단하고 자연스럽고 실용적이면서 프로그래머가 문제를 해결할 때 이전에는 겪을 수 없는 힘과 기쁨을 함께 느끼게 해준다.

이러한 함수형 프로그래밍의 잠재력을 끌어내기 위해서는 *함수형 디자인 (Functional Design)* 에 대한 이해가 필요하다. ([Ziverge](https://ziverge.com/), [Patron metorship program](https://patreon.com/) 을 참고하라) 함수형 디자인은 함수형 프로그래밍에 실용성을 부여한다. 함수형 프로그래밍의 정수를 하나의 툴킷으로 만들어 시간, 장소, 언어, 레거시 코드 또는 아무런 베이스가 없을 때도 아무런 제약없이 사용할 수 있다. 

물론 함수형 디자인에 입각한 개발 스킬을 익히기 위해서는 시간과 연습이 필요하지만 그 컨셉은 매우 심플하다. (그래서 블로그에 포스트할 주제로 적합하다) 그래서 이 글을 통해 함수형 디자인에 대해 설명하겠다.

## Elements of Functional Design

*함수형 디자인* 에서 우리는 관심이 있는 도메인 내부에서 어떤 해법으로 문제로 해결하는 과정을 모델링하기 위해 불변 데이터 타입 (immutable data type)과 생성자를 함께 만든다. 이를 통해 우리는 *간단한 해법* 과 해법들을 서로 결합하고 변환할 수 있는 연산자 (composable operator)를 만들 수 있다. 따라서 우리는 몇몇의 기초 연산자, 생성자들을 이용해 관심이 있는 도메인 내부의 가능한 모든 문제를 해결할 수 있다.

아쉽게도 위의 설명은 너무 추상적이다. 이제 저 설명을 쪼개서 하나씩 이해해보자.

- *함수형 디자인*. 함수형 디자인은 객체 지향 디자인의 대안이라고 할 수 있다. 두 디자인 모두 유지보수성을  높이고, 결함을 줄이려는 공통의 목적을 가지고 있다.
- 불변 데이터 타입 (Immutable data types). 함수형 프로그래밍에서는 불변 데이터 타입을 사용한다. 불변 데이터 타입을 사용하는 경우 동시성 어플리케이션, 제어의 역전(invert control, 특정 함수를 호출할 때 전달하는 데이터를 그 함수가 멋대로 바꾸지 못한다.)에서 유리함을 갖는다. 따라서 코드 유지보수가 쉽고, 위험성도 적은 편이다.
- *모델*. 대부분의 함수형 코드는 곧바로 문제를 해결하지 않는다. 대신 해법으로 문제를 해결하는 과정을 모델로 만들고 (models of solutions to problems) 이를 실제 문제가 발생했을 때 실행시킨다.  예를 들어 REST 웹 서버는 URL 경로에 따른 핸들링을 모델로 만들고 나중에 요청이 발생했을 때 실행된다. 모델은 항상 불변 데이터 타입을 이용하여 표현한다.
- *해법 (Solutions)*. 어플리케이션의 모든 레벨들에 있는 문제들은 각각의 해법을 필요로 한다. 데이터는 유효성 검증이 필요하고, 요청은 응답이 필요하며, 비즈니스 로직은 트리거 이메일이 필요하고, 숫자들은 총계되기를 (aggregate) 원한다. 그 외에도 계속 ...
- *관심 있는 도메인 (Domain of Interest)*. 대부분의 어플리케이션은 수십 개에서 수백 개에 이르는 관심 도메인들이 있다. 각 도메인들은 관련된 엔티티 유사한 문제들을 포함하고 있다. 그리고 우리는 문제들을 쉽게 테스트하고 유지보수 할 수 있는 방법으로 풀려는 목적을 갖는다.
- *생성자 (Constructor)*. 우리는 생성자를 통해 불변 데이터 타입을 생설할 수 있다. 또한 불변 데이터 타입으로 해법으로 간단한 문제를 해결하는 모델을 만든다.
- *합성가능한 연산자 (Composable Operators)*. 우리는 연산자를 이용해서 해법으로 부분 문제를 해결하는 모델들을 합성할 수 있다. 이 과정을 통해 해법으로 더 크고 복잡한 문제를 해결하는 모델을 만들 수 있다.

함수형 디자인은 레거시 코드 또는 아무런 베이스가 없는 코드에서 모두 잘 적용될 수 있다. 물론 프로그래머는 두 상황에서 약간 다른 방법을 사용해야 한다. 

*함수형 도메인* 은 특정 관심 도메인을 참조하기 위한 지름길 같은 역할을 한다. 이 과정에는 해법으로 문제를 해결하는 모델을 만들 때 사용할 불변 데이터 타입, 생성자, 연산자 들이 포함된다.

다음 섹션에서는 함수형 도메인의 몇 가지 예시를 보여주겠다.

> 아래부터 해법으로 문제를 해결하는 모델 (solution to problem)을 해법=>문제 로 표기하겠습니다.

## Example Domains

함수형 디자인은 훌륭한 함수형 프로그래밍 라이브러리의 심장에 있다고 할 수 있다. 

아래는 우리가 이미 알고 있는 몇 가지 예시들이다.

- **파서 컴비네이터 (Parser Combinators)**. 파서 컴비네이터는 텍스트 포맷을 파싱하여 구조적인 데이터로 변환하는 도메인을 위해 만들어졌다. 파서 자체는 문자 입력을 데이터 조각으로 변환하는 방법을 기술한 모델이다. 파서 생성자를 통해서 간단한 파서를 생성할 수 있고, 연산자를 통해 파서끼리 변환하거나 합성해서 더 복잡한 형식의 문자를 파싱할 수 있다. 프로그래머는 파서를 실행하고 인풋을 넣으면 에러를 발생시키거나 구조화된 데이터로 변환할 수 있다.

- **함수형 효과 (Functional Effects)**. [ZIO](https://zio.dev/) 와 같은 함수형 효과 라이브러리는 동시성 프로그래밍, 안전한 에러 관리, 리소스 핸들링에 사용된다. 효과(effect)는 순차적 또는 동시적 (concurrent) 작업들을 어떻게 실행할 지를 기술한 모델이다.  효과 생성자를 통해 간단한 비동시적 작업들을 만들거나 연산자를 통해 효과를 변환, 합성할 수 있다. 이를 통해 더 복잡한 동시적 작업을 모델링할 수 있다. 프로그래머는 효과를 실행하여 기술해놓은 작업을 작동시킬 수 있다.

- **옵틱스 (Optics)**. [Monocle](https://www.optics.dev/Monocle/) 과 같은 옵틱스 라이브러리는 불변 데이터에 접근하거나 변환할 때 사용한다. 옵틱은 데이터 조각을 확대(zoom), 조회(retrieve)하거나 수정하는 방법들을 표현한 모델이다. 옵틱 생성자로 간단한 접근자와 수정자를 만들 수 있고, 연산자를 이용해 다른 옵틱들을 변환하거나 합성하여 매우 크고 복잡한 데이터에 접근하거나 수정할 수 있다. 프로그래머는 옵틱스를 실행하여 데이터와 변환을 넣고 변환된 데이터를 얻을 수 있다.

  > Optics를 광학이라고 번역할까 하였지만 쉽게 의미를 알기 어려워 옵틱스라고 하였습니다.

- **스트림 (Streams)**. [ZIO Stream](https://zio.dev/) 과 같은 스트림 라이브러리는 동시성 스트리밍 데이터 도메인을 위해 만들어졌다. 스트림은 동시에 발생하는 값 시퀀스를 모델링한다. 스트림 생성자는 간단한 스트림을 만들고 연산자는 스트림을 변환 합성하여 더 복잡한 데이터 파이프 라인 모델을 만들 수 있다. 프로그래머는 스트림을 실행함으로써 소스 위치에서 데이터를 읽고 모델별로 동시적으로 데이터를 변환, 합성할 수 있다.

지금부터 ZIO의 함수형 도메인 예시를 실제 코드 레벨까지 파보면서 최대한 구체적으로 확인해보겠습니다.

## In Depth: ZIO Schedule

*ZIO Schedule* 은 해법=>유한하거나 무한하게 반복되는 스케줄을 구현하는 문제를 모델링하는 불변 데이터 타입이다. 다른 함수형 도메인과 마찬가지로 모델, 생성자, 연산자가 있다. 

이 모델의 타입은 `zio.Schedule[Env, In, Out]`이다. 이는 스케줄을 실행하는 환경인 `Env` 필요한 인풋 타입인 `In`, 아웃풋 타입인 `Out`으로 이루어져 있다.

에러 발생으로 실패한 효과를 재시도 할 때 스케줄을 사용하는 경우, 스케줄의 인풋 타입은 이펙트의 에러타입이 될 것이다. 반대로 성공한 효과를 반복할 때 스케줄을 사용하는 경우, 스케줄의 인풋 타입은 이펙트의 성공(success) 타입이 될 것이다.

다른 모델들처럼 `Schedule` 은 반복하는 스케줄을 기술하는 것 외에 따로 실행하는 작업이 없다. 스케줄 모델을 실행하기 위해서는 아무 ZIO 효과의 `retry`, `repeat` 메소드를 실행한 후 스케줄 모델에 주입하면 된다.

생성자는 해법=>매우 간단한문제 모델을 생성한다. 예를 들어 우리가 매 주 목요일마다 반복되는 스케줄을 기술한다고 할 때 다음과 같이 `Schedule.dayOfWeek` 생성자를 사용할 수 있다. 

```scala
val onThursday = Schedule.dayOfWeek(DayOfWeek.THURSDAY)
```

유사한 방법으로 오전 6시, 낮 12시에 반복되는 스케줄을 기술할 때는 `Schedule.hourOfDay` 생성자를 사용한다.

```scala
val sixAndTwelve = Schedule.hourOfDay(6, 12)
```

함수형 디자인의 다른 모델들처럼 스케줄 모델도 연산자를 통해 스케줄들을 변환, 합성할 수 있다. 따라서 우리는 더 어려운 문제를 간단한 부분 해법=>문제 모델들을 이용해 해결할 수 있다. 

예를 들어 우리가 목요일 오전 6시, 낮 12시에 반복되는 스케줄을 만든다고 할 때 우리는 `&&` (intersection) 연산자를 이용해 위에서 정의한 2개의 스케줄을 합성할 수 있다.

```scala
val thursdaySixAndTwelve = onThursday && sixAndTwelve
```

유사한 방법으로 이 스케줄을 수정하여 일정한 아웃풋을 발생시키고 싶다면 (예를 들어 유닛 값인`()`), `map` 을 이용하여 변환된 아웃풋을 발생시키는 새로운 스케줄을 만들 수 있다.

```scala
val finalSchedule = thursdaySixAndTwelve.map(_ => ())
```

*ZIO Schedule* 은 함수형 도메인이기 때문에 몇 가지 생성자와 연산자를 이용해 매우 다양한 스케줄링 문제들을 해결할 수 있다. 함수형 디자인의 멋진 원리 덕분에 테스트하기 쉽고, 코드를 이해하기 편하고, 비즈니스 요구사항에 맞춰서 쉽게 변경할 수 있다.

지금까지 예시를 통해 함수형 디자인에 대해 자세히 알아보았다. 이제 우리는 간단한 함수형 도메인을 만들고 모델을 만들기 위해 어떤 선택들이 있는지 알아볼 것이다.

## Email Filtering

간단한 이메일 웹 어플리케이션을 만든다고 생각해보자. 어플리케이션 사용자에게는 이메일 필터를 생성하는 기능이 필요하다. 그 기능을 이용해 이메일을 포워드 하거나 이메일을 특정 폴더에 옮기거나 삭제할 수 있다. ([이메일 필터링](https://en.wikipedia.org/wiki/Email_filtering)) 

사용자 입장에서는 다양한 필터를 생성하고 싶을 것이다. 또한 개발자 입장에서 우리는 다양한 유즈 케이스를 지원하면서 동시에 테스트 가능하고 적은 유지보수 비용이 들어가는 어플리케이션을 원할 것이다. 이런 이점을 얻기 위해서 우리는 함수형 디자인 테크닉을 사용해야 한다.

### The Model

이메일 필터를 위한 새로운 함수형 도메인에서 가장 먼저 필요한 것은 *모델* 이다. 여기서 모델은 불변 데이터 타입이 될 것이고, 사용자가 생성한 자신이 받는 메일에 적용할 필터에 대해 기술한다. 이 모델을 실행할 방법 중 하나로는 직접 모델을 이메일에 적용시켜 필터가 어떻게 이메일을 판별하는지 확인하는 것이다.

함수형 디자인에서는 모델을 인코딩하는 두 가지 방법이 있다. (여기서 인코딩은 모델을 기술하는 방식을 의미하는 것 같습니다.)

- **실행가능한 인코딩 (Excutable Encoding)**. 이 인코딩에서 우리는 모든 생성자, 연산자를 모델 자체의 실행으로 표현한다.
- **선언적 인코딩 (Declarative Encoding)**. 이 인코딩에서 우리는 생성자, 연산자를 이용해 모델을 재귀적인 트리 구조의 순수한 데이터로 표현한다.

첫 번째 인코딩을 다른 말로 *종단 (final)* 이라고 한다. (그 이유는 생성자와 연산자가 최종적으로 계산된 형식으로 표현되기 때문이다. 그리고 이 최종 형식은 미리 결정된다.) 반면 두 번째 인코딩은 다른 말로 *초단(initial)* 이라고 불린다. (이유는 초기 형식이 주어지고, 나중에 모델은 계산을 거쳐 여러 가지 형식으로 변경되기 때문이다. 이때 최종 형식은 미리 결정되지 않는다.) 

함께 이메일 필터를 두 가지 인코딩으로 나눠서 어떻게 구현되는지 확인해보자.

(원글에서 "predetermined in advance"가 정확히 어떤 것이 미리 결정되는지 설명하기 어려워서 직역하였습니다.)

### Excutable Encoding

실행 가능한 인코딩은 `case class` 타입을 이용하거나 선결정적(predetermined)으로 모델을 실행하는 함수들을 가지고 있는 `trait` 타입을 이용하여 나타낼 수 있다.

이 케이스에서 우리는 이메일 필터를 실행시키기 위해 이메일에 적용시켜서 필터가 이메일을 판별하는지 확인해볼 것이다. 따라서 `EmailFilter` 의 실행가능한 인코딩을 만들기 위해 우리는 이메일을 받아서 참, 거짓을 반환하는 함수를 가진 `case class` 정의한다.

```scala
final case class EmailFilter(matches: Email => Boolean)
```

위 모델 내부의 `matches` 함수를 이용하여 모델을 실행시킬 수 있다. 이메일을 넣으면 그 이메일을 판별하여 참, 거짓 반환한다.

이제 기존 생성자를 발전시켜서 이메일이 특정 문구를 포함하고 있는지 판별할 수 있는 해법=>문제 모델을 만들어 보았다.

```scala
def subjectContains(phrase: String): EmailFilter =
  EmailFilter(_.subject.contains(phrase))
```

이 외에도 좀 더 현실적인 예시로 이메일 본문 내용에 관한 필터링, 수신자 리스트에 관한 필터링, 보낸 날짜로 필터링 등이 있을 것이다.

생성자들은 간단한 문제만 해결할 수 있다. 따라서 더 복잡한 문제를 풀기 위해서는 기존 모델을 변환, 합성할 수 있는 연산자가 필요하다. 그러므로 이러한 메소드들을 `EmailFilter` 클래스 내부에 넣어보겠다.

```scala
final case class EmailFilter(matches: Email => Boolean) { self =>
  def &&(that: EmailFilter): EmailFilter =
    EmailFilter(email => self.matches(email) && that.matches(email))
  
  def ||(that: EmailFilter): EmailFilter =
    EmailFilter(email => self.matches(email) || that.matches(email))
  
  def unary_!: EmailFilter =
    EmailFilter(email => !self.matches(email))
}
```

위의 단항 연산자와 이항 연산자들만으로도 꽤나 재미있는 이메일 필터링 문제들을 해결할 수 있다. 예를 들어 다음 예시 "discount", "clearance"를 포함하고 "liquidation"을 포함하지 않는 이메일을 판별하는 필터이다.

```scala
val filter =
  (subjectContains("discount") || subjectContains("clearance"))
  !subjectContains("liqudation"))
```

실행가능한 인코딩은 직설적이고 꽤 괜찮다. 이메일 필터를 직접적인 실행으로 표현하기 때문에 이메일 필터에 대해 잘 모르거나 함수형 디자인에 익숙하지 않아도 쉽게 이해할 수 있다. 

### Declarative Encoding

선언적 인코딩은 `sealed trait` 타입 또는 `enum` 타입 (Scala 3)를 이용해 구현한다. 이 경우 서브 타입이 하나의 모델과 대응되고 내부에는 기본적인 생성자와 연산자를 가지고 있다. 각 서브 타입들은 생성자와 연산자들이 필요로 하는 인자(argument)들을 수집하여 재귀적인 순수 데이터 타입 형태로 저장한다.

이전에 보았던 실행가능한 인코딩에서 사용한 기능들을 똑같이 구현해보자. 이를 위해서는 `sealed trait` 에 4개의 서브타입이 필요하다. 하나는 생성자, 하나는 단항 연산자, 둘은 이항 연산자이다. 

```scala
sealed trait EmailFilter { self => 
  def &&(that: EmailFilter): EmailFilter = And(self, that)
  
  def ||(that: EmailFilter): EmailFilter = Or(self, that)
  
  def unary_!: EmailFilter = Not(self)
}
final case class SubjectContains(phrase: String) extends EmailFilter
final case class And(left: EmailFilter, right: EmailFilter) extends EmailFilter
final case class Or(left: EmailFilter, right: EmailFilter) extends EmailFilter
final case class Not(value: EmailFilter) extends EmailFilter

def subjectContains(phrase: String): EmailFilter = SumbjectContains(phrase)
```

위에서 볼 수 있듯이 선언적 인코딩에서 각 서브타입들은 연산자 또는 생성자에 필요한 인자를 저장할 뿐이다. 각 서브타입들은 이메일 필터들이 어떻게 생성되고 변환, 합성하는지 과정만 기술할 뿐 실제로 실행하지 않는다.

실행가능한 인코딩 예시의 `filter` 를 똑같이 선언적 인코딩 형태로 나타내본다면 다음과 같을 것이다.

```scala
And(
  Or(SubjectContains("discount"), SubjectContains("clearance"))
  Not(SubjectContains("liquidation"))
)
```

다른 데이터 트리와 마찬가지로 위의 트리는 변환하거나 내부를 조회할 수 있다.

선언적 인코딩을 사용한 모델은 순수한 데이터일 뿐이므로 내부에 모델을 실행시킬 방법이 없다. 따라서 우리가 만든 필터가 올바르게 이메일을 판별하는지 테스트하기 위해 간단하게 모델을 실행시키는 함수를 하나 작성해야 한다. (*인터프리터* 또는 *실행기 (executor)* 라고 부른다.)

다음의 실행기는 모델을 하나 받아서 필터가 이메일을 잘 판별하는지 테스트한다.

```scala
def matches(filter: EmailFilter, email: Email): Boolean = 
  filter match {
    case And(l, r) => matches(l, email) && matches(r, email)
    case Or(l, r) => matches(l, email) || matches(r, email)
    case Not(v) => !matches(v, email)
    case SubjectContains(phrase) => email.subject.contains(phrase)
  }
```

선언적 인코딩은 간접적으로 추가적인 처리를 하는 계층이 존재한다. 하지만 좀 더 넓은 관점에서 보면 해법=>문제 모델의 형태를 자연스럽게 기술한다는 장점이 있다. (모델의 실행을 위해서는 데이터 구조체를 탐색하는 별개의 과정이 필요하다.)

> The declarative encoding has a layer of indirection, which adds additional ceremony. However, it is a nice way to think about models in general, because it is obvious they describe solutions to problems—the execution of the model requires a separate traversal of the data structure.
>
> 역: 위 문단은 이해가 어려워 원문을 첨부합니다. 제 생각에는 실행기가 따로 필요하기 때문에 그 부분을 추가적인 간접 계층이라고 표현한 것 같고, 언뜻 부자연스러울 수 있지만 실행기 자체가 solutions to problems에 대한 기술을 담고 있는 것이 꽤 괜찮다. 라고 말하는 것 같습니다.

## New Interpreters

이전 예시에서 우리는 `EmailFilter` 모델을 실행시킬 때 이메일을 넣어주고 필터의 이메일 판별 결과에 따라 참, 거짓을 받았다. 하지만 이는 모델을 실행시키는 수많은 방법 중에 하나일 뿐이다. 예를 들면 모델을 실행시킬 때 이메일 필터가 어떻게 생성, 변환, 합성 되었는지 문자열 형태로 나타내고 싶을 수 있다. (디버깅을 하거나 사용자에게 보여주기 위한 목적)

이 작업을 실행가능한 인코딩, 선언적 인코딩 두 가지에서 모두 수행할 수 있다.

실행가능한 인코딩에서는 간단하게 `case class` 내부에 간단한 `() => string` 타입의 함수를 넣으면 된다. 두 번째 인자인 이 실행기는 이메일 필터를 표현한 문자열을 생성한다.

```scala
final case class EmailFilter(matches: Email => Boolean, describe: () => String) { 
  self => 
    def &&(that: EmailFilter): EmailFilter =
      EmailFilter(
        email => self.matches(email) && that.matches(email),
        () => s"(${self.describe()} && ${that.describe()})")
    
    def ||(that: EmailFilter): EmailFilter =
      EmailFilter(
        email => self.matches(email) || that.matches(email),
        () => s"(${self.describe()} || ${that.describe()})")
    
    def unary_!: EmailFilter =
      EmailFilter(email => !self.matches(email),
        () => s"!${self.describe()}")
}
def subjectContains(phrase: String): EmailFilter =
  EmailFilter(_.subject.contains(phrase), () => s"(subject contains ${phrase})")
```

> 원문에서는 that.describe에 ()를 붙이지 않아 lazy value인 상태였는데 개인적으로 오타라고 판단해 수정하였습니다. 

일반적으로 모델을 실행시키는 `n` 개의 방법이 있다고 한다면 실행 가능한 인코딩에서는 `case class` 내부에 `n` 개의 함수를 가져야 한다. (또는 `trait` 내부에 `n` 개의 메소드를 가진다.)

따라서 실행가능한 인코딩에서는 새로운 인터프리터가 생길 때마다 생성자를 사용하는 모든 코드에 일일이 인터프리터 부분을 추가해야 하기 때문에 잠재적으로 코드의 양이 계속 늘어난다는 단점이 있다. 반면 실행가능한 인코딩에서 다른 생성자(기본 생성자 제외)와 연산자를 통해 *유도된 (derived)* 연산자 또는 생성자를 추가하는 경우에는 유지보수 비용을 최소화 할 수 있는 장점이 있다. 

선언적 인코딩의 경우 모델은 순수한 데이터이기 때문에 새로운 인터프리터가 필요하다면 그 역할을 하는 함수를 추가하면 된다. 

```scala
def describe(filter: EmailFilter): String =
  filter match {
    case And(l, r) => s"(${describe(l)} && ${describe(r)})"
    case Or(l, r) => s"(${describe(l)} || ${describe(r)})"
    case Not(v) => s"!${describe(v)}"
    case SubjectContains(phrase) => s"(subject contains ${phrase})"
  }
```

따라서 선언적 인코딩에서는 모델을 실행하는 새로운 실행기를 추가하는 것이 비교적 쉽다는 장점이 있다.

### Encoding Tradeoffs

실행가능한 인코딩과 선언적 인코딩은 서로 *반대되는* 장점과 단점을 가지고 있다.

실행가능한 인코딩에서는 기존 코드를 수정하지 않고 새로운 생성자와 연산자를 추가하는 것이 자유롭다. 하지만 새로운 인터프리터를 추가하기 위해서는 기존의 모든 생성자와 연산자를 수정해야 한다.

선언적 인코딩에서는 새로운 인터프리터를 추가할 때 기존 코드를 수정할 필요가 없다. 그러나 새로운 (기본) 연산자와 생성자를 추가하기 위해서는 기존의 모든 인터프리터를 수정해야 한다.

> [Expression problem](https://en.wikipedia.org/wiki/Expression_problem)을 참고하면 좋을 것 같습니다.

함수형 프로그래밍에서는 실행가능한 인코딩과 선언적 인코딩이 서로 "짝 (duals)"을 이룬다. 이 둘은 서로 거울을 마주보듯 닮아있지만 반대편에 서있다. 두 인코딩에서 모두 새로운 생성자와 연산자를 추가하거나 새로운 인터프리터를 추가할 수 있다.

선언적 인코딩의 경우 순수한 데이터이기 때문에 실행가능한 인코딩에 비해 최적화하기 용이하다. 따라서 퍼포먼스가 중요한 상황에서 선언적 인코딩은 비교적 우위를 가지고 있다. 추가적으로 모델을 영속적으로 (persistence) 관리하는 경우 선언적 인코딩이 적합하다. 순수한 데이터이기 때문에 관계형 데이터베이스나 그 외 영속 계층에서 직접적으로 저장과 조회를 할 수 있다.

한편 실행가능한 인코딩의 경우 기존에 존재하는 (순수하지 않은) 인터페이스와 클래스들을 매끄럽게 재사용할 수 있기 때문에 주로 레거시 코드가 있는 상황에 적합하다. 예를 들어 기존 코드가 `InputStream` 을 많이 사용하는 경우 우리는 간단하게 함수형 스트림을 다음과 같이 쓸 수 있다.
```scala
final case class Stream(create: () => InputStream)
```

프로그램에서 선언적인 형식을 유지함으로써 위 데이터 타입은 기존 코드와의 연계를 쉽게 유지하면서도 많은 생성자와 연산자를 가질 수 있다. (이 때 리소스 핸들링과 중간(intermediate) 인풋 스트림을 위한 에러 확산을 신경쓸 필요 없다.)

## Where From Here

함수형 디자인은 함수형 프로그래밍을 실용적으로 사용할 수 있는 매우 강력한 도구이다. 함수형 디자인을 통해 현실에서 만날 수 있는 문제들을 쉽게 테스트할 수 있고 간단하게 유지보수하면서 재미있게 풀어나갈 수 있다.

물론 이 글에서 이야기한 내용은 기초일 뿐, 이 너머에는 어떻게 함수형 도메인을 타입 안전하게 만들지 (제네릭, 팬텀 타입, 타입 레벨 프로그래밍을 사용), 어떻게 직교성(orthogonality), 표현력(expressivity), 조합성(composablity)의 원리를 사용하면서 좋은 생성자와 연산자를 설계하는지, 변환과 합성을 하는 일반적인 패턴, 비즈니스 어플리케이션 내부에서 함수형 도메인을 정의하는 것 등 다양한 이야기들이 있다.

이 글이 재미있고 더 배우고 싶다면 [내 Github](https://github.com/jdegoes/functional-design) 또는 [Ziverge](https://ziverge.com/) 워크샵 또는 [Patreon](https://patreon.com/jdegoes) 의 스파르탄 티어에 가입하기 바란다. 또한 이 글에 주제와 관련해서 [the expression problem](https://en.wikipedia.org/wiki/Expression_problem), [object algebras](https://www.cs.utexas.edu/~wcook/Drafts/2012/ecoop2012.pdf), [tagless-final style](http://okmij.org/ftp/tagless-final/index.html) 등이  두 가지 다른 인코딩의 상충 관계에 대한 더 많은 배경 지식을 제공해줄 것이다.



## 끝

함수형 디자인에 대한 좋은 글을 추천받아서 공부하는 겸 번역해보았습니다. 항상 다른 프로그래밍 번역 서적을 읽을 때는 더 잘 번역할 수 있을 것 같았는데 제가 직접 해보니 쉬운 일이 아닌네요. 어색하거나 이상한 번역이나 이해가 안되는 점들은 댓글이나 이메일 주시면 정말 감사드리겠습니다.    :)


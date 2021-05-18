---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "ZIO를 이용해 의존성 관리하기 [번역]"
subtitle: ""
summary: ""
authors: []
tags: [ZIO, Scala, Functional Programming, Dependency Injection]
categories: [Functional Programming]
date: 2020-08-27T22:07:42+09:00
lastmod: 2020-08-27T22:07:42+09:00
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

이 글은 [Adam Warski](https://blog.softwaremill.com/@adamwarski)의 [Managing dependencies usin ZIO](https://blog.softwaremill.com/managing-dependencies-using-zio-8acc1539e276)를 번역한 글입니다.

---

[ZIO](https://zio.dev/)는 안전한 타입과 함께 조합 가능하며서 (composable) 비동기 (asynchronous) 또는 동시성 (concurrent) 프로그래밍을 할 수 있는 Scala의 라이브러리이다. 최근 자신들의 ["환경" 컴포넌트를 점검](https://zio.dev/docs/howto/howto_use_layers) 하였다. 여기서 그들은 프로그램을 실행시키기 위한 의존성을 관리 할 때 사용할 수 있는 새로운 접근법을 제시했다.

그런다면 ZIO의 "환경"이란 무엇을 말하는 걸까? ZIO에서는 비동기/동시성 프로세스를 **값**으로 표현하는데 이 때 그 **값**의 타입은 `ZIO[R, E, A]`이다. `E`는 실패할 수도 있는 연산에서 발생할 수 있는 에러를 나타내고, `A`는 연산의 결과 나오는 타입이다. 그리고 `R`은 "환경"(또는 필요조건) 나타내는데 이는 연산에 필요한 것들을 의미한다.

간단하게 `ZIO[R, E, A]`를 `R` 타입을 인자로 받고 연산이 실패할 경우 에러 타입 `E`, 성공할 경우 `A` 타입을 리턴하는  함수 `R => E | A` 로 **표현**할 수도 있다.  당연하지만 ZIO를 이용해 표현한 프로세스가 "보통의" 스칼라 함수보다 더 강력하다. ZIO에서 제공하는 몇 가지 컴비네이터로 비동기, 동시성, 에러 복구/시그널, 여러 환경 필요조건 등 다양한 상황의 연산을 안전하게 구현할 수 있다.

(역: 컴비네이터([combinator](https://stackoverflow.com/questions/58681172/what-is-a-combinator-in-scala-or-functional-programming))는 어떤 개체들을 결합하여 새로운 개체를 생성하는데 개체의 타입은 유지되는 연산자라고 이해하면 될 것 같습니다.)

어떤 어플리케이션을 개발하든 **의존성 관리**는 흔하게 문제를 발생시키고 관련된 논란도 많다. 그렇다면 ZIO에서는 어떻게 의존성을 관리할까? **의존성 주입 (Dependency Injection)**을 사용할 것인가? 만약 사용한다면 다른 의존성 주입 라이브러리들과 다른 점은 무엇일까?

우리는 이 글을 통해 위의 질문들을 하나씩 처리해볼 것이다.

> 이미 [다른 글](https://blog.softwaremill.com/zio-environment-meets-constructor-based-dependency-injection-6a13de6e000)을 통해 ZIO environment와 다른 의존성 주입 접근들을 비교한 바가 있다. 하지만 이는 이전 세대의 ZIO env까지에 적용되고 최근 구현체에서는 상당히 달라진 (그리고 발전된) 형태를 띄고 있기 때문에 새롭게 이 글을 쓰게 되었다.

## But What is DI?

의존성 주입에 대해 들어본 사람이라면 각자 자신만의 정의를 가지고 있을 것이다. 따라서 ZIO에 대해 이야기 하기 전에 내가 의존성 주입이란 말을 어떻게 생각하는지 말하는 것이 도움이 될 것 같다.

이전 글 ["What is Dependency Injection?"](https://blog.softwaremill.com/what-is-dependency-injection-8c9e7805502f) :

> 의존성 주입은 정적이고, 상태가 없는 서비스 오브젝트 그래프를 생성하는 프로세스이다. 각 서비스는 의존성에 의해 매개변수화(parameterised)된다. 
>
> (역: parameterised의 의미를 정확히 파악하지 못해 매개변수화라는 말로 표현하였습니다.)

반면 ZIO environment는 다른 목적으로도 쓰일 수 있다. 여기서 우리는 ZIO에서 어떻게 정적 오브젝트 그래프를 생성하는지에 집중할 것이다. 정적 오브젝트 그래프는 대부분 어플리케이션에 유용한 기능을 부여하기 위해 상태가 없는 서비스 형태로 쓰인다. 

## Basic building blocks

ZIO 기반의 어플리케이션에서 `ZIO[R, E, A]`가 기본 블록이 (building block) 되는 것처럼 의존성을 관리할 때 중요한 기본 블록 컴포넌트는 `ZLayer`이다.

`ZLayer[RInt, E, ROut]`은 `ROut`을 생성하기 위해 `RIn` 에 의존하고 있음을 나타낸다. 그리고 이러한 의존 관계 생성 과정이 실패할 경우 발생하는 에러의 타입은 `E`이다. 

위의 레이어를 이용해 단일 의존관계를 표현할 수 있다. 이는 생성자를 이용한 의존성 주입보다 더 강력하다. 다음은 일반적인 생성자를 이용한 의존성 주입 예시이다.

```scala
class ServiceA(b: ServiceB, c: ServiceC)
```

이 코드를 함수로도 표현할 수 있다.

```scala
(ServiceB, ServiceC) => ServiceA
```

ZIO env 에서는 위 상황을 타입과 함께 "값"으로 나타낼 수 있다.

```scala
ZLayer[Has[ServiceB] with Has[ServiceC], Nothing, Has[ServiceA]]
```

의존 관계 생성을 다양한 방법으로 표현할 수 있다. 위의 예시에서는 생성자를 이용한 방법을 보았다. 예를 들어 의존 관계 생성 과정에 효과 로직 (effectual logic)이 있는 경우가 있을 수 있다. 또는 생성 과정에서 얻은 리소스를 다 사용하고 난 후 방출(release)시키는 과정을 포함해야 할 수도 있다. 이런 경우 안전하게 서비스를 생성하는 것뿐만 아니라 더 이상 필요하지 않은 경우 닫아주는 과정 또한 필요하다. `ZLayer`가 가진 몇 가지 컴비네이터를 이용해 이런 다양한 사용 예를 해결할 수 있다.

> `Has[_]` 타입의 의미가 궁금한 독자들이 있을 것이다. `Has`는 ZIO에서 다중 의존성을 `with`을 이용해 표현하기 위해 만들어진 도구이다. 이 도구를 이용해 여러 의존성끼리 합치거나 제거하여 네임스페이스 충돌을 방지할 수 있다. 대부분의 `Has[_]` 는 기술적인 디테일로 다뤄지고 구현 디테일은 다루지 않는다.

레이어는 다중 의존 관계 생성도 표현할 수 있다. 한 레이어의 결과가 다른 레이어의 입력으로 사용되는 방식으로 레이어들을 결합하여 더 큰 크기의 레이어를 생성할 수 있다. 이는 마치 함수 합성과 비슷하다. 물론 이 과정을 위한 몇 가지 컴비네이트를 다양한 상황에 적용시킬 수 있다.

`ZLayer`는 어플리케이션의 서비스 그래프 단편을 생성하는 방법을 표현한다. 이는 의존성 주입의 정의와 잘 맞아떨어진다. 

당신이 방금 읽은 내용이 맞다. ZIO environment는 의존성 주입을 하나의 어플리케이션에 대응되도록 구현한다. 그와 동시에 타입 안전, 리소스 안전, 잠재적으로 동시적 실행, 에러 처리를 리플렉션([reflection](https://stackoverflow.com/questions/58681172/what-is-a-combinator-in-scala-or-functional-programming))이나 클래스패스 스캐닝 (classpath scanning)없이 할 수 있다.

## Recipe for single ZIO dependency

만약 ZIO environment와 `ZLayer`를 이용해 어플리케이션의 의존성을 관리하고 싶다면 우선 **단일 서비스**부터 시작하여 점진적으로 확장하는 것이 좋다. 퍼즐을 하나씩 맞추듯이 진행하다보면 완전한 서비스 그래프에 도달할 수 있다.

그렇다면 `ZLayer`를 이용해 단일 서비스를 만드는 방법은 무엇일까? 이 방법은 아마 여러분들이 이미 알고있는 생성자를 이용한 의존성 주입이나 그 외 다른 방법들과 비교해서 조금 복잡하게 느껴질 수 있다.

일반적으로 하나의 서비스를 구성하기 위해 4개의 컴포넌트를 결합한다. (물론 각 컴포넌트는 개별적으로도 동작할 수 있다.)

1. 서비스의 **인터페이스**. 인터페이스에 속하는 메소드들의 반환 타입은 일반적으로 효과 관련 연산의 결과이다. (effectful computation) 단, **어떠한 의존관계도 존재하지 않는다.** 즉, 각 비즈니스 메소드는 아무런 "환경"(필요조건)을 필요로 하지 않는 ZIO[Any, E, A] 타입을 반환한다. (여기서 `Any`의 의미는 "필요한 것이 없음" 또는 "모든 환경에서 작동함"으로 해석할 수 있다.) 여기서 우리가 필요로 하는 "환경"이란  서비스 구현의 **세부 사항**이라고 할 수 있다. 우리는 이러한 세부 사항을 서비스를 사용하는 유저로부터 **숨기기 위해** 위와 같은 일을 하는 것이다.
2. `ZLayer`로 표현되는 서비스의 구현. 서비스의 기능을 구현하기 위해 필요한 **의존관계**는 레이어의 **입력(input)**이 된다. 레이어의 출력은 서비스의 구현이 된다.
3. (선택사항) 인터페이스 내부 메소드들의 접근자. 이 값들은 서비스의 메소드를 호출하기 위한 필요조건들을 설명한다. 이들의 인스턴스들은 반드시 "환경" 내부에 존재해야 한다. 

4. (패키지 오브젝트 내부에서 선택사항) 타입 별칭(alias). 서비스의 인터페이스를 `Has[_]`로 감싼 후 타입 별칭으로 지정한다.

1과 2는 모든 의존 관계 관리/의존성 주입을 하는 경우 핵심적인 부분인데 반해 ZIO environment를 사용할 때 필요한 3은 전형적인 **보일러플레이트(boilerplate) 코드**이다. 스칼라는 유연하지만 이러한 작업을 자동으로 처리하는 부분에서는 유연성이 떨어진다. 비슷하게 4에서도 타입 별칭은 의존성 주입 코드의 가독성을 올리기 위한 기술적인 팁이다. 

3과 4는 선택사항이기 때문에 최소한으로 필요한 것은 서비스의 인터페이스를 참조하는 방법(예: `trait`, `class`)과 레이어 정의이다. 

## By example

위의 설명은 꽤나 추상적이였다. 예시를 통해 자세히 알아보자. 다음은 `UserModel` 서비스로 `insert` 메소드를 하나 가지고 있고 `DB` 서비스 인스턴스에 의존하고 있다. 

```scala
// UserModel.scala
import zio.{Task, ZIO, ZLayer}

object DB {
  // 1. service
  trait Service {
    def execute(sql: String): Task[Unit]
  }
  
  // 2., 3. omitted here
}

object UserModel {
  // 1. service
  trait Service {
    def insert(u: User): Task[Unit]
  }

  // 2. layer - service implementation
  val live: ZLayer[DB, Nothing, UserModel] = ZLayer.fromService { db =>
    new Service {
      override def insert(u: User): Task[Unit] = 
        db.execute(s"INSERT INTO user VALUES ('${u.name}')")
    }
  }

  // 3. accessor
  def insert(u: User): ZIO[UserModel, Throwable, Unit] = ZIO.accessM(_.get.insert(u))
}

// ---

// 4. type aliases in package.scala
type DB = Has[DB.Service]
type UserModel = Has[UserModel.Service]
```

위에서 짚었던 것들을 예시에서 다시 확인해보자.

1. 먼저 `insert` 메소드는 `Task[Unit]` (`ZIO[Any, Throwable, Unit]`의 별칭)을 반환한다. 우리는 이 메소드를 통해 사용자를 DB에 삽입할 수 있다. 위에서 언급했듯이 인터페이스의 메소드에 선언부에서는 아무런 의존관계도 명시되어 있지 않다. 의존관계는 구현 파트에서 제공된다.
2. `live` 레이어는 서비스의 구현부이다. 인터페이스와 별개로 정의되기 때문에 테스팅, 다른 설정을 사용하는 다른 구현도 존재할 수 있다. 위 코드에서는 `fromService`를 이용하여 레이어를 생성하고 있다. `fromService`는 의존 서비스를 인자로 받고 서비스의 구현을 반환하는 함수를 인자로 받는다. 구현 코드 내부에서는 인자로 받은 의존 서비스의 메소드를 사용하여 필요한 기능을 구현한다. 유사하게 `db.execute`는 `Task[Unit]`을 반환하고 이 메소드를 사용할 때 필요한 캡슐화를 통해 의존 관계는 숨겨져 있다.  

3. 메소드 접근자는 레이어 정의 바깥에서 메소드를 사용할 때 유용하다. 시그니처를 보면 해당 함수를 호출하기 위해 필요한 필요 조건인 서비스의 인스턴스가 무엇인지 알 수 있다. 물론 모든 의존 관계에서 필요한 것은 아니다. (역: 마지막 문장은 주어가 무엇을 가리키는지 명확히 알기가 어려워 모호하게 번역되었습니다.)

4. 타입 별칭은 우리 서비스의 의존관계를 좀 더 편리하게 표현하기 위해 사용한다. 레이어 정의 부분에서 `DB` 인스턴스에 의존하는 것을 표현한 부분에서는 패키지 오브젝트에서 정의한 타입 별칭을 사용하고 있다. `type DB = Has[DB.Service]`

이번에는 `UserModel`에 의존하는 `UserRegistration` 서비스를 보겠다. 이전 예시의 "trait + 익명 클래스 내부에서의 구현" 대신 "생성자가 있는 클래스"를 사용할 것이다.

```scala
import zio.{Task, ZIO, ZLayer}

object UserRegistration {
  // 1. service 
  class Service(notifier: UserNotifier.Service, userModel: UserModel.Service) {
    def register(u: User): Task[User] = {
      for {
        _ <- userModel.insert(u)
        _ <- notifier.notify(u, "Welcome!")
      } yield u
    }
  }

  // 2. layer
  val live: ZLayer[UserNotifier with UserModel, Nothing, UserRegistration] =
    ZLayer.fromServices[UserNotifier.Service, 
                        UserModel.Service, 
                        UserRegistration.Service](
      new Service(_, _)
    )

  // 3. accessor
  def register(u: User): ZIO[UserRegistration, Throwable, User] = 
    ZIO.accessM(_.get.register(u))
}

// ---

// 4. type alias in package.scala
type UserRegistration = Has[UserRegistration.Service]
```

[전체 예시는 GitHub](https://github.com/adamw/zioenv)에서 볼 수 있다. `UserRegistration`은 6개의 서비스를 포함한 간단한 어플리케이션이다. 위 코드에는 "ZIO environment를 이상하게 사용", "생성자 사용", "혼합된 접근법 (뒤에서 더 자세히 볼 것이다.)" 이렇게 3가지 변형이 존재한다. 의존 관계 그래프는 다음과 같이 표현된다. 

![Image for post](https://miro.medium.com/max/2268/1*z6VC1pbqXGhZ6EIPu1ZkCg.png)

전체 그림을 완성하기 위해, 다음 코드는 로우 레벨 서비스 2개의 정의이다. `DB`는 `ConnectionPool`에 의존하고 있고, `ConnectionPool`은 `DBConfig`에 의존하고 있다. `ConnectionPool`의 레이어는 이전과는 다르게 절차적인 서비스로부터 (예를 들면 자바 라이브러리) 생성하고 있다. 따라서 취득/방출 (acquire/release) 로직을 이용해 ZIO의 세상으로 "승격 (lift)"시키는 과정이 필요하다. 

```scala
import zio.{Task, ZIO, ZLayer, ZManaged}

object DB {
  // 1. service
  trait Service {
    def execute(sql: String): Task[Unit]
  }

  // 2. layer
  val liveRelationalDB: ZLayer[HasConnectionPool, Throwable, DB] = ZLayer.fromService 
    { cp => new Service {
      override def execute(sql: String): Task[Unit] =
        Task(println(s"Running: $sql, on: $cp"))
    } }
  }

// 1. procedural, low-level interface
class ConnectionPool(url: String) {
  def close(): Unit = ()
  override def toString: String = s"ConnectionPool($url)"
}

// 2. integration with ZIO
object ConnectionPoolIntegration {
  def createConnectionPool(cfg: DBConfig): ZIO[Any, Throwable, ConnectionPool] =
    ZIO.effect(new ConnectionPool(cfg.url))
  val closeConnectionPool: ConnectionPool => ZIO[Any, Nothing, Unit] = 
    (cp: ConnectionPool) => ZIO.effect(cp.close()).catchAll(_ => ZIO.unit)
  def managedConnectionPool(cfg: DBConfig): ZManaged[Any, Throwable, ConnectionPool] =
    ZManaged.make(createConnectionPool(cfg))(closeConnectionPool)

  val live: ZLayer[HasDBConfig, Throwable, HasConnectionPool] =
    ZLayer.fromServiceManaged(managedConnectionPool)
}

// ---

// 4. type aliases in package.scala
type HasDBConfig = Has[DBConfig]
type HasConnectionPool = Has[ConnectionPool]
type DB = Has[DB.Service]
```

## Putting things together

여기까지 우리는 몇가지 레이어들을 만들어보았다. 각 레이어들은 의존 관계와 결과로 나오는 구현을 정의했다. 이제 레이어들을 결합하여 아무런 의존 관계가 없는 최종 레이어를 만들어서 어플리케이션을 실행할 일만 남아있다.

우리는 레이어의 3가지 기본 연산자를 사용할 것이다. 먼저 첫 번째는 아무런 의존 관계없이 레이어를 생성하는 것이다. 이 과정은 값을 받아서 레이어를 출력하는 `ZLayer.succeed`를 이용한다. 두 번째는 `layer1 >>> layer2`이다. 이는 `layer1`의 출력을 `layer2`에 주입한다. 이는 함수 합성, 중첩된 생성자 호출과 유사하다. 세 번째는 `layer1 ++ layer2`이다. 이는 각 레이어의 입력들과 출력들을 결합시켜 새로운 레이어를 생성한다. 우리는 이 연산자들을 이용해 어플리케이션의 완전한 서비스 그래프를 생성할 수 있다.

```scala
import zio._

object Main extends App {
  override def run(args: List[String]): ZIO[zio.ZEnv, Nothing, ExitCode] = {
    // using the UserRegistration's method accessor to construct the program,
    // outside of a layer definition
    val program: ZIO[UserRegistration, Throwable, User] =
      UserRegistration.register(User("adam", "adam@hello.world"))

    // composing layers to create a DB instance
    val dbLayer: ZLayer[Any, Throwable, DB] = 
      ZLayer.succeed(DBConfig("jdbc://localhost")) >>>
      ConnectionPoolIntegration.live >>>
      DB.liveRelationalDB

    // composing layers to create a UserRegistration instance
    val userRegistrationLayer: ZLayer[Any, Throwable, UserRegistration] =
      ((dbLayer >>> UserModel.live) ++ UserNotifier.live) >>> UserRegistration.live

    // creating the complete application description
    program
      .provideLayer(userRegistrationLayer)
      .catchAll(t => ZIO.succeed(t.printStackTrace()).map(_ => ExitCode.failure))
      .map { u =>
        println(s"Registered user: $u (layers)")
        ExitCode.success
      }
  }
}
```

위 코드에서 우리는 접근자 메소드인 `UserRegistration.register`를 이용하여 우리가 실행하고 싶은 프로그램을 표현한다. (여기서는 간단한 단일 메소드 호출로 나타낸다.) 이 `program`을 실행하기 위해 `UserRegistration` 인스턴스를 필요로 한다. (주의, `UserRegistration`은 타입 별칭이다.) 이 서비스는 레이어 합성으로 생성된다. 이 과정이 끝나면 최종적으로 자기-충족(self-contained)되는 실행 가능한 어플리케이션을 표현할 수 있다. 

## There's more to layers

만약 이 모든 것들이 간단한 어플리케이션 그래프를 만들기에 너무 복잡해보인다면... 아마 당신의 생각이 맞다. 이 과정은 어렵다. 하지만 여기서 제시한 것은 기본 메커니즘을 보여주기 위한 간단한 예시이다. 따라서 이 예시만으로 최근 사용되는 접근법들이 복잡할 것이라고 판단하지 않았으면 한다.

레이어들은 이 외에도 다양한 일을 할 수 있다. 다음은 몇 가지 예시이다.

- 자동으로 레이어를 병렬로 생성할 수 있다. (가능하다면)
- 지역 단위에서 (locally) 의존 관계를 업데이트
- 레이어 생성 시 에러 관리, 재시도
- 샤딩 또는 지역 인스턴스
- 다양한 컴비네이터를 이용해 효과(effectful)가 있거나 관리가 필요한 (리소스 취득, 방출이 필요한) 인스턴스와 통합

`ZLayer` 에 대해 더 깊게 알고 싶다면 [공식 문서](https://zio.dev/docs/howto/howto_use_layers)로 시작하는 것을 추천한다. 추가적으로 Debasish Ghosh의 "Functional and Reactive Domain Modeling" 책에 나오는 다양한 효과(effect) 표현들을 이용해 구현한 [코드 예시](https://github.com/debasishg/frdomain-extras/tree/with-zio)도 좋은 접근이 될 수 있다. 마지막으로 `ZLayer`에 대한 소개로 [Pavels Sisojevs](https://scala.monster/welcome-zio/), [appddeevv](https://appddeevvmeanderings.blogspot.com/2020/05/zio-layers-and-framework-integration.html?m=1) 두 글을 추천한다.

## An alternative

대안은 무엇이 있을까? 가장 간단한 방법은 [생성자의 인자](http://di-in-scala.github.io/)를 이용하여 의존관계를 나타내고 생성자를 호출하여 오브젝트 그래프를 생성하는 것이다. 그러나 이 방법을 사용하다보면 문제에 직면하게 될 것이다. 우리는 리소스의 취득&방출 또는 효과(effectful)가 있는 초기화에 대해 수동으로 리소스를 관리해야 한다. 이러한 생성자를 이용한 의존성 주입이 장황하고 복잡한 상황에서  `ZLayer`는 그 진가를 드러낸다. 

위의 예시에서는 규모가 작아 보여주지 않았지만 `ZLayer`는 레이어 합성을 이용해 작은 부분 서비스 그래프들을 결합하여 더 큰 서비스 그래프르 생성하는 과정을 우아하게 (elegant) 수행할 수 있다. 반면 생성자를 사용하여 오브젝트 그래프를 생성할 때는 이러한 프로세스를 모듈화해야만 한다. (예: traits-as-modules를 이용)

한편 몇 가지 의존 관계가 있고 의존 서비스를 이용해 로직을 구현하는 "보통의" 서비스들에서 `ZLayer`는 과도한(overkill) 선택일 수 있다.  다음은 예시 서비스를 생성자를 이용해 구현한 예시이다. 

```scala
import zio.Task

trait DB {
  def execute(sql: String): Task[Unit]
}

class RelationalDB(cp: ConnectionPool) extends DB {
  override def execute(sql: String): Task[Unit] =
    Task {
      println(s"Running: $sql, on: $cp")
    }
}

// ---

trait UserModel {
  def insert(u: User): Task[Unit]
}

// service implementation
class DefaultUserModel(db: DB) extends UserModel {
  override def insert(u: User): Task[Unit] = 
    db.execute(s"INSERT INTO user VALUES ('${u.name}')")
}

// ---

// service (interface w/ implementation)
class UserRegistration(notifier: UserNotifier, userModel: UserModel) {
  def register(u: User): Task[User] = {
    for {
      _ <- userModel.insert(u)
      _ <- notifier.notify(u, "Welcome!")
    } yield u
  }
}
```

위에서 언급했듯이 이런 상황에서는 리소스 취득을 "수동으로" 구현해야 한다. 이 예시에서는 리소스가 하나만 존재하기 때문에 간단한 편이다. 

```scala
import zio._

object Main extends App {
  override def run(args: List[String]): ZIO[zio.ZEnv, Nothing, ExitCode] = {
    ConnectionPoolIntegration
      .managedConnectionPool(DBConfig("jdbc://localhost"))
      .use { cp =>
        lazy val db = new RelationalDB(cp)
        lazy val userModel = new DefaultUserModel(db)
        lazy val userRegistration = new UserRegistration(DefaultUserNotifier, userModel)
        userRegistration.register(User("adam", "adam@hello.world"))
      }
      .catchAll(t => ZIO.succeed(t.printStackTrace()).map(_ => ExitCode.failure))
      .map { u =>
        println(s"Registered user: $u (constructors)")
        ExitCode.success
      }
  }
}
```

일반적으로 `ZLayer`의 메커니즘을 사용하는 대신 `ZManaged`를 이용한 리소스 관리, `ZIO`를 이용한 효과(effectful)가 있는 초기화, "평범한" 서비스들의 생성자 그리고 모나딕 합성 (`for-comprehensions`를 이용)들을 이용하여 최종 오브젝트 그래프를 만들 수 있다.

##  Combining the two

생성자를 이용한 의존성 주입과 `ZLayer`를 이용한 의존성 주입은 각자 장단점을 가지고 있다. 그렇다면 자연스럽게 다음 질문으로 이어진다. 두 방법의 장점만을 결합하여 이용하는 것이 가능한가?

그 답은 이론적으로 "그렇다". 실제 상황에서는 "상황에 따라 다르다". 다음 예시를 통해 어떻게 두 방법을 모두 이용해 서비스를 구성했는지 볼 수 있다. `ConnectionPool`과 `DB`는 레이어를 이용해 관리하고 "순수한" 비즈니스 로직 컴포넌트들은 (`UserModel`, `UserNotifier`, `UserRegistration`)은 생성자를 이용해 연결하였다. 

```scala
import zio._

object Main extends App {
  override def run(args: List[String]): ZIO[zio.ZEnv, Nothing, ExitCode] = {
    // instead of a method accessor, explicitly accessing the environment
    val program: ZIO[Has[UserRegistration], Throwable, User] =
      ZIO.accessM[Has[UserRegistration]](_.get.register(User("adam", "adam@hello.com")))

    // the DB service is created using through layers (which wrap managed resources)
    val dbLayer: ZLayer[Any, Throwable, DB] = 
      ZLayer.succeed(DBConfig("jdbc://localhost")) >>>
      ConnectionPoolIntegration.live >>>
      DB.liveRelationalDB

    // the UserRegistration service graph is created using construtors
    val userRegistrationLayer: ZLayer[Any, Throwable, Has[UserRegistration]] =
      dbLayer.map { db =>
        lazy val userModel = new DefaultUserModel(db.get)
        lazy val userRegistration = new UserRegistration(DefaultUserNotifier, userModel)
        Has(userRegistration)
      }

    program
      .provideLayer(userRegistrationLayer)
      .catchAll(t => ZIO.succeed(t.printStackTrace()).map(_ => ExitCode.failure))
      .map { u =>
        println(s"Registered user: $u (layers)")
        ExitCode.success
      }
  }
}
```

이 구분을 통해 모든 서비스들 좀 더 일반적으로 모든 의존관계들은 서로 같지 않다는 사실을 알 수 있다. 어떤 의존 관계는 **상태가 있고**(stateful) 규정된 라이프 사이클이 있다. (모든 종류의 리소스들: 쓰레드, 커넥션 풀 등이 여기에 속한다.) 이런 상황에서 레이어를 사용할 때 장점이 빛을 발한다. 바로 레이어를 통한 풍부한 에러 핸들링과 리소스 관리 수용력이다.

어떤 의존 관계는 로우-레벨에 속하며 **바깥 세상과 통합**되는 경우가 있다. 

다른 서비스들은 어플리케이션의 **중심 기능 (core functinality**)을 포함하며, 로우-레벨 컴포넌트를 이용해 비즈니스 로직을 구현한다. 이러한 메소드들을 "서비스"들로 묶는 이유는 가독성, 모듈화 등의 이유가 크다.

## Summing up

당연하지만 이 방법을 더 큰 프로젝트에서 시도하는 것이 좋다. 하지만 이를 위해서는 최소한 ZIO 1.0 릴리즈를 기다려야 한다. ZIO environment를 더 큰 규모에서 사용하는 것이 가능할까? 내 생각엔 그렇다.

대부분의 경우 단일 레이어를 사용하여 부분 오브젝트 그래프를 생성하고 (여러 서비스들과 함께) 나머지는 생성자를 사용하여 두 접근을 함께 사용하는 것으로 충분하다. 접근자 메소드는 웬만하면 필요없지만 "잎(leaf)" 서비스들에서 사용하는 경우를 위해 만들 때도 있다. 타입 별칭을 만드는 것이 힘들 수도 있지만 충분히 해볼만한 수준이라고 생각한다.

그렇다면 ZIO environment의 좋은 점을 정리해보자.

- ZIO 에코시스템에서 의존 관계를 관리할 때 **통일되고 일관성있는 접근**을 제공한다.
- **효과(effectful)가 있거나 관리가 필요한 리소스**와 연관된 서비스를 마치 "평범한" 서비스들과 같이 사용할 수 있다. 특별한 할당/해제 로직이 필요하지 않다.
- 단일 레이어를 생성하거나 레이어들을 결합할 수 있는 **다양한 컴비네이터**들을 제공한다.
- 할당이 필요하거나 특정한 순서 (또는 병렬)로 실행되어야 하는 다양한 리소스 생성 로직을 에러 핸들링과 함께 간단한 코드로 표현할 수 있다.
- 부분 오브젝트 그래프를 레이어 값으로 표현할 수 있고 오브젝트 생성을 일관성있게 모듈화할 수 있다.

그렇다면 단점은 무엇일까?

- **보일러 플레이트 코드**: 매우 간단한 서비스에서도 레이어 관련한 코드들이 필요하다. 메소드 접근자는 탑-레벨에서 편리하게 서비스의 메소드를 호출하기 위해 필요하다. 타입 별칭은 코드의 가독성을 위해 사용하는 것을 추천한다.
- 모든 것을 ZIO의 세상으로 **승격**시켜야 한다. 대부분의 스칼라/자바 라이브러리는 "평범한" 값들과 생성자를 이용해 자신들의 기능을 구현한다. 이것들을 `ZIO`, `ZLayer`의 값들로 감싸서 사용해야 한다.

그렇다면 언제 레이어를 사용하고 언제 생성자를 사용해야 할까? 실전에서 경험을 쌓으면서 복합적인 사용이 어느 정도로 효과적인지 알아내야 하지만 우선적으로 적용해볼 수 있는 몇 가지 가이드 라인을 제시한다.

- ZIO environment: **로우-레벨, 효과(effectful)가 있는 의존관계** - 데이터베이스 커넥션 풀, 이메일 서비스, 메시지 브로커 인터페이스 등 바깥 세상과 직접적으로 통합되는 것.
- ZIO envrionment **부분 오브젝트 그래프** - 상대적으로 적은 의존 관계를 가진 어플리케이션의 일부분. 어플리케이션-레벨 모듈 간의 연결의 표현.
- 생성자: **비즈니스 로직** - 관리 가능하고 가독성이 높고 함수형인 부분을 캡슐화. 이러한 비즈니스 로직은 ZIO environment로 만든 로우-레벨 서비스에 의존할 수 있다. 

모듈화와 의존 관계 관리는 확실히 어려운 부분이다. 이는 많은 책, 논문, 라이브러리, 프레임워크의 주제가 된다. 이 문제를 해결하기 위해 과거에도 상당히 많은 시도가 있었음에도 여전히 새로운 접근들이 발견되고 있다. 

이러한 분야에서 ZIO가 혁신을 일으키고 있어서 좋고, 새로운 아이디어는 언제나 환영한다. 

*만약 당신이 위의 예시 코드에 대해 더 자세히 알고 싶다면 모든 코드는 [GitHub](https://github.com/adamw/zioenv)에 있다.*

## 끝

함수형 프로그래밍을 이용한 의존 관계 관리는 어떤 점이 다른지 알 수 있는 좋은 글이었습니다. (정확히는 ZIO를 이용한 방법이지만...) 처음 스프링 프레임워크를 사용할 때 의존 관계 설정으로 많이 고생했던 기억이 떠올랐습니다. 그 당시의 기억과 이 글을 비교하면서 읽으니 새삼 함수형 프로그래밍의 재미는 참 무궁무진하다는 생각이 들었습니다. 또 회사에서 사용하는 ZIO 코드들에 대한 이해도도 높아진 것 같습니다. 여러모로 저에게 많은 도움이 되는 번역이었습니다.

번역이 어색하거나 틀린 정보가 있거나 이해가 안되는 점들은 댓글이나 이메일 주시면 정말 감사드리겠습니다 :)


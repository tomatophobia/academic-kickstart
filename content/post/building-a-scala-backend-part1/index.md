---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "스칼라로 백엔드 서버 구축하기 1편"
subtitle: ""
summary: ""
authors: [Youngseo Choi]
tags: [Functional Programming, Scala, Backend]
categories: [Functional Programming]
date: 2021-05-23T23:48:19+09:00
lastmod: 2021-05-23T23:48:19+09:00
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

스칼라 백엔드 개발자이지만 코드베이스 없이 맨 땅에서 스칼라로 간단한 백엔드 서버를 구축해보라고 하면 부끄럽지만 여전히 어려움을 느낍니다. 이 글과 함께 스칼라를 이용한 웹 서버 구축 과정을 확실하게 익혀보려 합니다.

## 큰 그림

우선 간단히 어떤 라이브러리와 기술 스택을 사용할지 간단히 정리해보겠습니다.

일반적으로 스칼를 이용한 백엔드 서버라고 하면 [Play Framework](https://www.playframework.com)를 이용하는 경우가 많습니다. 분명 좋은 프레임워크이지만 현재 제 업무에서 사용하고 있지는 않습니다. 이 프로젝트에서는 현재 업무에서 애매한 지식 수준으로 사용하고 있는 라이브러리들에 대한 학습이 목표 중 하나이므로 실제로 사용 중인 라이브러리를 사용하려 합니다.

간단히 나열해보면 다음과 같습니다.

### [ZIO](https://zio.dev)

ZIO는 순수 함수형 프로그래밍을 기반으로 비동기(async), 동시성(concurrent) 프로그래밍에 사용하는 라이브러리입니다. ZIO를 정확하고 자세히 설명하기에는 제 식견이 부족하여 어려울 것 같습니다. 다만 제 생각에 가장 중요한 부분은 부수효과(side effect)를 순수한 데이터 타입으로 표현한다는 점입니다. 이를 [함수형 효과(functional effect)](https://scalac.io/blog/introduction-to-programming-with-zio-functional-effects/)라고 합니다. 이에 대해서는 제 글 중 [함수형 디자인 소개](https://easywritten.org/post/an-introduction-to-functional-design-translation/)에서도 소개되어 있으니 참고하시기 바랍니다.

현재 제가 사용하는 라이브러리들 중 가장 핵심적인 라이브러리가 될 것으로 예상됩니다. 현재 스칼라 라이브러리들 중 인기와 주목도가 매우 높으며 함수형 프로그래밍에 관해서 이론적인 배경이 많지 않아도 쉽게 접근할 수 있도록 도와주어 상당히 유용하게 사용하고 있습니다.

### [http4s](https://http4s.org)

마찬가지로 순수 함수형 프로그래밍 기반으로 작동하는 웹 프레임워크의 일종입니다.

사실 스칼라를 이용해 순수 함수형 프로그래밍과 함께 웹 개발을 할 때 항상 나오는 프레임워크 중 하나입니다. 그런데 문제는 개념이 너무 어려워서 지금까지 다른 코드베이스를 보고 가져다 쓰기만 할 뿐 정확한 동작 원리에 대해 이해하고 있지 못하였습니다. 이번에는 꼭...

스칼라로 순수 함수형 프로그래밍을 할 때 가장 많이 쓰이는 라이브러리인 [cats](https://typelevel.org/cats/)를 이용하는데 (정확히는 [cats-effect](https://github.com/typelevel/cats-effect)) 여기서 막혀서 과거에 실패했던 기억이 있습니다.

### [tapir](https://github.com/softwaremill/tapir)

HTTP API 엔드포인트를 구성할 때 사용하는 라이브러리입니다.

http4s와 비슷한 포지션을 취하고 있는 것으로 알고있습니다. CTO님의 표현으로는 tapir의 백엔드로 http4s를 사용할 수 있다고 하셨는데 정확히 알아들었는지는 모르겠네요. (ㅎ) 이번 기회를 통해 두 라이브러리가 정확히 어떤 포지션을 담당하는지 구별해보겠습니다.

### [Akka](https://akka.io)

분산시스템을 구성할 때 사용하는 라이브러리입니다. 매우 다양한 곳에 사용되고 스칼라 프로젝트 중에 아마 Spark 다음으로 유명하지 않을까 생각합니다.

과거 액터에 대해 공부할 때 간단하게 사용한 경험만 있습니다. 사실 이번 서버 개발에 어떻게 쓰일지 잘 모르나 꼭 한번 짚고 넘어가야 할 것 같아서 어떻게든 사용해보려 합니다.

참고적으로 현재 제 업무에서는 Akka-cluster를 이용해 AWS ECS에서 여러 개의 서버 노드를 구동시키고 있습니다. 이 역시 자세한 설정을 해본 적이 없어 이번 기회를 통해 정확히 알고 가고 싶습니다.

### [doobie](https://tpolecat.github.io/doobie/)

doobie는 스칼라와 [cats](https://typelevel.org/cats/)를 이용한 순수 함수형 JDBC 레이어입니다.

마찬가지로 과거에 잘 사용해보고 싶었지만 cats의 벽에 막혀 정확히 이해하지 못하고 넘어간 라이브러리 중 하나입니다. 아마 postgresql과 함께 사용하게 될 것 같습니다.

### 그 외

위에서 언급한 라이브러리 외에도 실제 업무에서 사용하는 라이브러리들을 최대한 활용해 볼 예정입니다 모두 어렴풋이 기능을 알고 있고 정확하게 어떤 역할을 하는지 구멍난 부분이 많습니다.

정리하면 다음과 같은 라이브러리들을 추가적으로 사용할 것 같습니다.

[shpeless](https://github.com/milessabin/shapeless), [fs2](https://github.com/typelevel/fs2), [cats](https://typelevel.org/cats/), [monocle](https://github.com/optics-dev/Monocle), [sttp](https://sttp.softwaremill.com/en/latest/), [tsec](https://github.com/jmcardon/tsec), [enumeratum](https://github.com/lloydmeta/enumeratum), [config](https://github.com/lightbend/config), [caliban](https://github.com/ghostdogpr/caliban)

---

스칼라를 좋아하고 함수형 프로그래밍을 좋아한다고 말하지만 실상 정확한 동작 원리나 역할을 모른 채 사용한 라이브러리가 너무 많은 것이 마음 속 응어리 중 하나였습니다. 이번 글을 통해 제가 몰랐던 것을 점검하고 더 나은 개발자가 되는 기회가 된다면 좋을 것 같습니다.

---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Wrapping impure code with ZIO [번역]"
subtitle: ""
summary: ""
authors: []
tags: [Functional Programming, Scala, ZIO]
categories: [Functional Programming]
date: 2020-08-23T23:37:39+09:00
lastmod: 2020-08-23T23:37:39+09:00
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

이 글은 [Pierre Ricadat](https://medium.com/@ghostdogpr) 의 [Wrapping impure code with ZIO](https://medium.com/@ghostdogpr/wrapping-impure-code-with-zio-9265c219e2e) 를 번역한 글입니다. 

---

만약 당신이 스칼라로 함수형 프로그래밍을 하고 있다면 상당히 헤맬 가능성이 높다. 우리가 다루는 코드가 함수형 프로그래밍의 기본 원리인 총체성 (totality), 결정성 (determinism), 순수성 (Purity)을 지키지 않을 수 있기 때문이다. 쉽진 않겠지만 다음을 상기하면서 이 문제를 해결해보자.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">FP is just programming with functions. Functions are:<br><br>1. Total: They return an output for every input.<br>2. Deterministic: They return the same output for the same input.<br>3. Pure: Their only effect is computing the output.<br><br>The rest is just composition you can learn over time.</p>&mdash; John A De Goes (@jdegoes) <a href="https://twitter.com/jdegoes/status/936301872066977792?ref_src=twsrc%5Etfw">November 30, 2017</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

> 함수형 프로그래밍은 함수로 프로그래밍을 하는 것일 뿐이다. 여기서 함수란:
>
> 1. 총체성: 모든 입력에 대해 출력을 반환한다.
> 2. 결정성: 항상 같은 입력에 대해 같은 출력을 반환한다.
> 3. 순수성: 출력을 계산하는 것 외에 효과가 발생하지 않는다.
>
> 이 외에 나머지는 앞으로 너가 배워나갈 것들의 조합일 뿐이다.

JVM 에코시스템은 매우 거대해서 프로그래머가 원하는 거의 모든 라이브러리를 찾을 수 있다. 하지만 그 중 대부분은 심지어 스칼라를 포함해서도 위에서 언급한 원칙을 지키지 않는다. 당신 또한 함수형이 아닌 레거시 코드를 가지고 있을 수 있다. 늦든 빠르든 이런 비함수형 코드들을 처리해야 할 때가 올 것이다. 이 글을 통해 나는 다양한 순수하지 않은 코드들을 어떻게 ZIO 라이브러리를 이용해 처리하는지 이야기할 것이다. 

앞으로 나올 예시들은 다양한 스칼라 또는 자바 코드에 종속성을 가진 실제 라이브러리 코드들을 (`zio-sqs`, `zio-akka-cluster`) 통해 설명한다. 따라서 이 글에서는 당신이 ZIO에 대해 배경지식이 있다고 가정하고 설명한다. 만약 ZIO를 처음 접한다면 [공식 문서](https://zio.dev/) 또는 [리소스](https://zio.dev/docs/resources/resources)들이 도움이 될 것 이다. 

![Image for post](https://miro.medium.com/max/600/1*lM5WtEonS2vERGNJUpYlqg.jpeg)

{{< figure src="https://miro.medium.com/max/600/1*lM5WtEonS2vERGNJUpYlqg.jpeg" caption="A unicorn --- **green** unicorn!" >}}




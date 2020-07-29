---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "직접 사용하면서 느낀 함수형 프로그래밍의 장점"
subtitle: ""
summary: ""
authors: []
tags: [Functional Programming]
categories: [Functional Programming]
date: 2020-07-29T21:33:55+09:00
lastmod: 2020-07-29T21:33:55+09:00
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

# 함수형 프로그래밍 진짜 좋은가?

 함수형 프로그래밍에 대한 장점을 인터넷에 검색하면 나오는 내용은 *"부작용(side effect)이 없다", "디버깅이 쉽다", "테스트하기 쉽다"* 등의 내용을 볼 수 있습니다. 하지만 이러한 장점을 느끼기 위해서 *1급 객체(First Class Object), 고차 함수(Higher-Order Function), 불변성(Immutabiliy)* 등 알아야 할 개념이 많고, 기존 명령형 프로그래밍 언어(C, Java)를 사용하던 사람이라면 순수 함수를 쓰는 것 자체가 불편하게 느껴질 것입니다. 저 또한 함수형 프로그래밍을 처음 시작할 때 장점이 많다는 소문에 비해 실속이 없다는 생각을 많이 했었던 기억이 있었습니다. 하지만 꾸준히 함수형 프로그래밍에 대해 공부하고 재미를 느끼게 되면서 비로소 몇 가지 장점이라고 할 만한 부분을 알게 되었습니다. 동시에 그 장점이 위에서 말한 알아야 할 지식이 없어도 충분이 느낄 수 있다는 것을 알 수 있었습니다.

 이 글에서는 간단한 행렬 회전 예제를 통해서 함수형 프로그래밍이 기존 명령형 프로그래밍과 비교하여 어떤 차별점을 갖는지 설명합니다. 여기서부터 장점이 아닌 차별점이라고 표현하는데 그 이유는 이것들이 누군가에게는 장점으로 느껴지지 않을 수 있기 때문입니다. 

# 두 가지 패러다임으로 예제 구현하기

 간단한 행렬(2차원 배열) 회전 문제를 생각해봅시다. 우리에게 행렬이 하나 주어져 있고 이를 **반시계방향**으로 회전한 결과 행렬을 반환하는 프로그램을 각각 명령형 프로그래밍, 함수형 프로그래밍으로 나눠서 작성해보겠습니다. 언어는 둘 다 자바스크립트를 사용하였습니다. (브라우저의 개발자 도구 켜시면 돌려보실 수 있도록 자바스크립트로 작성하였습니다.  ^^7)

## 명령형 프로그래밍으로 행렬(2차원 배열) 회전

```javascript
const input = [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10, 11, 12]];
```

> const는 제 취향일 뿐 var을 사용해도 무방합니다. 

먼저 2차원 배열을 반시계 방향으로 회전해야 한다는 문제를 받았을 때, 기존에 명령형 프로그래밍 언어(C, Java 등)로 프로그래밍을 하신 분들이라면 2차원 배열이라는 단어를 보자마자 반복문을 두 번 사용해야겠다는 생각이 **매우 빠르게** 들 것입니다. 

```javascript
const input = [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10, 11, 12]];

const m = input.length // 가로 길이 m행
const n = input(0).length // 세로 길이 n행

var output = new Array(m).fill(0).map(() => new Array(n).fill(null)) // null로 채워진 결과 행렬

for (i = 0; i < m; i++) {
  for (j = 0; j < n; j++) {
    // something happen
  }
}
```

그렇다면 저 중첩 반복문 내부에는 어떤 코드가 들어가야 할까요? 중첩 반복문 내부에서는 회전하기 전인 input 배열의 **내부 요소**를 하나씩 뽑아서 결과 배열의 **내부 요소**를 하나씩 채워나가야합니다. 

```javascript
output[x][y] = input[i][j] // x = ???, y = ???
```

또한 여기서 input의 각 요소가 output의 요소에 대응되는 규칙이 있기 때문에 x, y는 i, j로 표현된다는 것을 알 수 있습니다. 따라서 이를 통해 2개의 방정식으로 얻을 수 있습니다. 그리고 a, b, c, d, e, f가 우리가 구해야 할 값이 될 것입니다.

- x = ai + bj + c
- y = di + ej + f

여기까지 이해가 되셨나요? 조금 어렵게 표현한 것 같지만 input 행렬 i행 j열의 요소가 output 행렬의 어떤 위치로 가는지 알아내야 한다는 의미입니다. 

다행히 우리에게는 한 가지 힌트가 더 있습니다. 각 행렬의 꼭지점의 위치 이동은 짐작하기가 쉽다는 것입니다. 예를 들어 input 행렬 1행 1열의 값(왼쪽 위 꼭지점)은 output 행렬 n행 1열이 될 것입니다. 즉 (i, j) = (0, 0)  일 때 (x, y) = (n-1, 0) 로 정리해볼 수 있겠습니다. (1~n 행을 코드에서는 편의상 0~n-1행으로 표현하였습니다.) 

그럼 이러한 꼭지점 4개를 위의 연립방정식에 대입하면 a, b, c, d, e, f 값을 얻을 수 있습니다. 값을 대입하여 나온 x, y 입니다.

- x =  n - 1 - j
- y = i

이를 통해 명령형 프로그래밍으로 행렬 반시계방향 회전을 끝냈습니다.

```javascript
const input = [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10, 11, 12]];

const m = input.length // 가로 길이 m행
const n = input[0].length // 세로 길이 n행

var output = [] // 결과 행렬

for (i = 0; i < m; i++) {
  for (j = 0; j < n; j++) {
    output[n-1-j][i] = input[i][j] 
  }
}

```

## 함수형 프로그래밍으로 행렬(2차원 배열) 회전


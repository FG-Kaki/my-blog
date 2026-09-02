---
layout: post
title: "같은 태그에 색이 왜 이렇게 나올까 - CSS 기본 선택자 구분하기"
date: 2026-09-02 09:00:00 +0900
categories: [CSS]
tags: [css, selector, specificity]
mermaid: true
---

## 들어가며 (Situation)

HTML 태그를 어느 정도 익히고, 오늘부터는 CSS로 넘어갔다. 첫 시간은 CSS의 가장 기본인 **선택자(selector)** - 어떤 요소에 스타일을 적용할지 "고르는" 문법이었다.

## 문제 상황 (Task)

전체 선택자(`*`), 태그 선택자, 아이디 선택자(`#`), 클래스 선택자(`.`) 네 가지 모두 "요소를 골라서 스타일을 준다"는 개념 자체는 금방 이해했다. 문제는 실습 예제 하나에 이 네 가지가 전부 섞여 있었다는 점이다. 특히 아래 `<li>` 하나에는 네 가지 선택자가 **동시에** 적용되고 있었는데, 최종적으로 어떤 색이 보일지, 그리고 왜 그 색이 이기는지 구분이 잘 안 갔다.

```html
<li class="class3" id="id3">클래스 테스트3</li>
```

```css
* { color: red; }
li { color: pink; }
#id3 { color: magenta; background-color: yellow; }
.class3 { color: blue; background-color: yellow; }
```

이 한 줄짜리 `<li>`에 `*`, `li`, `#id3`, `.class3` 규칙이 전부 걸린다. 눈으로 결과 색만 보고 넘어가면 "그렇구나" 하고 끝나버릴 것 같아서, **왜 그 색이 이기는지**를 짚고 넘어가기로 했다.

## 해결 과정 (Action)

### 1. 네 가지 기본 선택자

| 선택자 | 문법 | 선택 대상 |
|---|---|---|
| 전체 선택자 | `*` | 문서 안의 모든 요소 |
| 태그 선택자 | `태그명` | 해당 태그를 쓴 모든 요소 |
| 아이디 선택자 | `#id값` | 그 id를 가진 요소 (한 페이지에 하나만 있어야 함) |
| 클래스 선택자 | `.class값` | 그 class를 가진 모든 요소 |

```html
<ul>
  <li>테스트1</li>
  <li>테스트2</li>
  <li>테스트3</li>
</ul>
```

```css
li { color: pink; }
```

여기까지는 "이 태그를 쓴 애들 전부"라서 직관적이었다.

### 2. 여러 선택자가 겹치면? - 명시도(Specificity)

문제의 `<li class="class3" id="id3">`로 돌아오면, 4개 규칙이 전부 이 요소에 적용된다. 이때 브라우저는 **명시도(specificity)**가 더 높은 규칙을 최종적으로 적용한다. 명시도는 대략 이런 순서다.

```mermaid
flowchart LR
    A["전체 선택자 *<br/>명시도 최하"] --> B["태그 선택자 li"]
    B --> C["클래스 선택자 .class3"]
    C --> D["아이디 선택자 #id3<br/>명시도 최상"]
```

즉 `#id3`가 `.class3`를 이기고, `.class3`는 `li`를 이기고, `li`는 `*`를 이긴다. 그래서 `<li class="class3" id="id3">`는 네 규칙이 전부 걸려 있어도 **아이디 선택자가 준 `magenta`**로 보인다. 다만 `background-color`는 `#id3`와 `.class3` 둘 다 똑같이 `yellow`를 줬기 때문에 어느 쪽이 이기든 결과는 같아서, 배경색만 보고는 명시도 차이가 눈에 띄지 않았던 것도 헷갈림의 원인 중 하나였다.

### 3. 클래스 선택자는 여러 요소에, 아이디는 하나에

```html
<ul>
  <li class="class1">클래스 테스트1</li>
  <li class="class2">클래스 테스트2</li>
  <li class="class3" id="id3">클래스 테스트3</li>
  <li class="class4">클래스 테스트4</li>
  <li class="class5">클래스 테스트5</li>
</ul>
```

클래스는 `class1`~`class5`처럼 **같은 이름을 여러 요소에 재사용**하도록 설계된 반면, 아이디는 원칙적으로 페이지 안에서 유일해야 한다는 차이도 이 예제로 명확해졌다. "여러 개에 공통 스타일을 주고 싶으면 클래스, 딱 하나만 정확히 짚고 싶으면 아이디"로 정리하니 언제 뭘 써야 할지가 더는 헷갈리지 않았다.

## 결과 (Result)

| 항목 | 이전 | 이후 |
|---|---|---|
| 여러 선택자가 겹칠 때 | 화면에 보이는 색을 그냥 결과로만 받아들임 | 명시도 순서(전체 < 태그 < 클래스 < 아이디)로 왜 그 색이 이기는지 설명 가능 |
| 클래스 vs 아이디 | 스타일 주는 방법이라는 것만 앎 | "재사용은 클래스, 유일한 대상은 아이디"로 용도 구분 |

`<li class="class3" id="id3">`처럼 여러 선택자가 겹치는 요소를 만나도, 이제는 최종 색이 왜 그렇게 나오는지 명시도를 기준으로 따라갈 수 있게 됐다.

## 더 학습하면 좋은 개념

- **CSS 명시도 계산 공식** - 인라인 스타일, id, class/속성/가상클래스, 태그/가상요소를 각각 점수로 환산해서 비교하는 정확한 규칙. 오늘은 "대략적인 순서"만 익혔는데, 인라인 스타일이나 `!important`까지 들어가면 이 공식으로 정확히 계산해야 한다.
- **`!important`와 소스 순서(source order)** - 명시도가 완전히 같을 때는 CSS 파일에서 나중에 선언된 규칙이 이긴다. `!important`는 이 우선순위 규칙 자체를 뒤엎는 예외라 남용하면 안 되는 이유도 함께 알아두면 좋다.
- **CSS 상속(inheritance)** - 선택자에 직접 걸리지 않은 요소는 부모의 스타일을 상속받기도 한다. 명시도와 상속을 같이 이해해야 "왜 이 요소가 이 색으로 보이는지"를 완전히 설명할 수 있다.

## 참고 자료
- [MDN - CSS selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_selectors)
- [MDN - Universal selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/Universal_selectors)
- [MDN - ID selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/ID_selectors)
- [MDN - Class selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/Class_selectors)
- [MDN - Specificity](https://developer.mozilla.org/en-US/docs/Web/CSS/Specificity)

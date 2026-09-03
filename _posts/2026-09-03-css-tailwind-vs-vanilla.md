---
layout: post
title: "직접 짠 hover 애니메이션을 유틸리티 클래스로 옮기면 - Tailwind와 바닐라 CSS 비교하기"
date: 2026-09-03 10:00:00 +0900
categories: [CSS]
tags: [css, tailwindcss, bootstrap, utility-first]
mermaid: true
---

## 들어가며 (Situation)

앞서 Flex로 배치한 상품 카드에 마우스오버 애니메이션과 배지 깜빡임 효과를 순수 CSS로 직접 짜본 뒤, 같은 결과를 **Tailwind의 유틸리티 클래스**로는 어떻게 표현하는지 클래스 단위로 대응시켜보는 실습을 했다. 더불어 Bootstrap을 CDN으로 끌어와 프리셋 컴포넌트를 써보는 실습도 같이 진행했다.

## 문제 상황 (Task)

바닐라 CSS 버전은 카드에 마우스를 올리면 위로 떠오르며 커지고 그림자가 생기는 효과, 그리고 `NEW` 배지가 기울어진 채 색이 깜빡이는 애니메이션까지 완성돼 있었다. 이걸 "React Query를 썼다"류로 나열하지 않으려면, 왜 그렇게 짰는지와 같은 결과를 다른 방식(Tailwind)으로는 어떻게 표현하는지를 실제 코드로 비교해야 했다.

## 해결 과정 (Action)

### 1. 바닐라 CSS로 만든 hover 애니메이션

```css
.card {
  background-color: #FFFFFF;
  border: 1px solid #E5E7EB;
  border-radius: 8px;
  padding: 16px;
  /* transition은 :hover가 아니라 "원래 규칙"에 쓴다.
     여기 써야 마우스를 뗄 때도 부드럽게 돌아온다 */
  transition: all 0.3s;
}

/* .card:hover 는 붙여 쓴다.
   공백을 넣으면(.card :hover) 카드 "안쪽" 요소를 가리키는 완전히 다른 뜻이 된다 */
.card:hover {
  background-color: #F9FAFB;
  transform: translate(0, -8px) scale(1.05);
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.12);
}
```

실습 코드에 남아 있던 두 코멘트가 핵심이었다.

- `transition`은 `:hover` 규칙이 아니라 **기본 상태인 `.card`**에 걸어야 한다. `:hover`에만 걸면 마우스를 올릴 때는 부드럽게 움직이지만, 뗄 때는 원래 상태로 뚝 끊기듯 돌아간다.
- `.card:hover`(붙여쓰기)와 `.card :hover`(공백 포함)는 완전히 다른 선택자다. 뒤엣것은 "`.card` **안에 있는** 어떤 요소의 hover 상태"를 가리키는 자손 선택자가 된다.

배지 깜빡임은 `@keyframes`로 만들었다.

```css
.badge {
  transform: rotate(-5deg);
  animation-name: blink;
  animation-duration: 1.2s;
  animation-iteration-count: infinite;
}

@keyframes blink {
  0%   { background-color: #DC2626; }
  50%  { background-color: #F97316; }
  100% { background-color: #DC2626; }
}
```

0%와 100%를 같은 색으로 맞춰야 무한 반복될 때 색이 끊기지 않고 이어진다는 점도 실습으로 확인했다.

### 2. 같은 효과를 Tailwind 유틸리티 클래스로 대응시켜보기

실습 파일에는 Tailwind 클래스로 옮긴 버전이 주석으로 함께 남아 있었다. 속성 하나하나를 클래스 이름과 짝지어 보면 이렇다.

| 바닐라 CSS | Tailwind 유틸리티 클래스 | 의미 |
|---|---|---|
| `transition: all 0.3s` | `transition` | 상태 변화를 부드럽게 |
| `.card:hover { background-color: #F9FAFB }` | `hover:bg-gray-50` | hover 시 배경색 변경 |
| `.card:hover { transform: translate(0,-8px) }` | `hover:-translate-y-2` | hover 시 위로 이동 |
| `.card:hover { transform: scale(1.05) }` | `hover:scale-105` | hover 시 확대 |
| `.card:hover { box-shadow: ... }` | `hover:shadow-lg` | hover 시 그림자 |
| `border-radius: 8px` | `rounded-lg` | 모서리 둥글게 |
| `font-size: 24px; font-weight: bold` | `text-2xl font-bold` | 제목 크기·굵기 |

바닐라 CSS는 `.card:hover`라는 **선택자 하나**에 여러 속성을 몰아 쓰는 방식이고, Tailwind는 속성 하나당 클래스 하나(`hover:bg-gray-50`, `hover:-translate-y-2`, `hover:scale-105`, `hover:shadow-lg`...)를 HTML에 나열하는 방식이다. 로직은 동일하지만 "어디에 규칙이 적히는가"가 다르다 - CSS 파일 vs HTML 클래스 속성.

배지의 `blink` 애니메이션처럼 **직접 정의한 `@keyframes`**는 Tailwind 기본 유틸리티(`animate-pulse`, `animate-bounce` 등)만으로는 그대로 재현되지 않는다. 색상이 두 단계로 오가는 지금 같은 애니메이션을 쓰려면 `tailwind.config.js`에 커스텀 keyframe을 직접 등록해야 한다는 점도 이번에 짚었다.

### 3. Bootstrap - CDN으로 프리셋 프레임워크 가져오기

```html
<script src="https://cdn.jsdelivr.net/npm/@popperjs/core@2.11.8/dist/umd/popper.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.8/dist/js/bootstrap.min.js"></script>
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.8/dist/css/bootstrap.min.css" rel="stylesheet">
```

```html
<button type="button" class="btn btn-success">Success</button>
```

Tailwind가 속성 하나당 클래스 하나를 조합하는 **유틸리티 우선(utility-first)** 방식이라면, Bootstrap은 `btn btn-success`처럼 이미 디자인이 끝난 **컴포넌트**를 클래스 이름으로 가져다 쓰는 방식이다. 둘 다 "CDN으로 CSS 프레임워크를 가져온다"는 점은 같지만, 실제로 코드를 짜는 방식은 서로 다르다는 걸 CDN 스크립트를 직접 연결해보며 체감했다.

| 항목 | Tailwind | Bootstrap |
|---|---|---|
| 접근 방식 | 유틸리티 클래스 조합 (utility-first) | 완성된 컴포넌트 클래스 (component-based) |
| 커스터마이징 | 클래스 조합만으로 세밀하게 조정 | 컴포넌트 기본 스타일을 덮어써야 함 |
| CDN 적용 방식 | Play CDN 스크립트(`cdn.tailwindcss.com`)를 불러와야 클래스가 실제로 해석됨 | CSS/JS 파일을 링크·스크립트로 연결하면 바로 적용 |

## 결과 (Result)

| 항목 | 바닐라 CSS | Tailwind |
|---|---|---|
| hover 효과 구현 | 선택자 1개(`.card:hover`) 안에 속성 3개 | 유틸리티 클래스 3~4개를 HTML에 나열 |
| 재사용성 | 클래스 이름을 새로 지어야 재사용 가능 | 클래스 조합만 복사하면 바로 재사용 |
| 세밀한 커스텀(예: `blink` 애니메이션) | `@keyframes`로 자유롭게 정의 | 기본 유틸리티로는 불가, config 확장 필요 |

같은 hover·애니메이션 효과를 두 방식으로 나란히 짚어보니, "무엇을 할 수 있는가"는 거의 동일하지만 "어디에 규칙을 적는가"와 "얼마나 세밀하게 손댈 수 있는가"에서 차이가 갈린다는 걸 확인했다.

## 더 학습하면 좋은 개념

- **Tailwind Play CDN과 빌드 파이프라인의 차이** - `cdn.tailwindcss.com` 스크립트는 학습·프로토타입용이고, 실무에서는 빌드 과정(PostCSS)을 거쳐 실제 쓰인 클래스만 남기는 최적화를 거친다. 왜 실무에서 CDN 방식을 그대로 쓰지 않는지 이해할 수 있다.
- **`tailwind.config.js`로 커스텀 keyframe 등록하기** - 오늘의 `blink`처럼 기본 유틸리티에 없는 애니메이션을 쓰려면 반드시 필요한 설정이다.
- **유틸리티 클래스와 명시도의 관계** - 유틸리티 클래스는 대개 단일 속성만 담당하기 때문에 `!important` 없이도 충돌이 적다. 앞서 다룬 [[css-specificity]] 개념과 이어서 보면 왜 그런지 설명이 된다.
- **컴포넌트 기반 프레임워크(Bootstrap)와 유틸리티 우선 프레임워크(Tailwind)의 설계 철학 차이** - 왜 최근 신규 프로젝트에서 유틸리티 우선 방식을 더 많이 채택하는지 배경을 알아두면 프레임워크 선택 기준이 명확해진다.

## 참고 자료
- [Tailwind CSS 공식 문서 - Play CDN](https://tailwindcss.com/docs/installation/play-cdn)
- [Tailwind CSS 공식 문서 - Hover, Focus, and Other States](https://tailwindcss.com/docs/hover-focus-and-other-states)
- [Tailwind CSS 공식 문서 - Adding Custom Animations](https://tailwindcss.com/docs/animation#customizing-your-theme)
- [Bootstrap 공식 문서 - CDN 링크](https://getbootstrap.com/docs/5.3/getting-started/introduction/#cdn-links)
- [MDN - CSS Transitions](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_transitions/Using_CSS_transitions)
- [MDN - CSS Animations](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_animations/Using_CSS_animations)

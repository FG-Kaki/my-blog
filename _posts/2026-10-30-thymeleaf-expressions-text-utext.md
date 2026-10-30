---
layout: post
title: "${}, *{}, #{}, @{}는 각각 무엇을 꺼낼까 - Thymeleaf 표현식과 th:text·th:utext"
date: 2026-10-30 09:20:00 +0900
categories: [Spring]
tags: [spring, thymeleaf, template-engine, xss, inline]
mermaid: true
---

## 들어가며 (Situation)

Spring MVC 수업 내내 `th:text="${message}"` 한 줄로 값을 화면에 출력해 왔습니다. 이번 실습(`chap07-thymeleaf`)은 템플릿 엔진 **Thymeleaf** 자체를 정리합니다. 이 글은 값을 꺼내는 **표현식**과 출력 방식, 다음 글은 제어문·반복·공통 영역 처리를 다룹니다.

> 실제 서비스에서 겪은 문제가 아니라, 수업 실습을 정리한 글입니다. Spring Boot 4.1.2, Thymeleaf 3.1 계열을 사용했고, 동작 설명은 Thymeleaf 공식 문서로 보완했습니다.

## 문제 상황 (Task)

- Model 값, 요청 파라미터, 세션 값을 화면에서 각각 어떻게 꺼낼까요?
- 같은 객체의 필드를 여러 번 출력할 때 `${member.xxx}`를 반복해야 할까요?
- 서버가 보낸 문자열에 HTML 태그가 들어 있으면 태그로 그려야 할까요, 글자로 보여야 할까요?
- 서버 값을 화면의 JavaScript 변수로 넘기려면 어떻게 할까요?

## 해결 과정 (Action)

### 1. 준비: 네임스페이스와 설정

```html
<html xmlns:th="http://www.thymeleaf.org">
```

`xmlns:th`는 `th:` 속성을 쓰기 위한 네임스페이스 선언입니다. 수업 주석대로 `th` 문법을 쓰지 않는 일반 HTML에는 필요 없습니다.

설정 파일의 주석은 Spring Boot의 기본값을 정리해 두었습니다.

```yaml
# 아래 값은 Spring Boot 기본값이므로 작성하지 않아도 동일하게 동작한다.
# "lecture/expression" -> classpath:/templates/lecture/expression.html
#spring.thymeleaf.prefix=classpath:/templates/
#spring.thymeleaf.suffix=.html
```

수업 코드는 뷰 이름 앞에 `/`를 붙이지 않습니다. 주석의 설명에 따르면 `"/lecture/expression"`처럼 쓰면 경로가 `templates//lecture/...`가 되어 **jar로 배포했을 때 템플릿을 못 찾을 수 있기** 때문입니다. 개발 환경에서는 동작하다가 배포 후에만 실패하는 종류의 문제라 습관으로 들여 둘 만합니다.

이 프로젝트는 실행 클래스가 `config` 패키지 안에 있어서, 별도 설정 클래스에서 스캔 범위를 상위 패키지로 지정했습니다. Spring Core 수업에서 정리한 "`@ComponentScan`은 해당 패키지와 그 하위만 스캔한다"는 규칙이 그대로 적용된 예입니다.

```java
// @SpringBootApplication 은 자신이 위치한 패키지(com.wanted.thymeleaf.config)와 그 하위만 스캔한다.
@Configuration
@ComponentScan(basePackages = "com.wanted.thymeleaf")
public class ContextConfiguration { }
```

### 2. 표현식 종류

| 표현식 | 이름 | 용도 |
|--------|------|------|
| `${...}` | 변수 표현식 | Model 값, 요청 파라미터, 세션 값 |
| `*{...}` | 선택 변수 표현식 | `th:object`로 선택한 객체의 필드 |
| `#{...}` | 메시지 표현식 | `messages.properties`의 문구 |
| `@{...}` | 링크 표현식 | URL 생성 (context path 자동 처리) |
| `~{...}` | 조각 표현식 | 다른 템플릿의 일부 (다음 글) |
| `[[...]]`, `[(...)]` | 인라인 표현식 | 태그 속성이 아닌 본문에 값 삽입 |

### 3. 변수 표현식 ${...}: 어디서 꺼내느냐에 따라 다르다

| 꺼낼 값 | 작성 방법 | 서블릿 수업의 대응 |
|---------|-----------|-------------------|
| Model / ModelAndView 값 | `${ 이름 }` | `req.getAttribute` |
| 요청 파라미터 | `${ param.이름 }` | `req.getParameter` |
| 세션 속성 | `${ session.이름 }` | `session.getAttribute` |

```html
<!-- /lecture/expression?title=표현식&no=5&no=6 으로 접근 -->
<p th:text="${ param.title }"></p>
<!-- 같은 이름의 파라미터가 여러 개면 배열처럼 인덱스로 꺼낸다. -->
<p th:text="${ param.no[0] }"></p>
<p th:text="${ param.no[1] }"></p>
```

서블릿 글에서 체크박스처럼 같은 이름의 값이 여러 개일 때 `getParameterValues`를 썼던 것을, Thymeleaf에서는 인덱스로 꺼냅니다. 수업 주석이 짚은 주의점도 있습니다. **존재하지 않는 인덱스에 접근하면 무시되지 않고 에러가 납니다.** 이 페이지는 `no` 파라미터가 반드시 있어야 하고, 주소창에 `/lecture/expression`만 입력하면 `param.no`가 `null`이라 500 에러가 납니다.

또 하나, Thymeleaf 3.1부터는 보안상 `${#request}`, `${#response}` 같은 서블릿 객체에 템플릿에서 직접 접근할 수 없습니다(수업 주석, Thymeleaf 3.1 문서 기준). 필요한 값은 컨트롤러에서 Model에 담아 넘깁니다.

### 4. 선택 변수 표현식 *{...}

```java
mv.addObject("member", new MemberDTO("홍길동", 20, '남', "서울시 서초구"));
```

```html
<p th:text="${ member.name }"></p>

<!-- th:object 로 객체를 선택해 두면 *{필드명} 으로 바로 꺼낸다 -->
<div th:object="${ member }">
    <p th:text="*{ age }"></p>
    <p th:text="*{ address }"></p>
</div>
```

`th:object`로 객체를 한 번 지정하면, 그 태그 안에서는 `*{age}`처럼 객체 이름 없이 필드를 꺼낼 수 있습니다. 같은 객체의 필드를 여러 개 출력하는 폼이나 상세 화면에서 코드가 짧아집니다.

### 5. 메시지 #{...}와 링크 @{...}

```properties
# resources/messages.properties
message.value=hello world
```

```html
<p th:text="#{ message.value }"></p>

<link rel="stylesheet" th:href="@{/css/common.css}" />
<a th:href="@{/}">메인으로</a>
<!-- 결과: /lecture/expression?title=링크&no=1 -->
<a th:href="@{/lecture/expression(title='링크', no=1)}">파라미터를 포함한 링크</a>
```

- `#{...}`는 Spring Boot가 기본으로 읽는 `messages.properties`의 키를 출력합니다. 같은 문구를 여러 페이지에서 쓰거나 다국어 처리를 할 때 씁니다.
- `@{...}`는 `/`로 시작하는 경로 앞에 애플리케이션의 **context path를 자동으로 붙여** 줍니다. 서블릿 글마다 "`/`로 시작하는 절대 경로는 루트 배포를 가정한다"고 지적했던 문제를 해결하는 방법입니다. 괄호 안에 쿼리스트링 파라미터도 쓸 수 있습니다.

### 6. th:text vs th:utext: escape 여부

```java
mv.addObject("hello", "hello!<h3>Thymeleaf</h3>");
```

```html
<li th:text="${ hello }"></li>    <!-- hello!<h3>Thymeleaf</h3> 가 글자 그대로 보임 -->
<li th:utext="${ hello }"></li>   <!-- hello! 다음에 h3 제목이 그려짐 -->
<li><input type="text" th:value="'타임리프'"></li>
```

| 속성 | escape | 결과 |
|------|--------|------|
| `th:text` | 적용 | `<`, `>`를 문자로 바꿔 **태그가 글자로** 보임 |
| `th:utext` | 미적용 | 문자열 속 **태그가 실제 태그로** 동작 |
| `th:value` | - | `input`의 `value` 속성 지정 |

수업 주석의 경고가 가장 중요한 부분입니다. **`th:utext`는 사용자가 입력한 값을 그대로 출력하면 `<script>` 같은 악성 코드가 실행될 수 있습니다(XSS).** 신뢰할 수 있는 값에만 쓰고, 기본은 `th:text`를 씁니다. 서블릿 수업에서 `StringBuilder`로 HTML을 만들며 "아이디를 HTML에 그대로 넣으면 XSS 위험이 있다"고 적었는데, Thymeleaf의 `th:text`는 이 escape를 **기본으로** 해 줍니다.

### 7. 인라인 표현식과 JavaScript

태그 속성이 아니라 본문 텍스트에 값을 넣을 때는 인라인 표현식을 씁니다.

| 문법 | 대응 |
|------|------|
| `[[${ hello }]]` | `th:text`처럼 escape |
| `[(${ hello })]` | `th:utext`처럼 escape 안 함 |
| `th:inline="none"` | 인라인 처리를 끄고 대괄호를 글자 그대로 출력 |

JavaScript 안에서 서버 값을 쓰려면 `<script th:inline="javascript">`가 필요합니다. 수업 주석은 세 가지 작성법을 비교했습니다.

```html
<script th:inline="javascript">
    // 1) 동적 페이지에서는 동작하지만, html 파일을 그냥 열면(정적) 문법 오류
    // let hello = [[${ hello }]];

    // 2) 동적 페이지에서 따옴표가 중복되어 문법 오류 -> let hello = ""hello!..."";
    // let hello = "[[${ hello }]]";

    // 3) 내추럴 템플릿 (권장): 정적일 때는 '기본값', 동적일 때는 서버 값
    let hello = /*[[${ hello }]]*/ '기본값';
</script>
```

| 작성법 | 서버 실행 시 | 파일만 열었을 때 |
|--------|--------------|-------------------|
| `[[${hello}]]` | 정상 | JS 문법 오류 |
| `"[[${hello}]]"` | 따옴표 중복 오류 | 정상 |
| `/*[[${hello}]]*/ '기본값'` | 서버 값으로 치환 | `'기본값'` 사용 |

JavaScript 모드에서 `[[...]]`는 값의 타입에 맞게 JS 리터럴로 바꿔 줍니다. 문자열이면 따옴표까지 붙여 주기 때문에 2번처럼 직접 따옴표를 쓰면 중복됩니다. 3번의 **내추럴 템플릿**은 주석 안에 표현식을 두어, 서버를 거치지 않고 파일을 열어도 문법 오류가 없게 만드는 Thymeleaf의 방식입니다.

### 8. 리터럴 치환 |...|, th:block, 주석

```html
<!-- '+' 없이 문자열과 값을 합친다 -->
<p th:object="${ member }" th:text="|name = '*{ name }'|"></p>

<!-- 태그를 만들지 않고 범위만 지정한다 -->
<th:block th:object="${ member }">
    <p th:text="*{ age }"></p>
</th:block>
```

| 문법 | 의미 |
|------|------|
| \|문자열 ${값}\| (세로 막대로 감싸기) | 리터럴 치환. 문자열 안에 값을 바로 끼워 넣음 |
| `<th:block>` | 렌더링 후 사라지는 태그. 범위 지정, 조건, 반복에 사용 |
| `<!--/* ... */-->` | parser-level 주석. 파일에서는 주석, 렌더링 후 **완전히 제거**되어 브라우저에 보이지 않음 |
| `<!--/*/ ... /*/-->` | prototype-only 주석. 파일에서는 주석, 렌더링 후 **내용이 출력됨** |

일반 HTML 주석(`<!-- -->`)은 페이지 소스에 그대로 남습니다. 내부 설명을 사용자에게 보이고 싶지 않다면 parser-level 주석을 쓰면 됩니다.

## 결과 (Result)

정량 지표는 없고, 아래를 정리했습니다.

- `${}`는 위치에 따라 Model(`${이름}`), 요청 파라미터(`${param.이름}`), 세션(`${session.이름}`)을 꺼냅니다.
- `*{}`는 `th:object`로 선택한 객체의 필드, `#{}`는 메시지 파일, `@{}`는 context path가 붙는 URL을 만듭니다.
- `th:text`는 escape해서 XSS를 막고, `th:utext`는 태그를 그대로 그리므로 신뢰할 수 있는 값에만 씁니다.
- JavaScript로 값을 넘길 때는 `th:inline="javascript"`와 내추럴 템플릿(`/*[[...]]*/ 기본값`)을 씁니다.
- 뷰 이름과 템플릿 경로 앞에는 `/`를 붙이지 않습니다.

## 더 학습하면 좋은 개념

- **XSS와 출력 escape** — `th:text`가 무엇을 바꿔 주는지, `th:utext`를 써야 할 때 어떤 정화(sanitize)가 필요한지 이해해야 합니다.
- **국제화(i18n)와 `messages_ko.properties`** — 메시지 표현식이 언어별 파일을 고르는 방식입니다.
- **내추럴 템플릿(Natural Templates)** — 디자이너가 서버 없이 파일을 열어도 화면이 깨지지 않게 하는 Thymeleaf의 설계 철학입니다.
- **SpringEL** — `${}` 안에서 쓰는 표현식 언어로, 메서드 호출과 연산이 가능한 이유입니다.
- **context path와 배포 환경** — `@{}`가 필요한 상황(앱이 `/app` 같은 경로 아래 배포될 때)을 이해해 두면 경로 문제를 줄일 수 있습니다.

## 참고 자료

- [Thymeleaf - Tutorial: Using Thymeleaf (Standard Expression Syntax)](https://www.thymeleaf.org/doc/tutorials/3.1/usingthymeleaf.html#standard-expression-syntax)
- [Thymeleaf - Tutorial: Expression Basic Objects (param, session)](https://www.thymeleaf.org/doc/tutorials/3.1/usingthymeleaf.html#appendix-a-expression-basic-objects)
- [Thymeleaf - Tutorial: Inlining (JavaScript inlining)](https://www.thymeleaf.org/doc/tutorials/3.1/usingthymeleaf.html#inlining)
- [Thymeleaf - Tutorial: Comments and Blocks](https://www.thymeleaf.org/doc/tutorials/3.1/usingthymeleaf.html#comments-and-blocks)
- [Thymeleaf 3.1 - What's new (restricted expression objects)](https://www.thymeleaf.org/doc/articles/thymeleaf31whatsnew.html)
- [Spring Boot Reference - Internationalization (messages)](https://docs.spring.io/spring-boot/reference/features/internationalization.html)

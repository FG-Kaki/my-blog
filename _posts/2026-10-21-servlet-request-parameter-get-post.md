---
layout: post
title: "폼 입력값을 서버에서 꺼내기 - getParameter와 쿼리스트링·폼 데이터"
date: 2026-10-21 09:30:00 +0900
categories: [Java]
tags: [java, servlet, request-parameter, querystring, http]
mermaid: true
---

## 들어가며 (Situation)

이번 실습(`c_parameter`)의 주제는 사용자가 입력한 값을 서버가 받는 방법입니다. 이름, 나이, 생일, 성별, 국적, 취미를 담은 폼을 **GET(쿼리스트링)** 과 **POST(폼 데이터)** 두 방식으로 보내고, 서블릿에서 `getParameter()`로 꺼냅니다.

> 실제 서비스에서 겪은 문제가 아니라, 수업 실습을 정리한 글입니다. 동작 설명은 Jakarta Servlet, HTTP 공식 문서로 보완했습니다. 수업 코드는 `name`과 `age`만 꺼내 콘솔에 출력하므로, 나머지 값이나 실행 출력은 다루지 않았습니다.

## 문제 상황 (Task)

- HTML 입력 요소의 값이 서블릿까지 어떤 형태로 전달될까요?
- 같은 값을 GET과 POST로 보낼 때 서버 코드와 요청은 어떻게 달라질까요?
- 수업 코드가 안전하게 동작하지 않는 경우는 언제일까요?

## 해결 과정 (Action)

### 1. 입력 요소의 `name`이 곧 파라미터 이름

```html
<form action="/querystring" method="get">
    <input type="text"   name="name">
    <input type="number" name="age">
    <input type="date"   name="birthday">
    <input type="radio"  name="gender" value="M"> <input type="radio" name="gender" value="F">
    <select name="national"> ... </select>
    <input type="checkbox" name="hobbies" value="movie"> ... (music, sleep)
    <input type="submit" value="GET 요청">
</form>
```

```mermaid
flowchart LR
    F["HTML form<br/>name 속성 = 파라미터 이름"] -->|"제출"| R["HTTP 요청<br/>name=값&age=값 ..."]
    R --> S["서블릿<br/>req.getParameter(\"name\")"]
```

서버가 값을 꺼내는 키는 `id`나 `label`이 아니라 **`name` 속성**입니다. `radio`는 같은 `name`을 공유해야 하나만 선택되고, 그 값은 선택된 항목의 `value`입니다. `checkbox`는 같은 `name`으로 여러 개가 선택될 수 있습니다.

### 2. GET: 쿼리스트링

```java
@WebServlet(value = "/querystring")
public class QueryStringServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        String name = req.getParameter("name");
        int age = Integer.parseInt(req.getParameter("age"));
        System.out.println("name = " + name + ", age = " + age);
    }
}
```

GET으로 제출하면 입력값이 URL 뒤에 붙습니다. 수업은 `<a>` 태그에 직접 쓴 형태도 보여 줍니다.

```
/querystring?name=홍길동&age=20&birthday=2021-01-08&gender=M&national=ko&hobbies=movie&hobbies=music&hobbies=sleep
```

| 구성 | 의미 |
|------|------|
| `?` | 경로와 쿼리스트링의 구분 |
| `키=값` | 파라미터 하나 |
| `&` | 파라미터 사이의 구분 |
| `hobbies=movie&hobbies=music` | 같은 키를 반복해 여러 값 전달 |

수업 주석의 핵심은 "URL로 전달되는 데이터는 모두 **String**이다. 원하는 자료형이 필요하면 파싱해야 한다"였습니다. 그래서 나이는 `Integer.parseInt`로 변환합니다.

### 3. POST: 폼 데이터

```java
@WebServlet(value = "/formdata")
public class FormDataServlet extends HttpServlet {

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) {
        String name = req.getParameter("name");
        System.out.println("name = " + name);
    }
}
```

폼의 `method`만 `post`로 바꾸면 값은 URL이 아니라 요청 **본문(body)** 에 담겨 전달됩니다. 서버 코드에서는 읽는 방식이 같습니다. `getParameter`는 쿼리스트링과 폼 본문의 파라미터를 같은 방식으로 꺼냅니다. 다른 점은 요청을 받는 메서드가 `doPost`라는 것뿐입니다.

### 4. 수업의 GET vs POST 설명을 공식 문서로 점검

수업 주석의 설명을 표로 옮기고, 공식 문서 기준으로 보정했습니다.

| 수업 주석 | 공식 문서 기준으로 보면 |
|-----------|--------------------------|
| GET은 URL에 데이터가 노출되어 보안상 취약 | 맞습니다. URL은 브라우저 기록, 서버 로그, 리퍼러에 남을 수 있습니다. 다만 POST도 HTTPS가 아니면 본문이 암호화되지 않아 "POST면 안전"은 아닙니다. |
| GET은 약 200자 내의 데이터만 전달 | HTTP 표준에 고정 길이 제한은 없고, 브라우저·서버·프록시마다 다릅니다. 보통 수천 자 이상이지만 서버 설정(예: 헤더 크기)에 걸립니다. "대략 200자"는 정확한 값이 아닙니다. |
| POST는 본문(payload)에 담아 큰 데이터 전달 가능 | 맞습니다. 게시글 같은 긴 입력에 적합합니다. |
| GET은 빠르고 조회에, POST는 느려서 단순 조회엔 부적합 | 속도 차이는 보통 미미합니다. 실제 구분 기준은 의미입니다. GET은 서버 상태를 바꾸지 않는 조회, POST는 데이터 생성·변경에 씁니다. |

수업이 "게시글 등을 insert하는 것에 GET은 적합하지 않다"고 한 결론은 맞지만, 이유는 길이 제한보다 **의미(조회 vs 변경)** 로 설명하는 편이 정확합니다. 수업 코드의 주석에는 "payload(baby)"처럼 오타가 있어 body로 읽었습니다.

### 5. 수업 코드가 깨지는 경우

실습 코드는 정상 입력만 가정합니다. 아래 경우를 직접 실행해 확인하지는 못했고, API 문서의 동작 설명을 바탕으로 정리했습니다. (예시)

| 상황 | 결과 |
|------|------|
| 나이를 비워 두고 제출 | `getParameter("age")`가 빈 문자열(`""`)을 반환 → `parseInt("")`에서 `NumberFormatException` |
| `age` 파라미터가 아예 없음(주소를 직접 입력) | `getParameter`가 `null` 반환 → `parseInt(null)`에서 `NumberFormatException` |
| 라디오를 선택하지 않음 | `gender` 파라미터가 전달되지 않아 `getParameter("gender")`가 `null` |
| 취미를 여러 개 선택 | `getParameter("hobbies")`는 첫 값만 반환 |

마지막 경우는 수업 코드가 `hobbies`를 꺼내지 않아서 나타나지 않았지만, 같은 이름의 값이 여러 개일 때는 `getParameterValues("hobbies")`가 `String[]`로 모두 돌려줍니다. 공식 API에서 확인한 내용이고 이번 수업 코드에는 없습니다.

### 6. 한글 입력은 어떻게 될까

`<a>` 태그의 쿼리스트링에는 한글 이름(`홍길동`)이 들어 있습니다. 브라우저는 URL의 비ASCII 문자를 퍼센트 인코딩해서 보내고, 서버가 이를 디코딩합니다. 요청 본문(POST)의 한글을 읽을 때 문자 인코딩이 맞지 않으면 글자가 깨질 수 있습니다. 서블릿 사양 버전과 컨테이너에 따라 기본 인코딩이 다르므로, 파라미터를 읽기 전에 `req.setCharacterEncoding("UTF-8")`을 호출하는 방식이 널리 쓰입니다. 이번 수업 코드에는 없고, 제 환경에서 깨짐을 재현해 보지도 않았습니다. 사용 중인 서버의 문서에서 기본 인코딩을 확인하시길 권합니다.

## 결과 (Result)

정량 지표는 없고, 아래를 정리했습니다.

- 파라미터의 키는 HTML 입력 요소의 `name` 속성이며, 서버에서는 `getParameter("name")`으로 꺼냅니다.
- 전달되는 모든 값은 `String`이라 숫자·날짜는 직접 변환해야 합니다.
- GET은 URL 쿼리스트링, POST는 요청 본문으로 보내고 서버의 `getParameter`는 둘을 같은 방식으로 읽습니다.
- 빈 입력과 누락된 파라미터에 대한 검증이 없으면 `NumberFormatException`이 납니다.
- GET/POST의 진짜 구분 기준은 길이나 속도가 아니라 **조회냐 변경이냐**입니다.

## 더 학습하면 좋은 개념

- **입력 검증과 예외 처리** — `null`과 빈 문자열, 형식 오류를 어디서 막을지 설계하는 기본기입니다.
- **`getParameterValues`와 `getParameterMap`** — 체크박스처럼 다중 값을 받는 방법입니다.
- **문자 인코딩(UTF-8)과 퍼센트 인코딩** — 한글 파라미터가 깨지는 문제의 원인과 해결법을 이해하는 데 필요합니다.
- **HTTPS와 민감 정보 전송** — POST가 안전하다는 오해를 바로잡고, 비밀번호 전송의 올바른 방식을 알게 합니다.
- **Spring의 `@RequestParam`과 `@ModelAttribute`** — `getParameter` 호출과 형 변환을 프레임워크가 대신해 주는 방식입니다.

## 참고 자료

- [Jakarta Servlet 6.1 API - ServletRequest (getParameter, getParameterValues)](https://jakarta.ee/specifications/servlet/6.1/apidocs/jakarta.servlet/jakarta/servlet/servletrequest)
- [MDN - Sending form data](https://developer.mozilla.org/en-US/docs/Learn_web_development/Extensions/Forms/Sending_and_retrieving_form_data)
- [MDN - GET method](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/GET)
- [MDN - POST method](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/POST)
- [RFC 9110 - HTTP Semantics (Methods)](https://www.rfc-editor.org/rfc/rfc9110#name-methods)
- [MDN - Percent-encoding](https://developer.mozilla.org/en-US/docs/Glossary/Percent-encoding)

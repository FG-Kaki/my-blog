---
layout: post
title: "서블릿이 화면을 직접 그리기 - HttpServletResponse와 PrintWriter로 HTML 응답하기"
date: 2026-10-22 09:10:00 +0900
categories: [Java]
tags: [java, servlet, response, printwriter, content-type]
mermaid: true
---

## 들어가며 (Situation)

지금까지의 서블릿은 요청을 받고 콘솔에만 출력했습니다. 그래서 브라우저에는 빈 화면이 나왔습니다. 이번 실습(`d_response`)은 처음으로 **서블릿이 직접 만든 HTML을 브라우저에 응답**합니다. 링크(`<a href="/response">응답해</a>`)를 누르면 `/response` 서블릿이 응답합니다.

> 실제 서비스에서 겪은 문제가 아니라, 수업 실습을 정리한 글입니다. 실습은 Tomcat에서 진행했고 별다른 문제 없이 동작했습니다. 동작 설명은 Jakarta Servlet 공식 문서로 보완했습니다.

## 문제 상황 (Task)

- 서블릿이 클라이언트에게 HTML을 돌려주려면 어떤 객체를 어떻게 써야 할까요?
- 한글이 깨지지 않게 하려면 무엇을 지정해야 할까요?
- 서블릿의 역할은 어디까지일까요?

## 해결 과정 (Action)

### 1. 서블릿의 역할 세 가지

수업 주석은 서블릿의 역할을 이렇게 정리했습니다.

| 역할 | 내용 |
|------|------|
| 1. 요청 전달받기 | GET/POST 방식에 따라 `parameter`로 전달된 데이터를 꺼냄 |
| 2. MVC 중 Controller 역할 | 전달받은 데이터를 Service 계층에 전달 |
| 3. 응답 전달하기 | `Response` 객체로 사용자에게 응답 |

JDBC 수업 후반의 `MenuController`와 같은 위치입니다. 이번 글은 세 번째 역할인 응답을 다룹니다.

### 2. HTML 문자열 만들기

```java
StringBuilder response = new StringBuilder();
response.append("<!doctype html>\n")
        .append("<html>\n")
        .append("<body>\n")
        .append("<h1>Servlet 의 응답 결과!</h1>")
        .append("</body>\n")
        .append("</html>\n");
```

응답할 HTML을 `StringBuilder`로 한 줄씩 이어 붙입니다. 이 방식은 곧 한계가 드러납니다. HTML이 길어지면 자바 문자열 안에서 태그와 따옴표를 이스케이프(`\"`)하며 작성해야 해서 가독성이 크게 떨어집니다. 이번 수업 코드에서도 `align=\"center\"`처럼 이스케이프가 나타납니다. 이런 불편을 줄이는 도구는 수업에서 다루지 않았습니다.

### 3. 응답 보내기: setContentType → getWriter

```java
System.out.println("response type : " + resp.getContentType());
// 서버에서 응답하는 데이터의 형태는 HTML 형태이다.
resp.setContentType("text/html;charset=UTF-8");

PrintWriter out = resp.getWriter();
out.print(response);
out.flush();
out.close();
```

```mermaid
sequenceDiagram
    participant B as 브라우저
    participant S as ResponseServlet
    B->>S: GET /response
    S->>S: resp.setContentType("text/html;charset=UTF-8")
    S->>S: PrintWriter out = resp.getWriter()
    S-->>B: out.print(HTML) / flush / close
    B->>B: Content-Type 을 보고 HTML 로 렌더링
```

| 코드 | 역할 |
|------|------|
| `resp.setContentType("text/html;charset=UTF-8")` | 응답 본문이 HTML이고 문자 인코딩이 UTF-8임을 브라우저에 알림 (`Content-Type` 헤더) |
| `resp.getWriter()` | 응답 본문에 문자를 쓰는 `PrintWriter` 반환 |
| `out.print(...)` | 본문에 쓰기 |
| `out.flush()` / `out.close()` | 버퍼의 내용을 내보내고 스트림 닫기 |

**순서가 중요합니다.** `setContentType`(특히 `charset`)은 `getWriter()`를 호출하기 **전에** 해야 합니다. API 문서는 문자 인코딩을 `getWriter()` 호출 전에 지정해야 적용된다고 안내합니다. 수업 코드는 이 순서를 지켰습니다. 순서를 바꾸면 한글이 깨질 수 있다는 점은 직접 재현해 보지 않았습니다.

수업 코드의 `resp.getContentType()` 출력은 `setContentType` **전**에 호출하므로, 값이 아직 지정되지 않은 상태를 확인하는 용도로 이해했습니다. 이때의 반환값은 기록하지 않았습니다.

### 4. 한글 응답 관련 주석

다음 글(예외 처리)의 `ExceptionServlet` 주석에는 "응답 시에 항상 해 주어야 한글이 깨지지 않는다. 계속 반복되는 코드"라는 말이 있습니다. 서블릿마다 `setContentType`과 `PrintWriter` 작성이 반복된다는 뜻입니다. 이 반복과 문자열 HTML의 불편함이 이후 단계(뷰 기술)의 필요성으로 이어진다고 이해했습니다. 수업에서 다음 주제가 무엇인지는 직접 확인하지 못했습니다.

### 5. 주의할 점: 사용자 입력을 그대로 HTML에 넣지 않기

이 글의 응답은 고정된 문자열이라 안전합니다. 하지만 같은 방식으로 **사용자가 입력한 값**을 HTML에 이어 붙이면, 입력에 `<script>` 같은 태그가 있을 때 그대로 실행되는 XSS 취약점이 생깁니다. 이 부분은 다음 글들의 코드(로그인 아이디를 응답에 포함)와도 연결됩니다.

## 결과 (Result)

정량 지표는 없고, 아래를 정리했습니다.

- 서블릿은 요청 수신, 데이터 전달(Controller), 응답이라는 세 가지 역할을 갖습니다.
- 응답은 `setContentType`으로 형식과 인코딩을 지정한 뒤 `getWriter()`로 쓰고, 이 순서를 지켜야 합니다.
- HTML을 자바 문자열로 조립하는 방식은 길어질수록 불편하며, 같은 코드를 서블릿마다 반복하게 됩니다.

## 더 학습하면 좋은 개념

- **HTTP 응답 구조(상태 줄·헤더·본문)** — `Content-Type`이 왜 필요한지, 브라우저가 이를 어떻게 쓰는지 이해하는 기초입니다.
- **`getWriter()` vs `getOutputStream()`** — 문자와 바이너리(이미지, 파일 다운로드) 응답을 구분하는 기준입니다. 둘을 한 응답에서 함께 쓰면 오류가 납니다.
- **XSS와 HTML 이스케이프** — 문자열로 HTML을 만들 때 가장 흔한 보안 위험입니다.
- **뷰 템플릿 기술(JSP, Thymeleaf 등)** — HTML을 자바 코드에서 분리하는 방법으로, 이번 코드가 왜 불편한지의 해결 방향입니다.
- **MVC 패턴** — 서블릿이 Controller라는 설명을 Spring MVC까지 이어서 이해하는 데 필요합니다.

## 참고 자료

- [Jakarta Servlet 6.1 API - HttpServletResponse](https://jakarta.ee/specifications/servlet/6.1/apidocs/jakarta.servlet/jakarta/servlet/http/httpservletresponse)
- [Jakarta Servlet 6.1 API - ServletResponse (setContentType, getWriter)](https://jakarta.ee/specifications/servlet/6.1/apidocs/jakarta.servlet/jakarta/servlet/servletresponse)
- [MDN - Content-Type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type)
- [OWASP - Cross Site Scripting (XSS)](https://owasp.org/www-community/attacks/xss/)

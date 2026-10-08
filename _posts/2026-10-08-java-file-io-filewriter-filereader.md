---
layout: post
title: "write()했는데 파일이 비어 있다? - FileWriter의 버퍼와 try-with-resources"
date: 2026-10-08 13:00:00 +0900
categories: [Java]
tags: [java, file-io, filewriter, filereader, try-with-resources]
mermaid: true
---

## 들어가며 (Situation)

5챕터의 두 번째 주제는 파일 입출력(File IO)입니다. `b_fileio.Application01`은 `FileWriter`로 파일에 문자열을 쓰고, `FileReader`로 다시 읽어 출력합니다. 앞선 예외 글들의 `try-catch`가 실제로 **강제되는 현장**이기도 합니다. `IOException`이 checked 예외이기 때문입니다.

> 실제 서비스에서 겪은 문제가 아니라, 개념 학습 중 정리한 내용입니다. 출력은 JDK 21(Windows)에서 직접 확인했습니다.

## 문제 상황 (Task)

- `write()`를 호출하면 바로 파일에 써질까요? 수업 코드에는 `flush()`가 있는데 없으면 어떻게 될까요?
- `new FileWriter("output.txt")`를 두 번 실행하면 내용이 쌓일까요, 지워질까요?
- `read()`의 반환 타입은 왜 `char`가 아니라 `int`일까요?
- 수업 코드는 `close()`를 생략했습니다. 실무에서는 어떻게 해야 할까요?

## 해결 과정 (Action)

### 1. 쓰기와 읽기의 기본 흐름

```mermaid
flowchart LR
    P["프로그램(메모리)"] -->|"FileWriter.write()"| F["output.txt"]
    F -->|"FileReader.read()"| P
```

쓰기는 프로그램 → 파일(Output), 읽기는 파일 → 프로그램(Input)입니다. 두 클래스는 **문자(char) 단위**로 처리합니다. 상대 경로 `"output.txt"`는 프로그램을 실행한 위치(IDE에서는 보통 프로젝트 폴더) 기준입니다.

```java
FileWriter writer = new FileWriter("output.txt");
writer.write("Hello, File IO!!");
writer.write("File Test");
writer.flush();
```

두 번 `write`했지만 줄바꿈 문자를 넣지 않았으므로 파일에는 `Hello, File IO!!File Test`로 이어서 저장됩니다.

### 2. 시행착오 1: flush()를 빼면 파일이 비어 있을까?

수업 주석은 "`write()`한 내용은 먼저 메모리 버퍼에 쌓이고 `flush()`나 `close()` 시 파일에 기록된다"고 합니다. 정말인지 파일 크기를 찍어 확인했습니다. (예시 코드)

```java
FileWriter nf = new FileWriter("nf.txt");
nf.write("abc");
System.out.println("flush 전 = " + new File("nf.txt").length());
nf.flush();
System.out.println("flush 후 = " + new File("nf.txt").length());
```

```text
flush 전 = 0
flush 후 = 3
```

`write("abc")` 직후 파일 크기는 **0바이트**, `flush()` 후에야 3바이트가 됐습니다. 버퍼 설명이 사실이었습니다. 그렇다면 `flush()`도 `close()`도 호출하지 않고 프로그램이 끝나면 데이터를 잃을 수 있다는 뜻이라, 수업 코드에서 `flush()`가 빠지면 안 되는 이유가 분명해졌습니다. (이 경우의 정확한 동작은 이번에 실험하지 않았습니다. 자원은 `close()`로 반납하는 것이 정석입니다.)

### 3. 시행착오 2: 덮어쓰기와 이어쓰기

수업 주석에는 "이미 있으면 기존 내용을 지우고 덮어쓴다"와 "`new FileWriter("output.txt", true)`로 이어쓰기"가 적혀 있습니다. 같은 파일에 이어서 `+`를 써 봤습니다. (예시 코드)

```java
try (FileWriter w = new FileWriter("o.txt", true)) {
    w.write("+");
}
```

```text
Hello, File IO!!File Test+
```

| 생성 방식 | 기존 파일이 있을 때 |
|-----------|--------------------|
| `new FileWriter("o.txt")` | 내용을 **지우고** 새로 씀 |
| `new FileWriter("o.txt", true)` | 끝에 **이어서** 씀 |

기본이 "덮어쓰기"라서, 로그처럼 쌓아야 하는 파일에 옵션 없이 쓰면 이전 내용이 사라집니다.

### 4. 읽기: 왜 read()는 int를 반환할까?

```java
FileReader reader = new FileReader("output.txt");
int data;
while ((data = reader.read()) != -1) {
    System.out.println((char) data);
}
```

`read()`는 한 문자를 읽어 그 값을 돌려주고, 파일 끝에서 `-1`을 반환합니다. 문자 값은 0 이상이므로 `-1`은 "끝"을 알리는 신호로 쓸 수 있는데, `char`는 음수를 표현할 수 없습니다. 그래서 값과 신호를 함께 담으려고 `int`를 반환하고, 출력할 때 `(char)`로 되돌립니다. `(data = reader.read()) != -1`은 대입과 비교를 한 줄에서 처리하는 관용구입니다. `println`이라 한 글자마다 줄바꿈되어 출력되는 점도 수업 주석 그대로였습니다.

### 5. 시행착오 3: 파일이 없을 때와 catch 순서

없는 파일을 열어 봤습니다. (예시 코드)

```java
try {
    new FileReader("nope.txt");
} catch (FileNotFoundException e) {
    System.out.println(e.getMessage());
}
```

```text
nope.txt (지정된 파일을 찾을 수 없습니다)
```

예외 메시지는 운영체제의 메시지를 그대로 가져오는 것으로 보여, 한국어 Windows에서 이렇게 나왔습니다. 환경마다 다를 수 있습니다.

그리고 `FileNotFoundException`의 부모를 확인해 보니 `IOException`이었습니다. 그래서 수업 주석처럼 자식을 먼저 써야 합니다. 순서를 바꾸면 이전 글과 같은 컴파일 에러입니다. (예시 코드)

```java
catch (IOException e) { } catch (FileNotFoundException e) { }
```

```text
error: exception FileNotFoundException has already been caught
```

### 6. 시행착오 4: close()를 안 하면 — try-with-resources

수업 코드는 `reader`도 `writer`도 닫지 않습니다. 수업 주석도 "실무에서는 try-with-resources가 안전하다"고 짚습니다. 닫는 코드를 `finally`에 직접 쓰면 `close()`도 `IOException`을 던져서 `try-catch`가 중첩되고 복잡해집니다. 대안을 비교하면 이렇습니다.

| 항목 | 닫지 않음 (수업 코드) | `finally`에서 `close()` | `try-with-resources` |
|------|----------------------|--------------------------|----------------------|
| 자원 반납 | 보장 안 됨 | 직접 구현 | 블록 종료 시 **자동** |
| 코드량 | 적음 | 많음 (중첩) | 적음 |
| 예외 발생 시 | 열린 채 남을 수 있음 | 닫힘 | 닫힘 |

```java
try (FileWriter w = new FileWriter("o.txt")) {
    w.write("Hello, File IO!!");
    w.write("File Test");
}   // 블록을 벗어나면 close() 자동 호출 (close 안에서 flush도 수행)
```

앞에서 본 `flush()` 문제도 `close()`가 자동 호출되면서 함께 해결됩니다. 위 코드로 쓴 파일과 읽기 결과가 `Hello, File IO!!File Test`로 같음을 확인했습니다.

### 7. 시행착오 5: 한글은 안전할까?

한글을 쓰고 파일 크기를 봤습니다. (예시 코드)

```java
try (FileWriter w = new FileWriter("한글.txt")) { w.write("한글"); }
System.out.println(Files.size(Path.of("한글.txt")) + " " + Charset.defaultCharset());
```

```text
6 UTF-8
```

한글 2글자가 6바이트(글자당 3바이트)이고 기본 문자셋이 UTF-8이라는 뜻입니다. 다만 이것은 이번 JDK 21 환경에서의 결과입니다. 오래된 JDK나 다른 설정에서는 기본 문자셋이 다를 수 있다고 알고 있는데, 정확한 버전별 차이는 [JEP 400](https://openjdk.org/jeps/400)에서 확인해야 합니다. 환경에 상관없이 안전하게 쓰려면 문자셋을 명시합니다. (예시 코드, `FileWriter(String, Charset)`은 Java 11부터)

```java
new FileWriter("o.txt", StandardCharsets.UTF_8);
```

## 결과 (Result)

- `write()` 직후 파일 크기가 **0바이트**, `flush()` 후 **3바이트**임을 측정해 버퍼링을 확인했습니다.
- 기본 생성은 덮어쓰기, `true` 옵션은 이어쓰기임을 출력(`Hello, File IO!!File Test+`)으로 확인했습니다.
- `read()`가 `int`를 반환하는 이유(문자 값 + 끝 신호 `-1`)와 `FileNotFoundException`이 `IOException`의 자식이어서 `catch` 순서가 중요함을 컴파일 에러로 재현했습니다.
- 수업 코드의 자원 미반납 문제를 `try-with-resources`로 개선하는 방법을 비교표로 정리했고, 한글 2글자가 UTF-8에서 6바이트가 되는 것을 확인했습니다.
- 예외 글에서 다룬 checked 예외의 강제가, `FileWriter`/`FileReader` 생성 시 `IOException` 처리를 요구하는 모습으로 실제로 나타나는 것을 확인했습니다.

## 더 학습하면 좋은 개념

- **`BufferedReader` / `BufferedWriter`** — 문자 하나씩이 아닌 줄(`readLine`) 단위로 읽고 쓰는 표준 방법입니다. 지금처럼 한 글자씩 `println`하는 방식보다 훨씬 실용적입니다.
- **바이트 스트림과 문자 스트림 (`InputStream`/`Reader`)** — `FileReader`는 문자용입니다. 이미지·압축 파일 같은 바이너리 처리에는 바이트 스트림이 필요합니다.
- **NIO.2 (`java.nio.file.Files`, `Path`)** — `Files.readAllLines`, `Files.writeString` 같은 더 간결한 현대적 파일 API입니다. 이번 실험의 `Files.size`도 여기 속합니다.
- **`AutoCloseable`과 억제된 예외(suppressed exception)** — `try-with-resources`가 어떻게 동작하고 `close()`의 예외를 어떻게 다루는지의 원리입니다.
- **문자 인코딩(UTF-8, CP949)** — 한글 파일이 깨지는 문제의 원인입니다. 파일을 주고받는 모든 상황에서 만나게 됩니다.

## 참고 자료

- [Oracle Java Tutorials - Basic I/O](https://docs.oracle.com/javase/tutorial/essential/io/index.html)
- [Oracle Java Tutorials - The try-with-resources Statement](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)
- [Java SE 21 API - FileWriter](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/io/FileWriter.html)
- [Java SE 21 API - FileReader](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/io/FileReader.html)
- [JEP 400: UTF-8 by Default](https://openjdk.org/jeps/400)

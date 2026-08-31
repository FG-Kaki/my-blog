---
layout: post
title: "add와 commit이 왜 나뉘어 있는가 - 첫 커밋에서 막힌 것들"
date: 2026-08-28 10:00:00 +0900
categories: [Git]
tags: [git, version-control, commit, beginner]
mermaid: true
---

## 들어가며 (Situation)

독학으로 프로그래밍을 조금 해봤지만 Git은 오늘 처음 제대로 배웠다. "저장 지점을 남기고 되돌릴 수 있게 해주는 도구"라는 설명은 들었는데, 정작 손으로 커밋을 찍어보기 전까지는 이 문장이 뭘 뜻하는지 감이 안 왔다.

## 문제 상황 (Task)

명령어 자체는 세 개뿐이라 외우는 건 어렵지 않았다.

| 단계 | 명령 | 하는 일 |
|---|---|---|
| 1 | `git add` | 커밋에 담을 것을 고른다 |
| 2 | `git commit` | 고른 것을 한 덩어리로 기록한다 |
| 3 | `git push` | 기록을 GitHub로 올린다 |

문제는 **왜 `add`와 `commit`이 굳이 두 단계로 나뉘어 있는지**였다. 그냥 `git commit`만 있으면 안 되나? 라는 의문이 계속 남았다. 거기다 `git log`를 처음 쳤을 때 화면이 그대로 멈춰버려서, 터미널이 죽은 줄 알고 당황했다.

## 해결 과정 (Action)

일부러 파일을 두 개 고친 뒤에 하나만 커밋해보기로 했다.

```mermaid
flowchart LR
    W["파일 A, 파일 B\n둘 다 수정함"] -->|git add 파일A만| S[Staging Area\n파일 A만 선택됨]
    S -->|git commit| C1[커밋: 파일 A만 기록]
    W -.->|파일 B는 아직 그대로| S
```

`git add 파일A`만 실행하고 `git commit`을 해보니, 실제로 파일 B는 그대로 작업 중인 상태로 남고 파일 A만 기록으로 남았다. `add`가 없었다면 "수정된 걸 전부"가 아니라 "이번에 담고 싶은 것만" 골라 담을 방법이 없었을 거라는 걸 직접 확인했다.

`git log`가 멈춘 것처럼 보인 것도 원인을 찾아보니 별일이 아니었다. 커밋 기록을 보여주는 화면(pager)이 열린 것이었고, `q`를 누르면 바로 빠져나올 수 있었다.

## 결과 (Result)

- `add`/`commit` 분리 이유를 문서가 아니라 직접 재현해서 확인함 — 파일 두 개 중 하나만 골라 커밋하는 걸 실제로 성공시킴.
- `git log`가 멈춘 게 아니라 pager 화면이라는 것, `q`로 빠져나온다는 것을 알게 되어 다음부터는 당황하지 않게 됨.

## 더 학습하면 좋은 개념

- **`git status`** — 지금 Working Directory와 Staging Area에 뭐가 올라가 있는지 매번 눈으로 확인하는 습관을 들이면 `add`/`commit` 단계를 헷갈릴 일이 줄어든다.
- **`.gitignore`** — 커밋에 담고 싶지 않은 파일(로그, 캐시 등)을 애초에 `add` 후보에서 빼는 방법. `add`의 "선택"이라는 개념과 바로 이어진다.
- **`git diff`** — `add` 전에 정확히 뭐가 바뀌었는지 미리 확인하는 명령어. Staging Area로 넘기기 전 검토 단계로 자연스럽게 이어진다.

## 참고 자료
- [Git 공식 문서 - git-add](https://git-scm.com/docs/git-add)
- [Git 공식 문서 - git-commit](https://git-scm.com/docs/git-commit)
- [Git 공식 문서 - git-log](https://git-scm.com/docs/git-log)

# Personal AI Workspace

AI와 함께 개발, 학습, 리서치, 문서 작업을 수행하기 위한 개인 작업 공간이다.

반복적으로 사용하는 **Skills**, 작업 과정에서 축적되는 **Knowledge와 경험**, 참고 자료 등을 한곳에서 관리하고 프로젝트 간에 재사용하는 것을 목표로 한다.

## Structure

```text
Personal-AI-Workspace/
│
├── .claude/
│   └── skills/
│       ├── lecture-summary/
│       ├── velog-post/
│       ├── notion-organize/
│       └── technical-research/
│
├── knowledge/
│   ├── android/
│   ├── flutter/
│   └── react/
│
├── wiki/
│   ├── mistakes/
│   ├── troubleshooting/
│   └── notes/
│
└── references/
    ├── official-docs/
    └── examples/
```

## Skills

반복적으로 수행하는 작업의 방법과 규칙을 정의한다.

```text
.claude/skills/
├── lecture-summary/       # 강의 및 세미나 정리
├── velog-post/            # Velog 글 작성
├── notion-organize/       # Notion 문서 정리
└── technical-research/    # 기술 리서치 및 정리
```

Skill은 특정 작업을 수행할 때 필요한 **절차, 규칙, 출력 형식** 등을 정의한다.

## Knowledge

AI가 참고할 수 있도록 기술 및 학습 내용을 정리해 둔다.

```text
knowledge/
├── android/
├── flutter/
└── react/
```

공식 문서나 여러 자료를 학습한 뒤, 나중에 다시 활용할 수 있는 형태로 정리한다.

## Wiki

작업하면서 발생한 문제와 해결 과정, 경험을 기록한다.

```text
wiki/
├── mistakes/
├── troubleshooting/
└── notes/
```

특히 AI를 활용하면서 발생한 실수나 문제 해결 과정을 기록하여 이후 작업에서 참고할 수 있도록 한다.

```text
AI 작업
   ↓
문제 / 실수 발생
   ↓
원인 및 해결 방법 기록
   ↓
Wiki에 축적
   ↓
다음 작업에서 참고
```

## References

필요한 경우 AI 작업에서 참고할 원본 자료나 예시를 보관한다.

```text
references/
├── official-docs/
└── examples/
```

모든 자료를 이곳에 저장하는 것이 목적은 아니며, 반복적으로 참고할 필요가 있는 자료 위주로 관리한다.

## Workflow

```text
                 Personal AI Workspace
                          │
        ┌─────────────────┼─────────────────┐
        ↓                 ↓                 ↓
      Skills          Knowledge            Wiki
        │                 │                 │
        │                 │                 │
        └─────────────────┼─────────────────┘
                          ↓
                     AI 작업 수행
                          ↓
                 새로운 지식 / 경험 축적
                          │
                          └──────→ 다시 Workspace에 반영
```

## Goal

AI에게 단순히 작업을 맡기는 것을 넘어,

> **내가 반복해서 하는 작업의 방법과 지식을 AI가 점점 더 잘 활용할 수 있도록 축적한다.**

개발뿐만 아니라 학습, 기술 리서치, 블로그 작성, Notion 정리 등 다양한 작업에 활용한다.

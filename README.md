# Personal AI Workspace

AI와 함께 개발, 학습, 리서치, 문서 작업을 수행하기 위한 개인 작업 공간이다.

반복적으로 사용하는 **Skills**, 작업 과정에서 축적되는 **Knowledge와 경험**, 참고 자료 등을 한곳에서 관리하고 프로젝트 간에 재사용하는 것을 목표로 한다.

이 폴더는 [Obsidian](https://obsidian.md) Vault이자 Claude Code의 작업 디렉토리로 함께 사용한다. 노트는 Obsidian에서 작성하고, 정리·요약·리서치는 AI가 같은 폴더에서 수행한다.

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
├── references/
│   ├── official-docs/
│   ├── examples/
│   └── conference/
│       └── GDG 안드로이드 소소밋업/
│
├── rules/
│
├── attachments/
│
└── projects/                # Git 추적 제외 (프로젝트별 별도 저장소)
    ├── CallFromAi/
    └── LinkU_Android/
```

`.obsidian/`(Obsidian 설정)과 `.claudian/`(Claudian 플러그인 설정·세션)은 도구 설정이므로 Git에서 제외한다.

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
├── examples/
└── conference/
    └── GDG 안드로이드 소소밋업/
```

- `official-docs/`: 반복적으로 참고하는 공식 문서
- `examples/`: 참고용 예시 코드 및 문서
- `conference/`: 컨퍼런스·밋업 발표를 듣고 정리한 노트. 행사별로 폴더를 만들고, 발표별 정리본을 둔다.

## Rules

여러 프로젝트에 공통으로 적용할 AI 작업 규칙(아키텍처, 코딩 컨벤션, 금지 사항 등)을 둔다.

프로젝트에만 해당하는 규칙은 각 프로젝트의 `AGENTS.md`에 두고, 프로젝트 간에 개인적으로 재사용할 규칙만 이곳에서 관리한다.

## Attachments

노트에 삽입하는 스크린샷 등 이미지 파일을 보관한다. Obsidian에서 `![[파일명.png]]` 형식으로 참조한다.

## Projects

실제로 개발 중인 프로젝트는 **Symbolic Link(심링크)** 방식으로 이 워크스페이스의 `projects/`에 연결한다. 프로젝트의 실제 파일은 별도의 로컬 디렉토리에서 관리하며, AI Workspace에서는 이를 참조할 수 있도록 구성한다. 

AI는 프로젝트의 코드와 함께 **Skills, Knowledge, Wiki, Rules**를 참고하여 작업한다.

```text
projects/
├── 프로젝트 1/      
└── 프로젝트 2/     
```

각 프로젝트는 자체 Git 저장소를 가지므로 이 저장소에서는 추적하지 않는다(`.gitignore`).

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

개발뿐만 아니라 학습, 기술 리서치, 블로그 작성, Notion&Obsidian 정리 등 다양한 작업에 활용한다.

# AI Memory System — Full Design Spec

macOS에서 디지털 생활 데이터를 자동 수집 → Claude가 3-tier 주기로 정제 → 구조화된 메모리로 누적하는 파이프라인.

빈 프로젝트에서 이 스펙만으로 전체 시스템을 구현할 수 있다. 모든 코드는 첨부 파일에 포함.

---

## Placeholders

이 스펙 전체에서 사용되는 플레이스홀더. 구현 시작 전 사용자에게 확인받고 치환한다.

| Placeholder | 의미 | 예시 |
|-------------|------|------|
| `{project}` | 프로젝트 디렉토리명 | `my-agent` |
| `{username}` | macOS 사용자명 | `john` |
| `{project-path}` | 프로젝트 절대 경로 | `/Users/john/my-agent` |
| `{user-name}` | 사용자 실명 | `홍길동` |
| `{domains}` | 사용자가 정의한 라이프 도메인 목록 | `career, finance, health, inner, relationships, learning` |

---

## Prerequisites

구현 시작 전 사용자에게 다음을 확인한다. 미설치 항목은 **사용자 동의 후** 설치.

- [ ] macOS
- [ ] Homebrew (`/opt/homebrew/bin/brew`)
- [ ] Claude Code CLI (`/opt/homebrew/bin/claude`) + Anthropic API 키 설정 완료
- [ ] Hammerspoon (`brew install --cask hammerspoon`)
  - [ ] 시스템 설정 > 개인정보 보호 및 보안 > 접근성 > Hammerspoon 허용
- [ ] Xcode Command Line Tools (`xcode-select --install`)
- [ ] Python 3 (`python3 --version`)
- [ ] Telegram Bot 토큰 + Chat ID ([@BotFather](https://t.me/BotFather)에서 생성)
- [ ] 카카오톡 설치 + 읽고 싶은 채팅방 창을 열어둔 상태
- [ ] 카카오톡 Dock 아이콘 우클릭 > 옵션 > "Assign To: All Desktops"

---

## Architecture

### Data Flow

```
Hammerspoon(앱감시+카톡) → sources/raw/
                              ↓ [10min tier — Claude]
                         sources/ingest/
                              ↓ [3hr tier — Claude]
                      memory/stream/daily/
                              ↓ [24hr tier — Claude]
                   memory/state/ + knowledge/
```

### 3-Tier Pipeline

| Tier | 주기 | 역할 | 입력 | 출력 |
|------|------|------|------|------|
| 10min | 600초 | 적재(Ingest) | sources/raw/ | sources/ingest/ |
| 3hr | 10800초 | 해석(Interpret) | sources/ingest/ + memory/ | memory/stream/daily/ |
| 24hr | 매일 04:00 | 성찰(Reflect) | memory/stream/daily/ + memory/ | memory/state/ + knowledge/ |

### Memory Model (인지과학 대응)

| 계층 | 역할 | 로드 시점 | 수명 | 인지 대응 |
|------|------|-----------|------|-----------|
| AGENT.md + USER.md | 행동 규칙 + 사용자 프로파일 | 매 세션 자동 | 거의 불변 | 절차 기억 |
| state/ | "지금" — 현재 상태 | 매 세션 최우선 | 지속 업데이트 | 작업 기억 |
| stream/ | "언제 무슨 일" — 시간순 | 온디맨드 | 압축 | 일화 기억 |
| knowledge/ | "영구 유효" — 의미 기반 | 온디맨드 | 영구 | 의미 기억 |
| sources/ | 원본 데이터 | 후처리 참조 | 불변 | 감각 버퍼 |

### Directory Structure

```
{project}/
├── CLAUDE.md
├── .gitignore
├── agent/
│   ├── .env / .env.example
│   ├── .venv/
│   ├── hammerspoon/
│   │   ├── watcher.lua
│   │   ├── kakaotalk.lua
│   │   └── ax-dump.lua
│   ├── launchd/
│   │   ├── com.{username}.ingest-10min.plist
│   │   ├── com.{username}.ingest-3hr.plist
│   │   └── com.{username}.ingest-24hr.plist
│   ├── logs/
│   ├── prompts/
│   │   ├── 10min.md
│   │   ├── 3hr.md
│   │   └── 24hr.md
│   └── scripts/
│       ├── run-tier.sh
│       ├── telegram-bot.py
│       ├── kakaotalk-reader.swift
│       ├── calendar.sh
│       ├── backup.sh
│       └── dedup-ingest.sh
├── memory/
│   ├── AGENT.md
│   ├── USER.md
│   ├── schema/
│   │   ├── structure.md
│   │   ├── frontmatter.md
│   │   ├── data-flow.md
│   │   ├── domains.md
│   │   └── tools.md
│   ├── state/
│   │   ├── index.yaml
│   │   ├── priorities.yaml
│   │   ├── goals.yaml
│   │   ├── domains/
│   │   ├── people/
│   │   └── projects/
│   ├── knowledge/
│   │   ├── concepts/
│   │   ├── decisions/
│   │   ├── insights/
│   │   └── patterns/
│   ├── stream/
│   │   ├── daily/
│   │   ├── weekly/
│   │   ├── monthly/
│   │   └── sessions/
│   └── sources/
│       └── imports/
└── sources/
    ├── raw/
    ├── ingest/
    └── telegram/
```

---

## Components

### 1. Data Collection (Hammerspoon)

**watcher.lua** — 앱 전환, 시스템 이벤트, 유휴 상태를 추적하여 `sources/raw/activity-{YYYY-MM-DD}.md`에 기록.
- `hs.application.watcher`: 앱 포커스 → `[focus] {앱} — {윈도우제목}`
- `hs.caffeinate.watcher`: 잠금/해제, 덮개, 슬립/웨이크
- `hs.eventtap`: 10분 무입력 → `[idle]`
- 잠금/덮개 닫힘 시 `triggerSessionEndIngest()` → Claude ingest 트리거

**kakaotalk.lua** — 카카오톡 메시지를 30초 간격으로 자동 수집.
- `kakaotalk-reader` Swift 바이너리를 `hs.task`로 비동기 실행
- `_running` 플래그로 중복 실행 방지

**ax-dump.lua** — AX 트리 디버깅 유틸리티.
- `dump(appName, maxDepth)`: 앱 전체 AX 트리 덤프
- `dumpWindow(appName, windowIndex, maxDepth)`: 특정 윈도우만

**kakaotalk-reader.swift** — macOS AXUIElement API로 카카오톡 채팅방 메시지 추출.
- 윈도우별 AXScrollArea → AXTable → AXRow 순회
- 발화자 판별: Profile 버튼 + 이름 → 상대방, 우측 정렬 → 나
- SHA256 해시 기반 메시지 중복 제거 (`.seen_hashes`)
- 출력: `sources/raw/kakaotalk-{room}-{timestamp}.md`
- **주의**: 카카오톡 AX 트리는 버전에 따라 달라질 수 있음. ax-dump로 구조 확인 후 조정 필요.

→ 코드: [04-hammerspoon.md](attachments/04-hammerspoon.md), [05-kakaotalk-reader.md](attachments/05-kakaotalk-reader.md)

### 2. 3-Tier Processing Pipeline

각 tier는 Claude CLI로 실행. 접근 권한이 엄격히 분리됨.

**10min — 적재(Ingest)**
- 접근: raw 읽기, ingest 읽기+쓰기, telegram 읽기, **memory 접근금지**
- 절차: telegram 응답 확인 → raw 읽기 → 같은 room 그룹핑+기존 ingest 병합 → 정제+요약+태깅 → ingest 저장 → 긴급 시 telegram 알림

**3hr — 해석(Interpret)**
- 접근: ingest 읽기+삭제, telegram 읽기+삭제, memory 읽기전용, **daily만 쓰기**
- 절차: telegram 응답 확인 → ingest 읽기 → memory 컨텍스트 로드 → daily에 append(투두 업데이트 포함) → ingest 삭제 → 조건부 telegram 질문

**24hr — 성찰(Reflect)**
- 접근: memory 전체 읽기, state/knowledge/stream 쓰기, **sources 접근금지**
- 절차: 시스템 규칙 로드 → 어제 daily 읽기 → 관련 컨텍스트 탐색 → state/ 갱신 → knowledge/ 승격 → goals 진척 반영 → 압축(월요일=weekly, 1일=monthly) → index 정합성

**run-tier.sh** — 각 tier의 엔트리포인트. 인터넷 체크, 로그, 실패 시 원본 보존, 긴급 시 3hr 즉시 트리거.

→ 프롬프트: [07-prompts.md](attachments/07-prompts.md)
→ 스크립트: [06-scripts.md](attachments/06-scripts.md)

### 3. Communication (Telegram)

**telegram-bot.py** — 순수 Python (urllib만). 4개 명령:
- `send "메시지" ["선택지"...]`: 텍스트/인라인 키보드 전송
- `poll`: 새 업데이트 → `sources/telegram/{timestamp}-response.md` 또는 `-callback.md`
- `queue "메시지"`: outbox에 저장 (즉시 전송 안 함)
- `flush`: outbox 전송

→ 코드: [06-scripts.md](attachments/06-scripts.md)

### 4. Scheduler (launchd)

3개 plist:
- `com.{username}.ingest-10min.plist` — StartInterval 600
- `com.{username}.ingest-3hr.plist` — StartInterval 10800
- `com.{username}.ingest-24hr.plist` — StartCalendarInterval Hour=4, Minute=0

등록: plist를 `~/Library/LaunchAgents/`에 심볼릭 링크 후 `launchctl load`.

→ 코드: [08-launchd.md](attachments/08-launchd.md)

### 5. Memory Schema

**도메인은 사용자 정의**. 셋업 시 사용자에게 "삶에서 추적하고 싶은 영역이 뭔가요?"라고 물어서 도메인 목록을 결정한다. 도메인별로 `state/domains/{domain}.yaml`을 생성하고, `schema/domains.md`에 정의를 기록한다.

예시 도메인: career, finance, health, inner, relationships, learning (사용자에 따라 추가/변경/삭제)

스키마 파일 5개:
- `structure.md` — 디렉토리 구조, 계층 역할, 네이밍/규모 가이드라인
- `frontmatter.md` — 파일 유형별 YAML frontmatter 전체 필드 정의
- `data-flow.md` — 데이터 흐름, 압축 규칙 (daily→weekly 5:1, weekly→monthly 10:1)
- `domains.md` — 사용자 정의 도메인 목록, 추적 항목, health_score 기준
- `tools.md` — 검색 방법 가이드

→ 스키마: [03-memory-schema.md](attachments/03-memory-schema.md)
→ 코어 템플릿: [02-memory-core.md](attachments/02-memory-core.md)

### 6. Utility Scripts

- `backup.sh` — memory/ → iCloud Drive tar.gz, 최대 10개 롤링
- `dedup-ingest.sh` — SHA256 해시 기반 중복 제거
- `calendar.sh` — macOS Calendar AppleScript 조회 (today/tomorrow/week/N일/날짜)

→ 코드: [06-scripts.md](attachments/06-scripts.md)

---

## Implementation Order

모든 설치/실행은 **사용자 확인 후** 진행.

1. **사용자 인터뷰** — 플레이스홀더 값 확인, 도메인 목록 결정, Telegram Bot 정보 수집
2. **프로젝트 초기화** — 디렉토리 생성, git init, .gitignore, .env, Python venv
3. **메모리 스키마** — schema/*.md, AGENT.md, USER.md 템플릿, state/*.yaml 초기 구조
4. **Telegram Bot** — telegram-bot.py (다른 컴포넌트가 의존)
5. **데이터 수집** — Hammerspoon 스크립트 + kakaotalk-reader.swift 컴파일
6. **유틸리티** — backup.sh, dedup-ingest.sh, calendar.sh
7. **3-Tier 파이프라인** — 프롬프트 3개 + run-tier.sh
8. **스케줄러** — launchd plist 3개 등록
9. **Hammerspoon init.lua** — 모듈 require + start
10. **초기 데이터 + 온보딩** — USER.md 작성, goals 설정, 초기 채팅 데이터 임포트

---

## Setup Guide

구현 완료 후, 사용자에게 다음 가이드를 순서대로 안내한다.

### Step 1: Hammerspoon 설정

```
1. Hammerspoon이 설치되어 있는지 확인 (brew install --cask hammerspoon)
2. Hammerspoon 실행
3. 시스템 설정 > 개인정보 보호 및 보안 > 접근성 > Hammerspoon 체크박스 활성화
4. ~/.hammerspoon/init.lua에 아래 코드 추가:

   package.path = os.getenv("HOME") .. "/{project}/agent/hammerspoon/?.lua;" .. package.path
   local watcher   = require("watcher")
   local kakaotalk = require("kakaotalk")
   local axDump    = require("ax-dump")
   watcher.start()
   kakaotalk.startTimer(30)

5. Hammerspoon 메뉴바 아이콘 > Reload Config
```

### Step 2: 카카오톡 설정

```
1. 카카오톡 실행
2. 읽고 싶은 채팅방을 각각 별도 창으로 열기 (채팅방 더블클릭)
3. 카카오톡 Dock 아이콘 우클릭 > 옵션 > "Assign To: All Desktops" 선택
   → 이렇게 하면 어떤 데스크탑에서든 카카오톡 창이 보여서 AX API가 접근 가능
4. 열어둔 채팅방 창은 닫지 않고 유지 (최소화는 OK)
```

### Step 3: Telegram Bot 생성

```
1. Telegram에서 @BotFather 검색 → 대화 시작
2. /newbot 명령 입력
3. 봇 이름 입력 (예: "My Memory Agent")
4. 봇 username 입력 (예: "my_memory_agent_bot")
5. 발급된 토큰을 agent/.env의 TELEGRAM_BOT_TOKEN에 입력

Chat ID 확인:
1. 생성한 봇에게 아무 메시지 보내기
2. 브라우저에서 https://api.telegram.org/bot{토큰}/getUpdates 접속
3. 응답 JSON에서 "chat":{"id": 숫자} 의 숫자가 Chat ID
4. agent/.env의 TELEGRAM_CHAT_ID에 입력

테스트:
python3 agent/scripts/telegram-bot.py send "테스트 메시지"
```

### Step 4: 초기 데이터 임포트

시스템이 돌아가기 전에 기존 데이터를 넣고 싶다면:

**카카오톡 기존 채팅 내보내기:**
```
1. 카카오톡 채팅방 > 설정(≡) > 대화 내보내기
2. 내보낸 .txt 파일을 sources/raw/ 에 .md 확장자로 저장
3. 수동으로 run-tier.sh 10min 실행하여 ingest로 변환
```

**기존 문서/노트 임포트:**
```
1. 기존 파일을 memory/sources/imports/에 저장
2. Claude Code에서 다음 프롬프트 실행:
   "memory/sources/imports/ 에 있는 파일들을 읽고,
    memory/AGENT.md 규칙에 따라 적절한 memory/ 위치에 정리해줘.
    stream/daily/에 기록하고, 필요하면 state/와 knowledge/도 업데이트."
```

### Step 5: 온보딩 프롬프트

시스템 구축 후 Claude Code에서 다음 프롬프트를 순서대로 실행한다:

**1) USER.md 작성:**
```
memory/USER.md를 작성해줘.
나에 대해 물어보고, 답변을 바탕으로 프로파일을 채워줘.
포함할 내용: 정체성, 성격 특성, 인지 특성, 에너지 관리, 현재 상태.
```

**2) 도메인 설정:**
```
memory/schema/domains.md를 보고, 내 도메인을 설정해줘.
내가 추적하고 싶은 삶의 영역을 물어보고,
각 도메인별로 state/domains/{domain}.yaml을 생성해줘.
```

**3) 목표 설정:**
```
memory/state/goals.yaml을 채워줘.
10년 뒤 어떤 사람이 되고 싶은지,
그걸 위한 마일스톤 체인은 뭔지 물어보고 작성해줘.
```

**4) 첫 번째 수동 실행:**
```
agent/scripts/run-tier.sh 10min 을 실행해봐.
sources/raw/에 파일이 있으면 처리되는지 확인.
문제가 있으면 로그(agent/logs/10min.log)를 보고 디버깅.
```

**5) launchd 등록 (자동화 시작):**
```
launchd plist를 등록해서 자동화를 시작해줘.
agent/launchd/의 plist 파일을 ~/Library/LaunchAgents/에 심볼릭 링크하고
launchctl load로 등록.
등록 후 launchctl list | grep ingest 로 확인.
```

---

## Attachment Index

| 파일 | 내용 |
|------|------|
| [01-config.md](attachments/01-config.md) | CLAUDE.md, .gitignore, .env.example |
| [02-memory-core.md](attachments/02-memory-core.md) | AGENT.md 템플릿, USER.md 템플릿, goals/priorities/index.yaml 초기 구조 |
| [03-memory-schema.md](attachments/03-memory-schema.md) | structure.md, frontmatter.md, data-flow.md, domains.md, tools.md |
| [04-hammerspoon.md](attachments/04-hammerspoon.md) | watcher.lua, kakaotalk.lua, ax-dump.lua |
| [05-kakaotalk-reader.md](attachments/05-kakaotalk-reader.md) | kakaotalk-reader.swift |
| [06-scripts.md](attachments/06-scripts.md) | run-tier.sh, telegram-bot.py, backup.sh, dedup-ingest.sh, calendar.sh |
| [07-prompts.md](attachments/07-prompts.md) | 10min.md, 3hr.md, 24hr.md |
| [08-launchd.md](attachments/08-launchd.md) | 3개 launchd plist |

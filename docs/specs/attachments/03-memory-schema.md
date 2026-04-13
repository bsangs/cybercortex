# 03 — Memory Schema Files

## memory/schema/structure.md

```markdown
# Memory 디렉토리 구조 및 계층 역할

## 전체 디렉토리 트리

​```
memory/
├── AGENT.md                     ← 에이전트 행동 규칙 (< 300줄, 매 세션 로드)
├── USER.md                      ← 사용자 프로파일 (~200줄, 매 세션 로드)
├── state/                       ← "지금" (매 세션 최우선 로드)
│   ├── index.yaml               ← 전체 파일 카탈로그 + 한 줄 요약
│   ├── priorities.yaml          ← 태스크, 미결 결정, 블로커, 알림
│   ├── goals.yaml               ← 10년 목표, 체인 마일스톤
│   ├── domains/                 ← 라이프 도메인 현재 상태 (사용자 정의)
│   ├── people/                  ← 인물별 현재 프로파일
│   └── projects/                ← 프로젝트별 현재 상태
├── stream/                      ← "무슨 일이 있었나" (시간순, 에피소드)
│   ├── daily/YYYY-MM-DD.md
│   ├── weekly/YYYY-WNN.md
│   ├── monthly/YYYY-MM.md
│   └── sessions/               ← AI 세션 요약
├── knowledge/                   ← "영구적으로 유효한 것" (의미 기반)
│   ├── insights/
│   ├── decisions/
│   ├── patterns/
│   └── concepts/
├── sources/                     ← 불변 원본 데이터
│   └── imports/
└── schema/                      ← 상세 참조 문서 (필요 시 로드)
​```

## 계층 역할 테이블

| 계층 | 로드 시점 | 규모 | 수명 | 인지과학 대응 |
|------|-----------|------|------|---------------|
| AGENT.md + USER.md | 매 세션 자동 | < 300 + ~200줄 | 거의 변하지 않음 | 절차 기억 (Procedural memory) |
| state/ | 매 세션 최우선 로드 (index → 선택적) | ~25개 파일 | 지속 업데이트 | 작업 기억 (Working memory) |
| stream/ | 온디맨드 검색 | 무한 증가 → 압축 | daily→weekly→monthly | 일화 기억 (Episodic memory) |
| knowledge/ | 온디맨드 검색 | ~수백 개 | 영구 | 의미 기억 (Semantic memory) |
| sources/ | 후처리 참조 전용 | 대용량 가능 | 불변, 삭제 가능 | 감각 버퍼 (Sensory buffer) |
| schema/ | 온디맨드 참조 | ~5개 파일 | 구조 변경 시에만 | 절차 기억 (Procedural memory) |

### 파일 확장자 규칙
- state/people/*.md, state/projects/*.md — Markdown (YAML frontmatter + body)
- state/domains/*.yaml — 순수 YAML
- state/index.yaml, state/priorities.yaml, state/goals.yaml — 순수 YAML (frontmatter 없음)
- stream/*.md — Markdown
- knowledge/*.md — Markdown

## 각 계층 상세 설명

### AGENT.md + USER.md (최상위 파일)

- **AGENT.md**: 에이전트의 행동 규칙, 프로토콜, 의사결정 원칙을 정의. 300줄 이내로 유지하며, 상세 내용은 schema/로 위임.
- **USER.md**: 사용자의 성격, 선호, 컨텍스트, 커뮤니케이션 스타일 기록. 에이전트가 사용자를 이해하는 기반.
- 두 파일 모두 매 세션 시작 시 자동 로드되므로, 간결함이 핵심.

### state/ (현재 상태 계층)

- **역할**: 현재 시점의 모든 상태를 반영. "지금 어떤 상태인가?"에 대한 답.
- **index.yaml**: 전체 memory/ 파일 카탈로그. 파일 경로 + 한 줄 요약.
- **priorities.yaml**: 현재 진행 중인 태스크, 미결 결정, 블로커, 긴급 알림.
- **goals.yaml**: 장기 목표(10년)와 중간 마일스톤(체인). 방향성 판단의 기준.
- **domains/**: 사용자가 정의한 라이프 도메인의 현재 건강도와 상태.
- **people/**: 중요 인물별 현재 관계 상태, 마지막 연락, 메모.
- **projects/**: 진행 중인 프로젝트의 현재 상태, 우선순위, 블로커.
- **원칙**: state/는 항상 현재를 반영. 히스토리를 남기지 않음. 과거는 stream/에서 추적.

### stream/ (시간 흐름 계층)

- **역할**: 시간순으로 기록된 일화. "언제 무슨 일이 있었나?"에 대한 답.
- **daily/**: 하루 단위 기록. `YYYY-MM-DD.md`. 모든 이벤트의 최초 진입점.
- **weekly/**: 주간 압축본. `YYYY-WNN.md`. 매주 월요일 5:1 비율로 압축.
- **monthly/**: 월간 압축본. `YYYY-MM.md`. 매월 1일 10:1 비율로 압축.
- **sessions/**: AI 세션 요약.
- **압축 규칙**: schema/data-flow.md 참조.

### knowledge/ (지식 계층)

- **역할**: 시간에 독립적인 영구 지식.
- **insights/**: 깨달음, 교훈, 관찰에서 도출된 원칙.
- **decisions/**: 중요 의사결정 기록과 그 맥락, 결과.
- **patterns/**: 반복되는 행동/사고 패턴과 대응책.
- **concepts/**: 개념 정의, 프레임워크, 멘탈 모델.
- **승격 기준**: stream/에서 시간 독립적 가치가 확인되면 knowledge/로 승격.

### sources/ (원본 데이터 계층)

- **역할**: 가공 전 원본 데이터 보관. 참조용.
- **imports/**: 외부에서 가져온 원본.
- **원칙**: 불변(immutable). 가공 완료 후 삭제 가능.

### schema/ (참조 문서 계층)

- **역할**: memory 시스템 자체의 상세 스펙 문서.
- **포함 파일**: structure.md, frontmatter.md, data-flow.md, domains.md, tools.md

## 파일 네이밍 규칙

| 계층 | 네이밍 패턴 | 예시 |
|------|-------------|------|
| state/domains/ | `{domain-name}.yaml` | `career.yaml` |
| state/people/ | `{person-name}.md` | `홍길동.md` |
| state/projects/ | `{project-slug}.md` | `side-project-a.md` |
| stream/daily/ | `YYYY-MM-DD.md` | `2026-04-10.md` |
| stream/weekly/ | `YYYY-WNN.md` | `2026-W15.md` |
| stream/monthly/ | `YYYY-MM.md` | `2026-04.md` |
| stream/sessions/ | `YYYY-MM-DD-NNN.md` | `2026-04-10-001.md` |
| knowledge/*/ | 의미 기반 슬러그 | `burnout-cycle.md` |
| sources/imports/ | 원본 파일명 유지 | `chatgpt-export.json` |

## 파일 규모 가이드라인

| 파일 유형 | 목표 크기 | 초과 시 조치 |
|-----------|-----------|-------------|
| state/ 개별 파일 | < 100줄 | 불필요 히스토리 제거 |
| stream/daily/ | < 200줄 | 핵심만 남기고 간소화 |
| stream/weekly/ | < 100줄 | daily 5일분의 5:1 압축 |
| stream/monthly/ | < 50줄 | weekly 4~5주분의 10:1 압축 |
| knowledge/ 개별 파일 | < 150줄 | 개념 분리 또는 요약 |
| index.yaml | 파일 수에 비례 | 삭제된 파일 항목 제거 |
```

---

## memory/schema/frontmatter.md

```markdown
# Frontmatter 스펙

모든 memory/ 파일은 YAML frontmatter를 포함한다. 이 문서는 파일 유형별 전체 frontmatter 필드를 정의한다.

## 공통 필드 (모든 파일)

​```yaml
type: state | stream | knowledge | source
subtype: person | project | domain | daily | weekly | monthly | session | insight | decision | pattern | concept
created: "2026-04-10T14:30:00+09:00"  # ISO 8601, 타임존 필수 (+09:00)
updated: "2026-04-10T15:00:00+09:00"  # ISO 8601, 타임존 필수
confidence: 0.85                       # 0.0~1.0, 정보 신뢰도
source: session | import | merge       # 데이터 출처
related: []                            # 관련 파일 경로 목록
tags: []                               # 자유 태그
​```

### 공통 필드 상세

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| type | enum | Y | 파일이 속한 계층 |
| subtype | enum | Y | 파일의 세부 유형 |
| created | string | Y | 최초 생성 시각. 한번 설정 후 변경 금지 |
| updated | string | Y | 마지막 수정 시각. 내용 변경 시 반드시 갱신 |
| confidence | float | N | 정보 신뢰도. 기본값 0.7 |
| source | enum | Y | session=대화, import=외부, merge=압축/병합 |
| related | list | N | 관련 파일의 상대 경로 |
| tags | list | N | 자유 태그 |

### 예외: 순수 YAML 파일
state/index.yaml, state/priorities.yaml, state/goals.yaml은 frontmatter가 아닌 최상위 YAML 구조를 사용하며 last_updated 필드로 최신성을 추적한다.

### confidence 가이드라인

| 값 범위 | 의미 | 예시 |
|---------|------|------|
| 0.9~1.0 | 확인된 사실 | 공식 문서, 직접 경험 확인 |
| 0.7~0.8 | 높은 확신 | 본인 진술, 신뢰할 수 있는 출처 |
| 0.5~0.6 | 보통 | 간접 정보, 추론 |
| 0.3~0.4 | 불확실 | 오래된 정보, 편향 가능성 |
| 0.0~0.2 | 매우 불확실 | 추측, 미확인 |

---

## state/ 계층 파일별 추가 필드

### state/people/ (인물 프로파일)

​```yaml
type: state
subtype: person
# ... 공통 필드 ...
name: "이름"
role: "직함/역할"
relation: mentor | partner | colleague | family | friend | acquaintance
importance: 9                           # 1~10
last_contact: "2026-04-08"             # YYYY-MM-DD
​```

### state/domains/ (도메인 상태)

​```yaml
type: state
subtype: domain
# ... 공통 필드 ...
domain: career                          # 사용자 정의 도메인 ID
health_score: 7                         # 1~10
trend: improving                        # declining | stable | improving
last_reviewed: "2026-04-10"
​```

### state/projects/ (프로젝트 상태)

​```yaml
type: state
subtype: project
# ... 공통 필드 ...
name: "프로젝트 이름"
status: active                          # active | paused | completed | dropped
role: "본인의 역할"
priority: high                          # high | medium | low
​```

---

## stream/ 계층 파일별 추가 필드

### stream/daily/ (일일 기록)

​```yaml
type: stream
subtype: daily
# ... 공통 필드 ...
date: "2026-04-10"
mood: 7                                 # 1~10, 비어있을 수 있음
energy: 6                               # 1~10, 비어있을 수 있음
exercise: false                         # 운동 여부
medication: true                        # 약 복용 여부 (해당되는 경우)
​```

### stream/weekly/ (주간 압축)

​```yaml
type: stream
subtype: weekly
# ... 공통 필드 ...
date_range: "2026-04-07 ~ 2026-04-13"
mood_avg: 6.5
energy_avg: 5.8
exercise_count: 3                       # 7일 중 실행 일수
medication_count: 7
​```

### stream/monthly/ (월간 압축)

​```yaml
type: stream
subtype: monthly
# ... 공통 필드 ...
month: "2026-04"
mood_avg: 6.2
energy_avg: 5.5
exercise_rate: 0.43                     # 실행일 / 전체일
medication_rate: 1.0
​```

### stream/sessions/ (세션 요약)

​```yaml
type: stream
subtype: session
# ... 공통 필드 ...
session_id: "2026-04-10-001"
duration_minutes: 45
topics: ["커리어 논의", "프로젝트 진행"]
decisions_made: []
action_items: []
​```

---

## knowledge/ 계층 파일별 추가 필드

### knowledge/decisions/ (의사결정 기록)

​```yaml
type: knowledge
subtype: decision
# ... 공통 필드 ...
domain: career
decision: "결정 내용 한 줄 요약"
date: "2026-04-10"
context: "결정 배경"
outcome: pending                        # positive | negative | pending
outcome_detail: ""
​```

### knowledge/insights/ (인사이트)

​```yaml
type: knowledge
subtype: insight
# ... 공통 필드 ...
domain:
  - career
  - learning
durability: permanent                   # permanent | seasonal | uncertain
​```

### knowledge/patterns/ (패턴)

​```yaml
type: knowledge
subtype: pattern
# ... 공통 필드 ...
domain:
  - inner
  - health
frequency: weekly                       # daily | weekly | monthly | situational
severity: high                          # high | medium | low
countermeasure: "대응 방법"
​```

### knowledge/concepts/ (개념)

​```yaml
type: knowledge
subtype: concept
# ... 공통 필드 ...
domain:
  - inner
origin: mentor                          # mentor | self | external
​```

---

## Frontmatter 작성 원칙

1. **타임존 필수**: 모든 시간 필드에 +09:00 (KST) 포함
2. **updated 갱신**: 내용 변경 시 반드시 updated 필드 갱신
3. **created 불변**: created는 최초 생성 시 한번만 설정
4. **confidence 정직**: 불확실하면 낮게 설정
5. **related 양방향**: A가 B를 related에 포함하면, B도 A를 포함
6. **source 정확**: merge는 압축/병합으로 생성된 경우에만
```

---

## memory/schema/data-flow.md

```markdown
# 데이터 흐름 및 압축 규칙

## 입력 → 계층 → 최종 위치

​```
┌──────────────────────────────────────────────────┐
│                     입력 채널                       │
├──────────────┬──────────────┬───────────────────┤
│  대화 세션     │  외부 가져오기  │  예약 작업          │
└──────┬───────┴──────┬───────┴────────┬──────────┘
       │              │                │
       ▼              ▼                ▼
┌──────────────────────────────────────────────────────────────┐
│                    1차 기록 (stream/daily/)                    │
└──────────────────────────┬───────────────────────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   state/     │  │  knowledge/  │  │  sources/    │
│  동시 업데이트  │  │  승격 대상     │  │  원본 보관    │
└──────────────┘  └──────────────┘  └──────────────┘
​```

​```
시간 흐름에 따른 압축:

stream/daily/     →     stream/weekly/     →     stream/monthly/
(매일 기록)              (매주 월요일 압축)          (매월 1일 압축)
                         5:1 비율                  10:1 비율
​```

---

## 5대 흐름 규칙

### 규칙 1: 모든 이벤트 → stream/daily/ 먼저
모든 의미 있는 이벤트는 해당 날짜의 stream/daily/YYYY-MM-DD.md에 먼저 기록한다.

### 규칙 2: 상태 변경 → state/ 동시 업데이트
이벤트가 현재 상태를 변경하면, daily에 기록함과 동시에 state/의 해당 파일도 업데이트한다.

### 규칙 3: 시간 독립 지식 → knowledge/로 승격
승격 판단 기준:
- "이 정보는 1년 후에도 유효한가?" → Yes이면 승격
- "특정 날짜를 빼도 의미가 있는가?" → Yes이면 승격

### 규칙 4: daily → weekly 압축 (5:1 비율, 매주 월요일)
- 직전 주 7일분의 daily를 1개의 weekly로 압축
- 파일: stream/weekly/YYYY-WNN.md
- daily 원본은 삭제하지 않음

weekly 구조:
​```markdown
# 주간 요약: YYYY-WNN

## 핵심 이벤트
- (가장 중요한 3~5개 이벤트)

## 도메인별 변화
- (변화가 있었던 도메인만)

## 수치 요약
- mood 평균: X.X, 범위: X~X
- energy 평균: X.X
- exercise: X/7일
- medication: X/7일

## 주요 결정/인사이트
- (knowledge/로 승격된 항목 참조)

## 다음 주 주의 사항
​```

### 규칙 5: weekly → monthly 압축 (10:1 비율, 매월 1일)
- 직전 월의 weekly 4~5개를 1개의 monthly로 압축
- 파일: stream/monthly/YYYY-MM.md
- weekly 원본은 삭제하지 않음

monthly 구조:
​```markdown
# 월간 요약: YYYY-MM

## 한 줄 요약

## 핵심 이벤트 (최대 5개)

## 도메인 건강도 변화
| 도메인 | 월초 | 월말 | 추세 |

## 월간 수치
- mood 평균, energy 평균, exercise_rate, medication_rate

## 핵심 결정

## 학습/성장
​```

---

## 압축 시 보존/폐기 규칙

### 절대 폐기 금지 항목

| 항목 | 이유 |
|------|------|
| 의사결정 기록 | 미래 참조 필수 |
| 패턴 발견 | 자기 인식의 핵심 |
| 인사이트/깨달음 | 영구 지식 |
| mood/energy 변곡점 | 급변 원인 추적 |
| 갈등/화해 이력 | 관계 컨텍스트 |
| 건강 이상 신호 | 패턴 추적 |

### 폐기 가능 항목

| 항목 | 조건 |
|------|------|
| 완료된 단순 태스크 | 교훈이 없는 경우 |
| 루틴 기록 | 특이사항 없는 평범한 날 |
| 원본 소스 데이터 | 가공 완료 |
| 중간 과정 메모 | 최종 결과 확정 |
```

---

## memory/schema/domains.md

```markdown
# 라이프 도메인 정의

도메인은 사용자가 직접 정의한다. 셋업 시 "삶에서 추적하고 싶은 영역"을 물어서 결정.
각 도메인별로 state/domains/{domain}.yaml을 생성한다.

## 도메인 정의 방법

각 도메인에 대해 다음을 정의:
1. **ID**: 영문 소문자 슬러그 (예: career, health)
2. **이름**: 한글 표시명 (예: 커리어, 건강)
3. **한 줄 설명**: 이 도메인이 추적하는 것
4. **추적 항목**: YAML 형태의 tracking 리스트
5. **health_score 가이드**: 1~10 점수별 상태 설명

## 도메인 예시

아래는 예시이며, 사용자가 자유롭게 추가/수정/삭제한다.

### career (커리어)

| 점수 | 상태 | 설명 |
|------|------|------|
| 1~3 | 위기 | 실직, 번아웃, 방향 상실 |
| 4~6 | 불안정 | 병목, 정체 |
| 7~9 | 양호 | 성장 중, 성과 달성 |
| 10 | 최적 | 목표 달성, 자유도 높음 |

tracking: active_projects, role, positioning, salary, side_projects, branding

### finance (재정)

| 점수 | 상태 | 설명 |
|------|------|------|
| 1~3 | 위기 | 빚 증가, 런웨이 < 1개월 |
| 4~6 | 불안정 | 빚 상환 중, 저축 없음 |
| 7~9 | 안정 | 빚 해소, 투자 진행 |
| 10 | 자유 | 패시브 인컴, 순자산 목표 |

tracking: income, expenses, debt, investments, net_worth, runway_months

### health (건강)

| 점수 | 상태 | 설명 |
|------|------|------|
| 1~3 | 위기 | 수면 붕괴, 약 미복용 |
| 4~6 | 불안정 | 불규칙 수면, 운동 미실행 |
| 7~9 | 관리 중 | 약 복용, 수면 루틴, 운동 |
| 10 | 최적 | 전영역 안정 |

tracking: medication, sleep_pattern, exercise, diet, weight, mental_health

### inner (내면)

| 점수 | 상태 | 설명 |
|------|------|------|
| 1~3 | 위기 | 감정 폭발 빈발, 방어기제 지배 |
| 4~6 | 인지 중 | 패턴 인식하지만 통제 어려움 |
| 7~9 | 전환 중 | 건강한 대응 우세 |
| 10 | 안정 | 자기 조절 기본값 |

tracking: emotional_patterns, mode_transitions, cognitive_distortions

### relationships (관계)

| 점수 | 상태 | 설명 |
|------|------|------|
| 1~3 | 위기 | 관계 단절, 심각한 갈등 |
| 4~6 | 불안정 | 소통 부족, 미해결 갈등 |
| 7~9 | 양호 | 정기 소통, 갈등 해소 |
| 10 | 충실 | 깊은 교류, 상호 성장 |

tracking: partner_status, family_contact, key_relationships, conflicts

### learning (학습)

| 점수 | 상태 | 설명 |
|------|------|------|
| 1~3 | 고갈 | 인풋 0 |
| 4~6 | 부족 | 간헐적 인풋 |
| 7~9 | 성장 중 | 다채널 인풋, 자체 인사이트 |
| 10 | 자립 | 지식 생산 |

tracking: input_pipeline, books_read, content_consumed, wisdom_path

## 도메인 간 상호작용

도메인은 독립적이지 않다. 예시:

​```
health ──→ inner ──→ relationships
  │          │           │
  │          ▼           │
  └──→ career ←──────────┘
          │
          ▼
       finance ←── learning
​```

## 도메인 리뷰 주기 권장

| 도메인 | 권장 | 이유 |
|--------|------|------|
| health | 매일 | 약 복용, 수면, 운동은 일일 추적 |
| inner | 주 2~3회 | 감정 패턴 빈번 |
| career | 주 1회 | 프로젝트 진행 상황 |
| relationships | 주 1회 | 핵심 관계 소통 |
| finance | 월 1회 | 월간 수입/지출 정산 |
| learning | 월 1회 | 인풋 파이프라인 점검 |
```

---

## memory/schema/tools.md

```markdown
# 도구 및 검색 방법

## 파일 네비게이션

### state/index.yaml 기반 탐색 (매 세션 기본 경로)

1. 세션 시작 시 state/index.yaml을 먼저 읽는다.
2. 전체 파일 목록과 한 줄 요약을 스캔한다.
3. 현재 맥락에 필요한 파일만 선택적으로 읽는다.

​```
세션 시작
  → AGENT.md 로드 (자동)
  → USER.md 로드 (자동)
  → state/index.yaml 로드
  → 필요한 state/ 파일 선택 로드
  → 필요 시 stream/, knowledge/ 온디맨드 검색
​```

### 시간 기반 탐색 (stream/)

​```
특정 날짜 → stream/daily/YYYY-MM-DD.md
특정 주   → stream/weekly/YYYY-WNN.md
특정 월   → stream/monthly/YYYY-MM.md
특정 세션 → stream/sessions/YYYY-MM-DD-NNN.md
​```

### 도메인 기반 탐색

​```
1. state/domains/{domain}.yaml          ← 현재 상태
2. grep으로 관련 키워드 검색             ← 관련 기록
3. knowledge/ 에서 domain 필드 검색      ← 영구 지식
​```

### 인물 기반 탐색

​```
1. state/people/{person}.md             ← 현재 프로파일
2. grep으로 이름 검색                    ← 관련 기록
3. related 필드 따라가기                  ← 연관 파일
​```

---

## grep 구조화 필드 검색

### 자주 사용하는 검색 패턴

​```bash
# 특정 도메인의 모든 파일
rg "^domain:.*career" memory/

# 특정 상태의 프로젝트
rg "^status: active" memory/state/projects/

# 높은 중요도 인물
rg "^importance: (9|10)" memory/state/people/

# 미결 의사결정
rg "^outcome: pending" memory/knowledge/decisions/

# 높은 심각도 패턴
rg "^severity: high" memory/knowledge/patterns/

# 특정 confidence 이하 파일
rg "^confidence: 0\.[0-4]" memory/

# trend가 declining인 도메인
rg "^trend: declining" memory/state/domains/
​```

---

## 복합 검색 전략

### 전략 1: 넓은 탐색 → 좁히기

​```
1. grep으로 키워드 검색          ← 관련 파일 식별
2. 결과 파일을 직접 읽기         ← 상세 내용 확인
3. related 필드 따라가기        ← 연관 파일 탐색
​```

### 전략 2: 시간 역추적

​```
1. state/ 에서 현재 상태 확인     ← "지금"
2. stream/weekly/ 역순 탐색      ← "최근 변화"
3. stream/daily/ 특정 날짜 확인  ← "그 날의 상세"
4. knowledge/ 에서 관련 인사이트  ← "영구 지식"
​```

---

## macOS 캘린더 조회

​```bash
calendar.sh              # 오늘
calendar.sh today        # 오늘
calendar.sh tomorrow     # 내일
calendar.sh week         # 이번 주 (7일)
calendar.sh 3            # 오늘부터 3일
calendar.sh 2026-04-15   # 특정 날짜
​```

출력: `캘린더명 | 일정명 | 시작~종료 | 장소 | 설명`

---

## 파일 생성/수정 체크리스트

1. [ ] frontmatter 필수 필드 모두 포함
2. [ ] created/updated 타임스탬프 설정 (+09:00)
3. [ ] state/index.yaml에 항목 추가/갱신
4. [ ] 양방향 related 링크 설정
```

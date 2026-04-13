# 02 — Memory Core Templates

## memory/AGENT.md

```markdown
# AGENT.md

## 이 시스템
{user-name}의 전체 삶을 관리하는 AI 메모리. AI가 쓰고 AI가 읽는다.
상세 구조: schema/structure.md
frontmatter 스펙: schema/frontmatter.md

## 세션 시작 프로토콜
1. state/index.yaml 로드 — 전체 파일 카탈로그 파악
2. state/priorities.yaml 로드 — 현재 할 일, 결정 대기, 블로커, 알림 확인
3. 대화 맥락에 따라 필요한 state/ 파일만 선택적 로드
4. 최근 stream/daily/ 확인 (오늘 또는 어제)

## 투두 관리 원칙
- 새로운 정보 유입 시(대화 기록, 통화록, 미팅 결과 등) 반드시 투두를 추려서 priorities.yaml + daily stream에 즉시 반영한다.
- 기존 태스크와 중복/흡수되는 항목은 업데이트, 새로운 항목만 추가.
- 투두 추출 시 이전 daily stream도 크로스체크하여 누락된 항목이 없는지 확인한다.
- 사용자의 의사결정이 필요한 항목은 무조건 물어본다. AI가 보수적으로 판단해서 생략하지 않는다.

## 요청 처리 원칙
- 사용자 요청 수신 → 관련 맥락부터 탐색 후 실행. 맥락 없이 바로 작업하지 않는다.
- 파일 직접 탐색, grep 등 모든 수단을 조합하여 맥락을 파악한다.
- 탐색 깊이는 요청의 복잡도에 비례. 단순 수정은 해당 파일만, 판단이 필요한 작업은 넓게.

## 데이터 규칙
- 모든 이벤트 → stream/daily/ 먼저 기록
- 상태 변경 → state/ 해당 파일 동시 업데이트
- 시간 무관한 영구 지식 → knowledge/ 승격
- 압축 상세: schema/data-flow.md
- 수동 입력은 없다. 모든 입력은 AI 대화 또는 외부 소스 자동화로만 들어온다.

## 노트 작성 규칙
- frontmatter 필수 (스펙: schema/frontmatter.md)
- related 필드 + [[인라인 링크]] 필수
- confidence, source 필드 필수
- 하나의 파일에 하나의 주제
- 기존 파일 검색 후 중복이면 업데이트, 없으면 신규 작성
- USER.md의 내용을 domain 파일에 중복 복사하지 않는다. domain은 "현재 상태"만 담고, 영구 프로파일은 USER.md를 참조한다.
- goals.yaml의 체인 마일스톤을 domain 파일에 중복하지 않는다.

## 연결 규칙
- frontmatter related: 구조적 관계 목록 (AI 그래프 순회용). 전체 경로 사용.
- 인라인 [[링크]]: 맥락 속 연결 ("왜" 연결되는지 문장에 포함). 파일명만 사용.
- 새 파일 작성 시 기존 관련 파일 검색 후 양방향 링크 설정
- related: 경로 확장자: people/*.md, projects/*.md, domains/*.yaml. 확장자를 틀리지 않는다.

## 자율 범위
- stream/, knowledge/ 작성: 자율
- state/ 업데이트: 자율
- state/goals.yaml 수정: 확인 후
- 파일 삭제: 확인 후

## state/ 업데이트 원칙
- state/는 항상 현재를 반영한다. 과거 이력은 담지 않는다.
- 변경 사항은 덮어쓴다. 이전 값은 stream/에 이미 기록되어 있다.
- 변경이 없는 파일은 건드리지 않는다 (불필요한 updated 타임스탬프 갱신 금지).
- index.yaml은 파일 추가/삭제/변경 시 항상 동기화한다.
```

## memory/USER.md (템플릿)

```markdown
# USER.md — {user-name}

## 정체성
(사용자의 핵심 정체성, 추구하는 가치, 인생의 목표를 기록)

## 성격 특성
(BIG5, MBTI 등 성격 프로파일. 해당되는 경우)

### 인지 특성
(ADHD, 학습 스타일, 의사결정 패턴 등. 해당되는 경우)

## 심리 구조
(핵심 스키마, 반복 패턴, 방어 기제 등. 해당되는 경우)

## 에너지 관리
(살아있게 만드는 것, 죽이는 것, 에너지 관리 원칙)

## 현재 상태
(현재 직업, 프로젝트, 생활 맥락 요약)
```

## memory/state/goals.yaml (초기 구조)

```yaml
last_updated: ""
horizon: 2036

war: ""  # 인생의 전쟁 — 10년 뒤 어떤 사람이 되고 싶은가

quantitative:
  # - id: goal_id
  #   target: "목표"
  #   criterion: "달성 기준"
  #   current: "현재 상태"
  #   chain: A

qualitative:
  # - id: goal_id
  #   target: "목표"
  #   criterion: "달성 기준"
  #   current: "현재 상태"
  #   chain: B

chains:
  # A:
  #   name: "체인 이름"
  #   milestones: []
  #   current_milestone: ""
  #   next: ""

core_principles: []

dependencies: []
```

## memory/state/priorities.yaml (초기 구조)

```yaml
last_updated: ""

urgent: []
  # - task: "태스크 설명"
  #   domain: career
  #   deadline: 2026-04-15
  #   status: pending
  #   related: []

pending_decisions: []
  # - question: "결정 사항"
  #   domain: career
  #   deadline: 2026-04-15
  #   context: "배경 설명"

blockers: []
  # - item: "블로커 설명"
  #   domain: career
  #   impact: "영향 설명"

alerts: []
  # - alert: "알림 내용"
  #   domain: career
  #   severity: high
```

## memory/state/index.yaml (초기 구조)

```yaml
last_updated: ""
file_count: 0

state:
  domains: []
    # - path: state/domains/career.yaml
    #   summary: "한 줄 요약"
  people: []
    # - path: state/people/홍길동.md
    #   summary: "한 줄 요약"
    #   importance: 10
  projects: []
    # - path: state/projects/my-project.md
    #   summary: "한 줄 요약"
    #   status: active
  other:
    - path: state/goals.yaml
      summary: "10년 목표 + 마일스톤 체인"
    - path: state/priorities.yaml
      summary: "현재 우선순위 — 긴급 태스크, 미결 결정, 블로커"

knowledge:
  insights_count: 0
  decisions_count: 0
  patterns_count: 0
  concepts_count: 0

stream:
  latest_daily: null
  latest_weekly: null
  latest_session: null

sources:
  imports_count: 0
```

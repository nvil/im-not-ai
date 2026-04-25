# Pattern Candidates Pool (v1.3~)

분류 체계 본진(`ai-tell-taxonomy.md`)에 승격되기 전, 에이전트들이 실전에서 발견한 "AI 티 의심 패턴" 후보를 단일 그릇으로 누적하는 곳. taxonomist가 이 풀을 주기적으로 점검해 v1.x로 승격하거나 기각한다.

## 역할 분담

- **detector·rewriter·naturalness-reviewer**: 미분류 의심 span을 발견하면 본 풀에 후보로 적재(또는 기존 후보의 `occurrences`를 +1).
- **korean-ai-tell-taxonomist**: 풀 운영자. 재현 2회 이상·심각도 일관 후보를 본진으로 승격, 부적합·중복은 기각.

본 풀은 SSOT가 아니라 **승격 전 작업대**다. 탐지기·윤문가는 풀의 후보를 직접 적용하지 않는다 — 본진(`ai-tell-taxonomy.md`)에 승격된 패턴만 운영 규칙이다.

## 후보 스키마

각 후보는 다음 YAML 항목으로 풀에 누적된다.

```yaml
- id: "cand-A-2026-001"      # 임시 ID. 형식: cand-{대분류 힌트}-{YYYY}-{NNN}
  pattern_label: "한 줄 라벨"  # taxonomy 본진 항목명과 동일 톤
  proposed_category: "A | B | C | D | E | F | G | H | I | J"
  proposed_severity: "S1 | S2 | S3"
  description: |
    이 패턴이 무엇이고 왜 AI 티인지 1~3줄 설명.
  signature_examples:          # 실제 발견된 원문 span (최소 1건, 승격 시 2건+)
    - text: "원문 span"
      context: "한 문장 정도의 주변 맥락"
      source_run_id: "2026-04-25-001"   # _workspace/{run_id}/01_input.txt에서 발견
      discovered_by: "ai-tell-detector | korean-style-rewriter | naturalness-reviewer | external"
      discovered_at: "2026-04-25"
  suggested_fix_draft: "윤문 처방 초안 (있을 때만)"
  occurrences: 1               # 동일 패턴이 다른 run에서 재현될 때마다 +1
  status: "pending"            # pending | promoted | rejected | merged
  status_reason: ""            # 기각·병합 시 사유 + 머지된 본진 ID (있을 때)
  created_at: "2026-04-25"
  last_seen_at: "2026-04-25"
  reviewed_by_taxonomist: false
```

## 적재 절차 (에이전트용)

1. **중복 검사**: 새 후보를 추가하기 전 풀의 기존 `pending` 항목을 훑어 동일 패턴이 있는지 확인. 같은 패턴이면 신규 항목 생성 대신 다음을 갱신:
   - `occurrences` +1
   - `signature_examples`에 새 사례 append (최대 5건까지, 그 이상은 누락)
   - `last_seen_at` 갱신
2. **본진 중복 검사**: `ai-tell-taxonomy.md`의 기존 패턴(`A-1`~`J-N`)과 의미가 겹치면 후보로 추가하지 말고, run 산출물에 "기존 ID `X-N`로 분류 가능" 기록만 남긴다.
3. **신규 항목 ID 발급**: `cand-{대분류 힌트}-{YYYY}-{NNN}`. NNN은 해당 연도 안에서 풀 전체 누적 카운트(대분류와 무관). 대분류 힌트가 불확실하면 `cand-X-YYYY-NNN`로 두고 taxonomist가 분류.
4. **원문 보존 필수**: `signature_examples[].text`는 원문 그대로. 윤문된 형태나 일반화된 형태로 적지 않는다.
5. **기록 후 알림**: 적재 직후 본 run의 산출물(예: `05_naturalness_review.json`의 `unclassified_candidates_appended`)에 신규/갱신된 후보 ID 목록을 남겨 taxonomist 점검 trigger.

## 승격 기준 (taxonomist 운영)

후보가 다음 조건을 모두 충족하면 본진으로 승격 가능 (taxonomist 최종 판정):

1. **재현**: `occurrences ≥ 2` (서로 다른 source_run_id 기준 2건 이상)
2. **장르 분산**: 같은 작가·같은 run에서만 발견된 패턴은 보류 — 최소 2개 이상의 서로 다른 입력 맥락 필요
3. **수술 가능성**: `suggested_fix_draft`가 의미 불변·장르 유지·과윤문 금지 4대 철칙과 충돌하지 않음
4. **본진 중복 아님**: 기존 패턴의 변종이면 본진 항목에 보강(예: `A-2`의 시그니처 예문 추가)으로 처리하고 후보는 `merged`로 닫음

승격 시: 본 후보의 `status`를 `promoted`로 갱신, `status_reason`에 머지된 본진 ID(예: `promoted to A-16`) 기재. 기각 시: `status: rejected` + 사유. 본진 보강으로 흡수: `status: merged` + 본진 ID.

**기각 사유 표준 라벨:**
- `not_ai_specific` — 인간 필자도 흔히 쓰는 표현
- `single_run_only` — 한 run에서만 발견, 장르 분산 미달
- `genre_dependent` — 특정 장르(에세이·SNS)에서만 자연스러운 변별, 일반화 불가
- `subjective_aesthetic` — "어색하다"는 주관 평가에 가깝고 객관 시그니처 부족
- `ambiguous_overlap` — 본진 패턴과 경계가 흐려 어느 쪽으로도 분류 불안정

## 라이프사이클 정책

- `pending` 상태로 90일 이상 머무르면서 `occurrences == 1`인 후보는 다음 taxonomist 점검 회차에서 자동 후보 만료(`status: rejected`, `status_reason: "single_run_only — 90일 미재현"`).
- `promoted`·`rejected`·`merged` 상태는 본 풀에 계속 보존(삭제 금지). taxonomy 변경 이력의 한 축이며, 같은 후보가 재제안되는 것을 방지한다.
- 본 풀의 항목 수가 200을 넘으면 `references/archive/pattern-candidates-{YYYY}.md`로 closed(promoted/rejected/merged) 항목을 분리해 본 파일은 pending 중심으로 유지.

## 외부 contributor 적재 (Issue/PR 경로)

GitHub Issue 또는 PR로 외부에서 후보를 제출할 때도 본 스키마를 그대로 사용한다.

- `discovered_by: "external"`
- `signature_examples[].source_run_id`: 외부 사례면 Issue/PR 번호 (예: `gh-issue-12`)
- 외부 contributor는 `id`를 직접 발급하지 말고 `cand-X-YYYY-pending`로 두면 머지 시 maintainer가 정식 ID 발급
- 외부 후보도 동일 승격 기준(재현 2회+·장르 분산·본진 중복 아님)을 적용

`§6 외부 회귀 검증 케이스 모집`(Issue #4)과 별개 트랙 — Issue #4는 voice profile 회귀용, 본 풀은 신규 패턴 발굴용이다.

## 풀 운영 주기 (taxonomist)

기본은 사용자 trigger 기반(taxonomist는 호출되어야 작동). 다음 4가지 trigger 중 하나에 해당하면 풀 점검:

1. **사용자 명시 요청**: "패턴 풀 점검", "후보 승격 검토", "v1.x 분류 확장"
2. **풀 누적 임계**: `pending` 항목 10건 이상
3. **고빈도 후보 등장**: 단일 후보의 `occurrences ≥ 3`
4. **외부 PR/Issue 도착**: 외부 contributor 후보가 새로 등재됨

점검 산출물은 `_workspace/taxonomy_changelog.md`에 회차별로 누적(어떤 후보가 어떤 사유로 승격·기각되었는지 추적 가능).

---

## 풀

<!-- pending: 아직 점검 전 -->
<!-- 신규 후보는 이 섹션에 append. 적재 절차의 중복 검사·필수 필드를 지킬 것. -->

(현재 0건 — 회차 1에서 모두 closed 상태로 이동)

---

<!-- promoted: 본진 승격 완료 -->

```yaml
- id: "cand-C-2026-001"
  pattern_label: "숫자 괄호 인덱싱 1) 2) 3)"
  proposed_category: "C"
  proposed_severity: "S2"
  description: |
    동일 문단 또는 인접 문장에서 항목을 `1) ... 2) ... 3) ...` 형식으로 나열.
    C-1(첫째·둘째·셋째)·C-2(불릿)와 별개의 표기 시그니처. 한국어 인간 필자도
    보고서에서 가끔 쓰지만 LLM 산출물에서는 3개 항목이 있을 때 거의 자동으로
    숫자 괄호 인덱싱이 등장하는 빈도가 압도적으로 높음.
  signature_examples:
    - text: "1) 표준화된 OT(Operational Technology) 데이터 수집 인프라가 일반화되면서 학습 데이터 확보가 용이해졌다. 2) 도메인 특화 LLM이 성숙하면서 한국어 운영 매뉴얼·작업 일지 같은 비정형 텍스트의 활용 범위가 넓어졌다. 3) 클라우드 GPU 단가가 2년 전 대비 크게 하락하면서 단일 공장 단위의 자체 학습이 비용 측면에서 정당화 가능한 수준에 들어왔다."
      context: "샘플 A 둘째 문단 — 제조업 디지털 전환 칼럼톤"
      source_run_id: "synthetic-claude-pilot-2026-04-25-001a"
      discovered_by: "ai-tell-detector"
      discovered_at: "2026-04-25"
    - text: "1) 카카오뱅크·케이뱅크·토스뱅크의 합산 순이익은 전년 동기 대비 18% 증가했지만, 사용자 1인당 평균 매출은 오히려 2.4% 감소했다. 2) 마이데이터 사업자 중 흑자 전환에 성공한 곳은 전체의 12%에 불과하며, 절반 이상은 데이터 활용 사례 발굴 단계에 머물러 있다. 3) 보험 비교·추천 플랫폼은 전년 대비 거래액이 두 배 가까이 늘었으나, 수수료 단가 인하 압력이 동시에 강화되어 영업이익률은 정체되고 있다."
      context: "샘플 B 둘째 문단 — 핀테크 리포트톤"
      source_run_id: "synthetic-claude-pilot-2026-04-25-001b"
      discovered_by: "ai-tell-detector"
      discovered_at: "2026-04-25"
  suggested_fix_draft: |
    3개 중 1개는 서술문으로 녹이고, 나머지 2개도 1)·2) 표기 대신 "우선~",
    "다음으로~" 형식으로 어휘 변주. 정말 동일 구조 나열이 의미 있을 때만
    숫자 괄호를 유지하되 한 문서에 1회 이하.
  occurrences: 2
  status: "promoted"
  status_reason: "promoted to C-9 (회차 1, 2026-04-25)"
  created_at: "2026-04-25"
  last_seen_at: "2026-04-25"
  reviewed_by_taxonomist: true
```

---

<!-- rejected: 기각 -->

(현재 0건)

---

<!-- hold: 데이터 부족으로 다음 회차 이월 -->

```yaml
- id: "cand-A-2026-002"
  pattern_label: "메타 진입 '~을 살펴보면 / 들여다보면'"
  proposed_category: "A"
  proposed_severity: "S2"
  description: |
    글의 첫 문장 또는 단락 진입에서 본격 서술 전 "X를 살펴보면 / 들여다보면"
    형태로 메타 단계를 한 번 거치는 LLM 특유 도입 패턴. 영어 `looking at / when
    we examine` 직역. 본진 A-1(에 대해) · A-3(에 있어) 인접하지만 형태소가 다름.
  signature_examples:
    - text: "최근 한국 제조업의 디지털 전환 흐름을 살펴보면 흥미로운 변화가 감지된다."
      context: "샘플 A 첫 문장 — 제조업 칼럼톤 도입부"
      source_run_id: "synthetic-claude-pilot-2026-04-25-001a"
      discovered_by: "ai-tell-detector"
      discovered_at: "2026-04-25"
    - text: "2026년 상반기 국내 핀테크 시장의 경쟁 구도를 들여다보면 몇 가지 중요한 신호가 잡힌다."
      context: "샘플 B 첫 문장 — 핀테크 리포트톤 도입부"
      source_run_id: "synthetic-claude-pilot-2026-04-25-001b"
      discovered_by: "ai-tell-detector"
      discovered_at: "2026-04-25"
  suggested_fix_draft: |
    메타 진입 자체를 삭제하고 본 서술로 직진. "최근 한국 제조업의 디지털 전환 흐름에서
    흥미로운 변화가 감지된다." 형태로 첫 문장에서 바로 결론 진입.
  occurrences: 2
  status: "hold"
  status_reason: "Gate 1.3 fail — 같은 합성 회차 출처라 작가/도메인 분산 미충족. 다음 회차에 다른 작가·다른 장르 샘플에서 재현되면 재판정"
  created_at: "2026-04-25"
  last_seen_at: "2026-04-25"
  reviewed_by_taxonomist: true

- id: "cand-H-2026-004"
  pattern_label: "'결국' 문두 단언 남발"
  proposed_category: "H"
  proposed_severity: "S2"
  description: |
    GPT가 단언으로 결산할 때 거의 자동으로 문두에 "결국"이 붙음. 본진 H-1의 명시
    어휘("또한·따라서·즉·나아가·아울러·게다가·더욱이")에 "결국"이 빠져 있는데,
    한국 매체에 게재된 GPT 출력에서는 H-1 명시 어휘보다 "결국·다시 말해·특히"가
    압도적으로 많음. 한국어 인간 필자도 사용하지만 한 문서에 9회 이상은 결정적
    AI 시그니처. 본진 H-1 어휘 셋 자체가 한국 매체 GPT 출력의 실제 분포와 어긋나
    있다는 발견의 일환.
  signature_examples:
    - text: "결국 'AI의 핵심은 일자리 제거보다 일의 방식과 시장의 구조를 바꾸는 것'에 있다는 뜻이다"
      context: "뉴스핌 ① 본문 핵심 해석"
      source_run_id: "external-newspim-gpt-2026-04-23-001"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "결국 산업 구조는 두 층으로 갈린다"
      context: "뉴스핌 ① 본문 응용 산업"
      source_run_id: "external-newspim-gpt-2026-04-23-001"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "결국 한국의 비교우위는 ~ 기존 강한 산업에 AI를 깊숙이 붙여 현장을 바꾸는 나라에 더 가깝다"
      context: "뉴스핌 ② 본문 한국 승부처"
      source_run_id: "external-newspim-gpt-2026-04-24-002"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "결국 살아남는 사람은 '내가 하던 일을 그대로 지키는 사람'이 아니라, 내 일을 AI 시대 방식으로 다시 조립할 줄 아는 사람이다"
      context: "뉴스핌 ② 본문 인재론 결산"
      source_run_id: "external-newspim-gpt-2026-04-24-002"
      discovered_by: "external"
      discovered_at: "2026-04-25"
  suggested_fix_draft: |
    한 문서에 "결국" 2회 이하로 제한. 결산 문장은 "결국" 없이 단언으로 직결
    (예: "결국 산업 구조는 두 층으로 갈린다" → "산업 구조는 두 층으로 갈린다").
    필요하다면 "정리하면·이는 곧·따져 보면" 등으로 어휘 변주, 단 한 문서에
    이런 결산 어휘 자체를 3회 이상 누적하지 않음.
  occurrences: 9
  status: "hold"
  status_reason: "Gate 1.3 fail — 같은 모델(GPT)·같은 기자(정성훈) 시리즈 두 편이라 source distinct는 2건이지만 작가/모델 분산 weak. 회차 3에서 Gemini·국내 모델·다른 매체 GPT 출력에서 재현되면 즉시 promoted 가능 (강력 후보)"
  created_at: "2026-04-25"
  last_seen_at: "2026-04-25"
  reviewed_by_taxonomist: true

- id: "cand-D-2026-005"
  pattern_label: "'X은 A가 아니라 B다' 부정-긍정 대구 결산"
  proposed_category: "D"
  proposed_severity: "S2"
  description: |
    GPT가 명제·결산 문장을 만들 때 거의 자동으로 "A가 아니라 B" 또는 "A보다 B"
    형태의 부정-긍정 대구로 결산. 한 줄 요약·소제목·문단 결말에서 빈출. C-8(A인가
    B인가, 질문형)·D-1(종결류)와 시그니처 유형이 다른 결산 공식. 한국어 인간
    필자도 사용하지만 한 문서에 7회 이상은 GPT 특유.
  signature_examples:
    - text: "AI 시대의 승자는 기술 보유자가 아니라 산업 재설계자다"
      context: "뉴스핌 ① 한 줄 요약"
      source_run_id: "external-newspim-gpt-2026-04-23-001"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "위험한 사람은 직급이 낮은 사람이 아니라, 업무가 쉽게 쪼개지고 문서화되며 규칙화되는 사람이다"
      context: "뉴스핌 ② 인재론 도입"
      source_run_id: "external-newspim-gpt-2026-04-24-002"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "AI는 사람을 완전히 밀어내는 기술이라기보다, 사람 1명이 만들어낼 수 있는 가치의 범위를 크게 넓히는 기술에 가깝다"
      context: "뉴스핌 ① 본문 결산"
      source_run_id: "external-newspim-gpt-2026-04-23-001"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "감원보다 확장, 자동화보다 재설계, 기술보다 확산"
      context: "뉴스핌 ① 결말부 — 3중 대구"
      source_run_id: "external-newspim-gpt-2026-04-23-001"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "결국 살아남는 사람은 '내가 하던 일을 그대로 지키는 사람'이 아니라, 내 일을 AI 시대 방식으로 다시 조립할 줄 아는 사람이다"
      context: "뉴스핌 ② 인재론 결산"
      source_run_id: "external-newspim-gpt-2026-04-24-002"
      discovered_by: "external"
      discovered_at: "2026-04-25"
  suggested_fix_draft: |
    한 문서에 "A가 아니라 B" 형태의 결산 대구 2회 이하. 결산이 필요한 곳은 단언
    하나로 (예: "AI 시대의 승자는 기술 보유자가 아니라 산업 재설계자다" → "AI
    시대의 승부는 산업 재설계에서 갈린다"). 부정-긍정 대구 자체보다 무엇을
    긍정하는지가 더 또렷한 표현으로.
  occurrences: 7
  status: "hold"
  status_reason: "Gate 1.3 fail — 같은 모델·같은 기자 시리즈. 회차 3에서 다른 모델·다른 작가에서 재현되면 즉시 promoted 가능 (강력 후보, D-7 신규 등재 예정)"
  created_at: "2026-04-25"
  last_seen_at: "2026-04-25"
  reviewed_by_taxonomist: true

- id: "cand-C-2026-006"
  pattern_label: "5~8개 영역 콤마 빠른 나열"
  proposed_category: "C"
  proposed_severity: "S2"
  description: |
    5개 이상 영역을 콤마로 빠르게 나열하는 패턴이 한 기사에 3~4회 등장. 한국어
    인간 필자는 보통 3~4개 이상 나열하면 불릿이나 번호를 붙이는데, GPT는 콤마
    나열을 선호. C-1(첫째·둘째·셋째)·C-2(불릿)와 다른 시그니처.
  signature_examples:
    - text: "검색, 번역, 요약, 설계 초안, 데이터 분석, 개발 보조, 고객 응대의 단가를 낮추면서"
      context: "뉴스핌 ① 7개 영역 콤마 나열"
      source_run_id: "external-newspim-gpt-2026-04-23-001"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "광고, 영상, 게임, 출판, 뉴스, 마케팅에서 소량 다품종 제작이 쉬워지며"
      context: "뉴스핌 ① 6개 영역"
      source_run_id: "external-newspim-gpt-2026-04-23-001"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "공정 데이터, 설비 운영, 품질 검사, 예지보전, 수요예측, 공급망 관리 등 AI가 침투할 수 있는 지점이 많다"
      context: "뉴스핌 ② 6개 영역"
      source_run_id: "external-newspim-gpt-2026-04-24-002"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "제조, 금융, 유통, 물류, 의료, 교육, 건설, 공공행정에 붙는 기업간거래(B2B) 소프트웨어 시장이다"
      context: "뉴스핌 ① 8개 영역"
      source_run_id: "external-newspim-gpt-2026-04-23-001"
      discovered_by: "external"
      discovered_at: "2026-04-25"
  suggested_fix_draft: |
    5개 이상 나열은 (a) 핵심 2~3개로 압축 + "등"으로 마무리, (b) 정말 모두
    중요하면 불릿으로 분리, (c) 그룹화해 "제조·물류 같은 산업 현장 영역과
    의료·교육 같은 서비스 영역" 형태의 의미 묶음으로 재서술. 콤마로 5개 이상
    한 줄에 늘어놓지 않음.
  occurrences: 4
  status: "hold"
  status_reason: "Gate 1.3 fail — 같은 모델·같은 기자. occurrences는 충분(4건)이나 분산 weak. 회차 3에서 검증 필요"
  created_at: "2026-04-25"
  last_seen_at: "2026-04-25"
  reviewed_by_taxonomist: true
```

---

<!-- merged: 본진 패턴에 흡수 -->

```yaml
- id: "cand-I-2026-003"
  pattern_label: "'X은 ~라는 점에 있다' 강조 위치 서술"
  proposed_category: "I"
  proposed_severity: "S2"
  description: |
    "핵심은 ~라는 점에 있다 / 의의는 ~라는 점에 있다 / 주목할 부분은 ~라는
    점에 있다" 형식의 강조 위치 서술. 본진 I-2(점·바·수·데), I-3(~라는 것)과
    의미상 강하게 겹침. 단일 ID 분리 가능성 검토 결과, I-2의 결합형 변종으로
    판정.
  signature_examples:
    - text: "핵심은 AI 도입의 진입장벽이 빠르게 낮아지고 있다는 점에 있다"
      context: "샘플 A 첫 문단 결말"
      source_run_id: "synthetic-claude-pilot-2026-04-25-001a"
      discovered_by: "ai-tell-detector"
      discovered_at: "2026-04-25"
    - text: "의의는 OT 데이터의 표준화가 여전히 사업장별로 들쭉날쭉하다는 점에 있다"
      context: "샘플 A 마지막 문단 중간"
      source_run_id: "synthetic-claude-pilot-2026-04-25-001a"
      discovered_by: "ai-tell-detector"
      discovered_at: "2026-04-25"
    - text: "특히 주목할 부분은 마이데이터 사업의 수익 모델이 여전히 정착되지 않았다는 점에 있다"
      context: "샘플 B 첫 문단 결말"
      source_run_id: "synthetic-claude-pilot-2026-04-25-001b"
      discovered_by: "ai-tell-detector"
      discovered_at: "2026-04-25"
    - text: "결국 핵심은 이용자 데이터 활용의 명확한 기준선을 어디에 그을지에 있다는 점에 있다"
      context: "샘플 B 마지막 문단 중간"
      source_run_id: "synthetic-claude-pilot-2026-04-25-001b"
      discovered_by: "ai-tell-detector"
      discovered_at: "2026-04-25"
  suggested_fix_draft: |
    "핵심은 ~다" 형식으로 "점에 있다"를 직결 단언으로 치환. 또는 술어 자체를
    구체 동사로(예: "AI 도입의 진입장벽이 빠르게 낮아진다"). I-2 처방과 동일 라인.
  occurrences: 4
  status: "merged"
  status_reason: "merged to I-2 (회차 1, 2026-04-25) — I-2 본진 항목의 시그니처 예문에 'X은 ~라는 점에 있다' 결합형 추가"
  created_at: "2026-04-25"
  last_seen_at: "2026-04-25"
  reviewed_by_taxonomist: true

- id: "cand-I-2026-007"
  pattern_label: "'~라는 뜻이다 / ~다는 뜻이다' 결말 단언 공식"
  proposed_category: "I"
  proposed_severity: "S2"
  description: |
    GPT가 서술을 형식명사로 결산할 때 거의 자동으로 "~라는 뜻이다" 또는
    "~다는 뜻이다"로 마무리. 본진 I-3 "~라는 것이다"의 명확한 변종이며
    의미·기능 동일(서술 결산을 형식명사로 마무리). Gate 2.2 본진 변종 판정으로
    별도 ID 분리 대신 본진 I-3 보강.
  signature_examples:
    - text: "결국 'AI의 핵심은 일자리 제거보다 일의 방식과 시장의 구조를 바꾸는 것'에 있다는 뜻이다"
      context: "뉴스핌 ① 본문"
      source_run_id: "external-newspim-gpt-2026-04-23-001"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "기술 도입보다 인력 전환이 더 큰 병목이 될 수 있다는 뜻이다"
      context: "뉴스핌 ② 본문"
      source_run_id: "external-newspim-gpt-2026-04-24-002"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "한국에서는 이 문제가 더 민감하다는 뜻이다"
      context: "뉴스핌 ② 본문"
      source_run_id: "external-newspim-gpt-2026-04-24-002"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "더 큰 시장은 그 위에 올라가는 응용 산업에서 나올 가능성이 높다는 뜻이다"
      context: "뉴스핌 ① 본문"
      source_run_id: "external-newspim-gpt-2026-04-23-001"
      discovered_by: "external"
      discovered_at: "2026-04-25"
  suggested_fix_draft: |
    "~라는 뜻이다 / ~다는 뜻이다"를 "~다" 직접 종결로 (예: "기술 도입보다
    인력 전환이 더 큰 병목이 될 수 있다는 뜻이다" → "병목은 기술 도입이
    아니라 인력 전환이다"). 한 문서에 형식명사 결산("~다는 것이다 / ~다는
    뜻이다 / ~다는 점이다") 합산 2회 이하.
  occurrences: 4
  status: "merged"
  status_reason: "merged to I-3 (회차 2, 2026-04-25) — I-3 본진 항목의 시그니처 예문에 '~라는 뜻이다 / ~다는 뜻이다' 결말 변종 추가"
  created_at: "2026-04-25"
  last_seen_at: "2026-04-25"
  reviewed_by_taxonomist: true

- id: "cand-H-2026-008"
  pattern_label: "'이 점에서 / 이 관점에서 / 이 말은' 메타 진입"
  proposed_category: "H"
  proposed_severity: "S2"
  description: |
    본진 H-3 "이는 ~"의 변종. 앞 문장을 받아 부연 설명할 때 "이 점에서 ~",
    "이 관점에서 보면 ~", "이 말은 ~" 형태가 빈번. Gate 2.2 본진 변종 판정으로
    별도 ID 분리 대신 본진 H-3 보강.
  signature_examples:
    - text: "이 관점에서 보면 AI 시대 유망 직무도 다시 보인다"
      context: "뉴스핌 ① 본문"
      source_run_id: "external-newspim-gpt-2026-04-23-001"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "이 말은 결국 기술 도입보다 인력 전환이 더 큰 병목이 될 수 있다는 뜻이다"
      context: "뉴스핌 ② 본문"
      source_run_id: "external-newspim-gpt-2026-04-24-002"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "이 점에서 앞으로 강해질 인재는 크게 다섯 부류다"
      context: "뉴스핌 ② 본문"
      source_run_id: "external-newspim-gpt-2026-04-24-002"
      discovered_by: "external"
      discovered_at: "2026-04-25"
    - text: "이 점에서 한국은 승산이 있다"
      context: "뉴스핌 ② 본문"
      source_run_id: "external-newspim-gpt-2026-04-24-002"
      discovered_by: "external"
      discovered_at: "2026-04-25"
  suggested_fix_draft: |
    "이 점에서 / 이 관점에서 / 이 말은"을 앞 문장과 직접 붙이거나 구체 서술로
    치환. 예: "이 관점에서 보면 AI 시대 유망 직무도 다시 보인다" → "AI 시대
    유망 직무도 같은 흐름에서 보인다". H-3 처방 라인.
  occurrences: 4
  status: "merged"
  status_reason: "merged to H-3 (회차 2, 2026-04-25) — H-3 본진 항목의 시그니처 예문에 '이 점에서 / 이 관점에서 / 이 말은' 변종 4건 추가"
  created_at: "2026-04-25"
  last_seen_at: "2026-04-25"
  reviewed_by_taxonomist: true
```

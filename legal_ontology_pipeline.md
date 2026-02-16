# 법리·쟁점 관계표현 중심 온톨로지-LLM 연구 파이프라인 (MVP)

## 0) 프로젝트 목표(한 문장)
- **법리를 새로 생성하지 않고**, 판결문/법령에 이미 존재하는 법리·쟁점·근거 관계를 구조화하여 질의응답 및 분석에 활용한다.

---

## 1) 전체 아키텍처 (권장)
1. **문서 수집 계층**
   - 입력: 판결문, 법제처 조문, 시행령/시행규칙 메타데이터
   - 출력: 문서 ID, 문단 ID, 시행일/개정일, 사건번호 등 정규화된 원천 데이터
2. **정보추출 계층 (LLM + 규칙 기반 혼합)**
   - 기능: 쟁점(Issue), 사실(Fact), 요건(Element), 판단(Holding), 근거(EvidenceSpan) 후보 추출
   - 출력: 후보 노드/관계 + confidence + 출처 span
3. **온톨로지/지식그래프 계층**
   - 기능: RDF/Property Graph 저장, 스키마 제약 검증(SHACL/커스텀 룰)
   - 출력: 검증 통과 그래프 + 위반 리포트
4. **자기점검 계층 (Self-check)**
   - 기능: 근거 일치성, 시간 유효성(시행시점), 관계 모순성 검사
   - 출력: pass/fail + 이유 코드 + 재추출 요청
5. **RAG/질의응답 계층**
   - 기능: 사용자 질의를 그래프 질의 + 벡터검색으로 분해, 근거 포함 답변 생성
   - 출력: 답변 + 근거 citation + 확신도 + 유효 시점

---

## 2) 첫 8주 연구 로드맵 (비법률전문가 팀 기준)

### Phase 1 (1~2주): 스코프 고정
- 도메인 축소: 민사/형사 중 하나만 선택
- 핵심 클래스 6개만 확정
  - `LegalSource`, `Issue`, `Element`, `Fact`, `Holding`, `EvidenceSpan`
- 핵심 관계 8개만 확정
  - `raises_issue`, `requires_element`, `applies_to_fact`, `supported_by`, `refers_to_source`, `conflicts_with`, `distinguishes`, `overrules`
- 산출물
  - annotation guideline v0.1
  - 샘플 판결문 30건

### Phase 2 (3~4주): 추출 파이프라인 MVP
- LLM 프롬프트 기반 후보 추출기 구현
- 규칙 기반 후처리(이벤트/날짜/사건번호 정규화)
- 노드/엣지에 출처 span 강제 저장
- 산출물
  - 자동 추출 JSONL
  - 수작업 정답셋(20~30건)

### Phase 3 (5~6주): 그래프 적재 + 검증
- 그래프 DB 적재(Neo4j 또는 RDF store)
- 제약 검증 룰 구현
  - 근거 없는 판단 금지
  - 시행일 이후 조문만 허용(질문 시점 기반)
  - 동일 사건 내 순환 모순 탐지
- 산출물
  - validation report
  - 오류 카테고리 대시보드

### Phase 4 (7~8주): QA 통합 + 평가
- 하이브리드 질의
  - 1차: 그래프 경로 탐색
  - 2차: 벡터 검색으로 추가 근거 확보
- 답변 템플릿 고정
  - 결론 / 근거 / 한계 / 시점
- 오프라인 평가
  - Evidence Precision
  - Relation F1
  - Temporal Validity Accuracy
- 산출물
  - 데모 API
  - 최종 리포트 v1

---

## 3) 모델/학습 전략 (과도한 파인튜닝 방지)
1. **초기 전략**: 파인튜닝보다 프롬프트+스키마 강제 우선
2. **중기 전략**: 오류가 많은 태스크만 LoRA/미세튜닝
   - 예: `Issue↔Holding` 매핑 오류가 크면 해당 태스크만 데이터셋 구축
3. **튜닝 원칙**
   - 생성형 답변 튜닝보다 **관계추출 튜닝**에 집중
   - 모든 출력은 JSON Schema 검증 통과를 필수화

---

## 4) Self-check 설계 (최소 필수 3종)
1. **근거 정합성 검사**
   - 주장 문장마다 `EvidenceSpan` 존재 여부
   - span 유사도/포함률 임계치 미달 시 fail
2. **시간 정합성 검사**
   - 질문 시점 대비 조문 시행일/개정일 유효 여부
3. **관계 정합성 검사**
   - `conflicts_with`가 있는데 동일 답변에서 동시 채택되는지 확인

실패 시 정책:
- 자동 재시도 1회 (retrieval 강화)
- 재실패 시 “확정 답변 보류 + 관련 근거 후보만 제시”

---

## 5) 팀 운영 방식 (법률전문가 부족 대응)
- 상시 자문 대신 **스팟 리뷰**
  - 주 1회, 60~90분, 실패 사례 상위 20개만 검토
- 내부 역할 분리
  - ML: 추출/검증
  - Data: 문서 정규화/품질
  - PM/리서치: 평가셋/오류 분류
- 품질 기준
  - “정답률”보다 “근거추적 가능성”을 1순위 KPI로 둔다

---

## 6) 바로 시작할 To-do (이번 주)
1. 대상 코퍼스 30건 선정
2. annotation guideline 2페이지 작성
3. 클래스 6개/관계 8개 스키마 확정
4. 추출 결과 JSON Schema 작성
5. 베이스라인 추출기(프롬프트 기반) 1차 실행

---

## 7) 실패를 줄이는 실무 팁
- 처음부터 판례 간 선후관계(전원합의체/파기환송 등)까지 확장하지 말 것
- 사건 내부 구조(쟁점-판단-근거)가 안정화된 뒤 사건 간 관계로 확장
- “근거 없는 고신뢰 답변”은 무조건 실패 처리


---

## 8) LoRA 파인튜닝 vs NeOn Few-shot vs 하이브리드 (의사결정 가이드)

질문: "LoRA로 튜닝할지, 고성능 모델+NeOn few-shot으로 갈지, 둘 다 할지"

### 결론(권장)
- **초기(0~2개월): 고성능 모델 + NeOn few-shot 우선**
- **중기(오류 패턴 고정 후): LoRA를 '관계추출 태스크'에만 제한 적용**
- 즉, 대부분 팀에선 **순차적 하이브리드**가 가장 안전하다.

### A. 고성능 모델 + NeOn few-shot이 유리한 경우
- 데이터셋이 아직 작고(라벨 1,000건 미만), 스키마도 자주 바뀌는 초기 단계
- 팀이 빠른 실험/반복이 필요할 때
- 장점
  - 개발 속도가 빠름
  - 프롬프트/예시 교체로 즉시 성능 조정 가능
  - 스키마 변경 대응이 쉬움
- 단점
  - 호출비용이 큼
  - 출력 편차(variance)가 남을 수 있음

### B. LoRA 파인튜닝이 유리한 경우
- 특정 오류가 반복적으로 고정되어 나타날 때
  - 예: `Issue↔Holding` 혼동, `supported_by` 누락
- 추론비용/지연시간 최적화가 중요한 운영 단계
- 장점
  - 동일 태스크에서 안정성/일관성 향상 가능
  - 장기적으로 추론비 절감 가능
- 단점
  - 데이터셋 구축/검수 비용이 큼
  - 스키마 변경 시 재튜닝 부담

### C. 추천 실행 순서 (실무)
1. NeOn few-shot으로 베이스라인 구축
2. 오류 분석으로 상위 2개 실패 유형 선정
3. 해당 유형만 소형 LoRA 데이터셋 구축(정밀 라벨링)
4. LoRA 모델을 추출 단계에만 적용하고, 최종 QA 생성은 고성능 모델 유지
5. 월 단위로 재학습 여부 판단

### D. 선택 기준 (정량)
- 아래 중 2개 이상 충족 시 LoRA 착수 권장
  - Relation F1이 2주 연속 목표치 미달
  - 동일 relation 누락/오분류가 전체 오류의 35% 이상
  - 호출비용이 예산 한도를 20% 이상 초과
  - 평균 응답 지연이 SLA를 반복 초과

### E. 현재 팀 상황에 대한 추천
- 비법률전문가 + 초기 연구 단계라면, **지금 당장은 NeOn few-shot이 우선**이다.
- 다만 "둘 중 하나"가 아니라, **few-shot으로 스키마/평가를 안정화한 뒤 필요한 부분만 LoRA**로 가져가는 전략이 리스크가 가장 낮다.


---

## 9) "도메인 한정"은 어디까지인가? (범용성과의 균형)

핵심은 **지식의 표현 틀은 범용적으로**, **라벨링/평가 단위는 제한적으로** 시작하는 것이다.

### A. 한정의 대상과 비한정의 대상
- 한정하는 것
  - 초기 코퍼스 범위(예: 민사 1개 하위유형)
  - 초기 관계 집합(핵심 8~12개)
  - 초기 평가셋(고빈도 질문 유형 중심)
- 한정하지 않는 것
  - 메타 스키마 구조(`Issue`, `Fact`, `Holding`, `EvidenceSpan` 등)
  - provenance/시간유효성/self-check 같은 품질 레이어
  - 확장 가능한 relation namespace 설계

즉, **도메인 자체를 영구히 좁게 고정**하는 게 아니라, **실험/검증 단위를 좁게** 가져가자는 의미다.

### B. 실무 기준: "국한"의 권장 폭
- 코퍼스: 전체 법 분야가 아니라 1개 하위영역(예: 계약해제, 손해배상, 횡령 구성요건 중 1개)
- 문서량: 초기 30~100건
- 클래스: 6~10개
- 관계: 8~12개
- 기간: 4~8주 내 안정화

이 범위에서 목표는 "전체 법률 범용성"이 아니라, **관계표현 파이프라인의 재현성**을 먼저 확보하는 것.

### C. 범용성은 어떻게 확보하나? (단계적 확장)
1. Core Ontology 고정
   - 도메인 공통 클래스/관계는 변경 최소화
2. Domain Pack 분리
   - 분야별 relation/용어는 별도 확장 모듈로 관리
3. 동일 평가축 유지
   - 분야가 바뀌어도 Evidence Precision, Relation F1, Temporal Validity를 동일하게 측정
4. 확장 순서
   - 사건 내부 관계 안정화 → 사건 간 관계 → 타 법영역 전이

### D. 의사결정 문장(팀 합의용)
- "우리는 법률 도메인을 좁게 정의하는 것이 아니라, **검증 가능한 단위로 쪼개서 순차 확장**한다."
- "초기 성과지표는 범용 지식량이 아니라, **근거 추적 가능성과 관계 정합성**이다."

---

## 10) 노동법 도메인 온톨로지 초안 (v0.1)

아래 설계는 "법리 생성"이 아니라, 노동법 문서(법령/판례/행정해석)의 **쟁점-요건-사실-판단-근거 관계**를 표현하기 위한 최소 코어다.

### A. 핵심 클래스 (Classes)
1. `LaborLegalSource`
   - 노동관계 법령/시행령/행정규칙/판례 원문 소스
2. `LaborCase`
   - 개별 사건(대법원/고법/지법 등)
3. `Party`
   - 근로자, 사용자, 노조, 사업장
4. `Issue`
   - 쟁점(해고의 정당성, 통상임금 해당성, 근로자성 인정 등)
5. `LegalElement`
   - 요건(긴박한 경영상 필요, 해고회피 노력, 합리적 이유 등)
6. `Fact`
   - 사실관계(근로계약 형태, 근무시간, 임금구성항목, 징계사유)
7. `Holding`
   - 판단 결론(인정/부정, 파기/기각 등)
8. `Remedy`
   - 구제 유형(원직복직, 임금상당액 지급, 부당해고 구제 등)
9. `EvidenceSpan`
   - 원문 근거 스팬(문서ID, 문단ID, 시작/끝 오프셋)
10. `TimeContext`
   - 시행일/개정일/사건 발생일/판결선고일

### B. 핵심 관계 (Object Properties)
- `raises_issue (LaborCase -> Issue)`
- `involves_party (LaborCase -> Party)`
- `requires_element (Issue -> LegalElement)`
- `applies_to_fact (LegalElement -> Fact)`
- `decides_issue (Holding -> Issue)`
- `supported_by (Holding -> EvidenceSpan)`
- `refers_to_source (EvidenceSpan -> LaborLegalSource)`
- `grants_remedy (Holding -> Remedy)`
- `has_time_context (LaborCase|LaborLegalSource -> TimeContext)`
- `conflicts_with (Holding -> Holding)`
- `distinguishes (LaborCase -> LaborCase)`
- `cites_precedent (LaborCase -> LaborCase)`

### C. 데이터 속성 (Data Properties)
- `caseNumber` (LaborCase)
- `courtLevel` (LaborCase)
- `decisionDate` (LaborCase)
- `articleNumber` (LaborLegalSource)
- `effectiveDate`, `amendedDate` (LaborLegalSource/TimeContext)
- `partyRole` (Party: worker/employer/union)
- `confidence` (Holding/Relation)
- `textSpan`, `paragraphIndex` (EvidenceSpan)

### D. 노동법 특화 쟁점 템플릿 (Starter Taxonomy)
- 근로자성 인정 여부
- 해고의 정당성(징계해고/통상해고/정리해고)
- 통상임금 해당성
- 연장·야간·휴일근로수당 산정
- 기간제/파견의 불법파견 여부
- 취업규칙 불이익 변경의 유효성
- 부당노동행위 성립 여부
- 산업재해/업무상 재해 인과관계

### E. 예시 트리플 (Turtle 스타일, 축약)
```ttl
:Case_2024Da12345 a :LaborCase ;
  :caseNumber "2024다12345" ;
  :raises_issue :Issue_UnfairDismissal ;
  :involves_party :Worker_A , :Employer_B ;
  :has_time_context :Time_2024Case .

:Issue_UnfairDismissal a :Issue ;
  :requires_element :Element_JustCause , :Element_DueProcess .

:Element_JustCause a :LegalElement ;
  :applies_to_fact :Fact_DisciplinaryRecord .

:Holding_DismissalInvalid a :Holding ;
  :decides_issue :Issue_UnfairDismissal ;
  :supported_by :Ev_001 ;
  :grants_remedy :Remedy_Reinstatement .

:Ev_001 a :EvidenceSpan ;
  :refers_to_source :LSA_Art23 ;
  :paragraphIndex "17" ;
  :textSpan "해고는 정당한 이유가 있어야 한다" .
```

### F. 최소 검증 규칙 (운영 필수)
1. `Holding`은 최소 1개의 `supported_by`를 가져야 함.
2. `Issue`는 최소 1개의 `requires_element`를 가져야 함.
3. 질의 시점 이후 시행된 조문은 답변 근거로 사용 금지.
4. 같은 답변에서 `conflicts_with` 관계의 결론을 동시 채택 금지.

### G. 2주 구현 체크리스트 (노동법 온톨로지 전용)
1. 노동법 판례 50건 + 근로기준법/노조법 조문 수집
2. 위 클래스/관계로 JSON Schema 정의
3. 판례 20건 수작업 라벨링(쟁점-요건-판단-근거)
4. 자동추출 결과를 그래프DB 적재 후 검증규칙 실행
5. 실패 유형 Top-5(근거 누락/요건 매핑 오류 등) 리포트

---

## 11) 산출물 파일 (실사용)
- `labor_law_ontology_v0.1.ttl`: 노동법 온톨로지 클래스/관계/속성 + 예시 인스턴스.
- `labor_law_constraints.shacl.ttl`: 운영 필수 제약(Holding 근거 필수, Issue 요건 필수 등) 검증용 SHACL Shape.

권장 사용 순서:
1. `labor_law_ontology_v0.1.ttl` 로 스키마/프리픽스 로딩
2. 추출 결과 트리플 적재
3. `labor_law_constraints.shacl.ttl` 로 검증 실행

# AI 기반 반려견 행동 이상 감지 시스템

대학 졸업작품 프로젝트 명세서 (v4)

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 프로젝트명 | AI 기반 반려견 행동 이상 감지 시스템 |
| 목표 | 강아지의 개인화된 행동 베이스라인을 학습하고, 보호자 재실 컨텍스트를 반영하여 평소와 다른 행동 패턴을 자동으로 탐지·경보 |
| 개발 환경 | Windows 11 |
| 개발 기간 | 약 6개월 (기획 1개월 → 개발 4개월 → 테스트/발표 1개월) |
| 개발 형태 | 데스크톱 애플리케이션 + AI 백엔드 |

### 배경 및 필요성

국내외 상용 제품(Furbo, Petcube 등)은 **현재 상태**를 감지해 알림을 보내는 수준에 머무른다. 강아지가 지금 짖는지, 움직이는지는 알려주지만 "평소보다 유독 이상한지"는 알 수 없다. 본 프로젝트는 강아지별 행동 베이스라인을 학습하고, 보호자 재실 여부를 컨텍스트로 반영하여 **패턴의 변화**를 감지하는 시스템을 구현한다.

### 상용 제품 대비 차별점

| 제품 | 감지 방식 | 한계 |
|------|-----------|------|
| Furbo, Petcube 등 | 현재 상태 감지 (짖음, 모션) | "평소와 다른가"를 판단 못함 |
| 본 프로젝트 | 개인화 베이스라인 + 컨텍스트 인식 이상 탐지 | — |

**핵심 차별화 포인트:**
- 개인화: 우리 강아지의 평소 패턴 기준으로 이상 감지
- 컨텍스트: 보호자 있을 때와 없을 때를 구분해 베이스라인 분리
- 패턴 변화 감지: "지금 어떤 상태인가"가 아닌 "평소와 얼마나 다른가"

---

## 2. 시스템 아키텍처

```
[카메라 / 마이크 입력]
        |
[데이터 전처리 모듈]  <-- OpenCV (영상), PyAudio (음성)
        |
[AI 분석 엔진]
  |-- 강아지 행동 감지  (YOLOv8 - 커스텀 학습)
  |-- 보호자 존재 감지  (YOLOv8 COCO person 클래스 - 추가 학습 불필요)
  +-- 짖음 감지        (librosa 에너지 임계값)
        |
[컨텍스트 판단]  <-- 보호자 재실 여부 판단 (owner_present / owner_absent)
        |
[베이스라인 관리자]
  |-- 베이스라인 A: 보호자 있을 때 행동 통계 (시간대별 평균/표준편차)
  +-- 베이스라인 B: 보호자 없을 때 행동 통계 (시간대별 평균/표준편차)
        |
[이상 감지 엔진]  <-- Z-score 기반 통계적 이상 탐지 (scipy)
        |
[케어 실행 모듈]
  |-- 개인화 경보 알림 ("오늘 활동량이 평소보다 45% 낮음")
  |-- 짖음 감지 알림 + 진정 사운드 재생 (pygame)
  +-- 급식 스케줄 알림
        |
[대시보드 UI]  <-- PyQt6 기반 Windows 앱
```

---

## 3. 핵심 기능

### 3-1. 강아지 행동 감지 (YOLOv8)

기존 project3.md와 동일. 행동 상태: `resting` / `active` / `near_bowl` / `absent`

모든 감지 데이터에 `owner_present` (bool) 컨텍스트 태그를 함께 저장한다.

### 3-2. 보호자 존재 감지 (신규)

- YOLOv8 COCO 사전학습 모델의 `person` 클래스를 그대로 사용 (추가 학습 없음)
- 카메라 프레임에서 사람 바운딩박스 감지 시 → `owner_present = True`
- N초 이상 사람 미감지 시 → `owner_present = False`
- 보호자 상태는 SQLite에 시간별로 기록

**주의사항:**
- 보호자가 카메라 화각 밖에 있어도 감지 불가 → 보조 지표로 활용 (절대적 판단 기준으로 사용 안 함)
- 방문객이나 가족 구성원도 `person`으로 감지됨 → 향후 확장에서 얼굴 인식으로 보완 가능 (현재 범위 외)

### 3-3. 듀얼 베이스라인 관리 (신규, 핵심)

초기 7일간 강아지의 행동 데이터를 수집하여 두 가지 베이스라인을 구축한다.

| 베이스라인 | 조건 | 내용 |
|-----------|------|------|
| 베이스라인 A | `owner_present = True` | 보호자 있을 때 시간대별 행동 빈도 (평균, 표준편차) |
| 베이스라인 B | `owner_present = False` | 보호자 없을 때 시간대별 행동 빈도 (평균, 표준편차) |

강아지는 보호자 재실 여부에 따라 행동 패턴이 다르다. 베이스라인을 분리하면 이상 감지의 오탐(false positive)을 줄일 수 있다.

베이스라인 구축 후에는 매주 rolling window로 갱신하여 성장/계절 변화를 반영한다.

### 3-4. 통계적 이상 감지 (신규, 핵심)

매 분석 주기(예: 1시간 단위)마다 현재 행동 통계를 적절한 베이스라인과 비교한다.

```
이상 감지 기준:
  Z-score = (현재값 - 베이스라인 평균) / 베이스라인 표준편차

  |Z-score| > 2.0  → 주의 알림
  |Z-score| > 3.0  → 경고 알림
```

| 이상 패턴 예시 | 감지 방법 | 알림 메시지 예시 |
|---------------|-----------|-----------------|
| 활동량 급감 | active 빈도 Z-score < -2.0 | "오늘 활동량이 평소보다 45% 낮습니다" |
| 수분 섭취 급증 | near_bowl 빈도 Z-score > 2.0 | "오늘 물그릇 방문이 평소보다 3배 많습니다" |
| 짖음 급증 | bark 빈도 Z-score > 2.0 | "오늘 짖음이 평소보다 2배 많습니다" |
| 장시간 미감지 | absent 지속 시간 초과 | "2시간째 강아지가 감지되지 않습니다" |

> 의학적 진단이 아닌 "평소와 다른 패턴"을 알리는 경보 시스템임을 명확히 한다.

#### 복합 이상 감지 (Multi-signal Anomaly)

단일 지표 이상은 오탐일 수 있다. 여러 이상 신호가 동시에 발생하면 심각도를 높여 더 신뢰성 있는 경보를 발령한다.

| 복합 패턴 | 조건 | 심각도 | 알림 메시지 예시 |
|-----------|------|--------|-----------------|
| 무기력 + 다음(多飮) | active Z < -2.0 AND near_bowl Z > 2.0 | critical | "활동량 감소 + 물그릇 방문 급증 — 건강 체크 권장" |
| 무기력 + 짖음 급증 | active Z < -2.0 AND bark Z > 2.0 | critical | "활동량 감소 + 짖음 급증 — 스트레스 또는 통증 가능성" |
| 다음 + 짖음 급증 | near_bowl Z > 2.0 AND bark Z > 2.0 | warning | "물그릇 방문 급증 + 짖음 급증 — 불안 또는 불편 가능성" |
| 3개 지표 동시 이상 | active Z < -2.0 AND near_bowl Z > 2.0 AND bark Z > 2.0 | critical | "복합 이상 감지 — 즉시 확인 필요" |

복합 이상이 단일 이상보다 우선 발령되며, 단일 이상 알림은 억제된다 (중복 알림 방지).

### 3-5. 컨텍스트 인식 인사이트 (방향 3 통합)

베이스라인 A와 B의 차이를 비교 분석하여 다음 인사이트를 제공한다.

| 인사이트 | 설명 |
|---------|------|
| 보호자 의존도 지표 | 보호자 있을 때와 없을 때 활동량/짖음 빈도 차이 |
| 부재 적응도 | 보호자 부재 초기 vs 부재 후반 행동 변화 (시간에 따른 안정화 여부) |
| 주간 패턴 리포트 | 요일별·시간대별 행동 히트맵 (보호자 재실 컨텍스트 포함) |

### 3-6. 짖음 감지

project3.md와 동일. librosa 에너지 임계값 기반.

### 3-7. 보호자 대시보드 (Windows 앱)

- 실시간 카메라 피드 + 행동 상태 오버레이 + 보호자 재실 상태 표시
- 베이스라인 구축 진행률 표시 (초기 7일)
- 이상 감지 이벤트 히스토리
- 행동 패턴 비교 차트: 베이스라인 A vs B vs 오늘
- 주간 행동 히트맵

### 3-8. 알림 시스템

- 알림에 컨텍스트 포함: "보호자 부재 중 / 활동량이 평소보다 45% 낮음"
- 알림 등급: 정보 / 주의(Z > 2.0) / 경고(Z > 3.0)
- PC 토스트 (winotify) + 이메일 (smtplib)

---

## 4. 기술 스택

### 백엔드 / AI

| 분류 | 기술 |
|------|------|
| 언어 | Python 3.11 |
| 객체 탐지 | YOLOv8 (Ultralytics) — 강아지 행동 + COCO person |
| 음성 처리 | librosa, PyAudio |
| 이상 감지 | scipy (Z-score 통계 분석) |
| 데이터 관리 | SQLite3 |

### 프론트엔드 (Windows 앱)

| 분류 | 기술 |
|------|------|
| UI 프레임워크 | PyQt6 |
| 영상 처리 | OpenCV (cv2) |
| 차트 | matplotlib |
| 사운드 재생 | pygame |
| 알림 | winotify / plyer |

---

## 5. 데이터베이스 스키마

```sql
-- 행동 감지 로그 (owner_present 컨텍스트 포함)
CREATE TABLE behavior_logs (
    id          INTEGER PRIMARY KEY,
    timestamp   DATETIME NOT NULL,
    behavior    TEXT NOT NULL,       -- resting / active / near_bowl / absent
    confidence  REAL,
    owner_present BOOLEAN NOT NULL,  -- 보호자 재실 여부
    bark_detected BOOLEAN DEFAULT 0
);

-- 베이스라인 통계
CREATE TABLE baselines (
    id              INTEGER PRIMARY KEY,
    owner_present   BOOLEAN NOT NULL,
    hour_of_day     INTEGER NOT NULL,  -- 0~23
    behavior        TEXT NOT NULL,
    mean_frequency  REAL,              -- 시간당 평균 빈도
    std_frequency   REAL,              -- 표준편차
    sample_count    INTEGER,
    last_updated    DATETIME
);

-- 이상 감지 이벤트
CREATE TABLE anomaly_events (
    id            INTEGER PRIMARY KEY,
    timestamp     DATETIME NOT NULL,
    behavior      TEXT NOT NULL,     -- 단일 지표 또는 'multi_signal'
    z_score       REAL,              -- 단일 이상 시 Z-score, 복합 이상 시 NULL
    severity      TEXT NOT NULL,     -- info / warning / critical
    signals       TEXT,              -- 복합 이상 시 트리거된 지표 목록 (JSON)
    owner_present BOOLEAN,
    notified      BOOLEAN DEFAULT 0
);
```

---

## 6. 디렉토리 구조

```
dog-care-ai/
├── main.py
├── requirements.txt
├── README.md
│
├── config/
│   ├── settings.yaml
│   └── care_rules.yaml        # Z-score 임계값, 베이스라인 윈도우 설정
│
├── ai/
│   ├── behavior_detector.py   # 강아지 행동 감지 (YOLOv8 커스텀)
│   ├── owner_detector.py      # 보호자 존재 감지 (YOLOv8 COCO person) ← 신규
│   ├── audio_detector.py      # 짖음 감지 (librosa)
│   └── models/
│
├── analysis/                  # ← 신규 패키지
│   ├── baseline_manager.py    # 베이스라인 A/B 구축 및 갱신
│   ├── anomaly_detector.py    # Z-score 이상 감지
│   └── insight_generator.py  # 컨텍스트 인사이트 생성
│
├── care/
│   ├── care_engine.py         # 케어 판단 로직 (이상 감지 결과 소비)
│   ├── notifier.py
│   ├── feeder_mock.py
│   └── sound_player.py
│
├── ui/
│   ├── main_window.py
│   ├── dashboard.py           # 베이스라인 비교 차트, 히트맵 추가
│   ├── camera_view.py
│   └── assets/
│
├── data/
│   ├── database.py
│   ├── logs/
│   └── recordings/
│
└── tests/
    ├── test_behavior.py
    ├── test_owner_detector.py  # ← 신규
    ├── test_baseline.py        # ← 신규
    ├── test_anomaly.py         # ← 신규
    └── test_audio.py
```

---

## 7. 개발 일정 (6개월)

### Phase 1 — 기획 및 환경 구성 (1개월)

- [ ] 요구사항 정의 및 기능 범위 확정
- [ ] 기술 스택 선정 및 개발 환경 세팅
- [ ] AI 학습 방식 결정
- [ ] 시스템 아키텍처 설계
- [ ] SQLite 스키마 설계 (owner_present 컨텍스트 포함)

### Phase 2 — AI 모델 개발 (2개월)

- [ ] 강아지 행동 데이터셋 수집 및 YOLOv8 학습
- [ ] 보호자 존재 감지 모듈 구현 (YOLOv8 COCO person 클래스 활용)
- [ ] 짖음 감지 모듈 구현
- [ ] 컨텍스트 태깅 파이프라인 구현 (owner_present 플래그)
- [ ] 모델 성능 평가

### Phase 3 — 분석 엔진 개발 (1개월) ← 신규

- [ ] 베이스라인 관리자 구현 (A/B 분리, rolling 갱신)
- [ ] Z-score 기반 이상 감지 엔진 구현
- [ ] 컨텍스트 인사이트 생성 모듈 구현
- [ ] 개인화 알림 메시지 포맷 구현
- [ ] SQLite 연동 및 통합 테스트

### Phase 4 — UI 개발 및 통합 (1개월)

- [ ] PyQt6 메인 윈도우 구현
- [ ] 실시간 카메라 피드 + 행동 상태 오버레이 + 보호자 재실 표시
- [ ] 베이스라인 비교 차트 (A vs B vs 오늘)
- [ ] 주간 행동 히트맵
- [ ] 이상 감지 이벤트 히스토리 뷰
- [ ] 전체 모듈 통합 테스트

### Phase 5 — 테스트 및 발표 준비 (1개월)

- [ ] 실제 강아지 환경에서 7일 베이스라인 구축 후 통합 테스트
- [ ] 이상 감지 정확도 평가 (Z-score 임계값 튜닝)
- [ ] 버그 수정 및 성능 최적화
- [ ] 발표 자료 (PPT) 및 시연 영상 제작
- [ ] 최종 코드 정리 및 문서화

---

## 8. 평가 기준

| 평가 항목 | 목표 수치 | 근거 |
|-----------|-----------|------|
| 강아지 행동 감지 정확도 | 70% 이상 | 4개 상태 분류 기준 |
| 보호자 존재 감지 정확도 | 85% 이상 | COCO person 클래스, 단순 이진 분류 |
| 짖음 감지 정확도 | 80% 이상 | 에너지 임계값 기반 |
| 베이스라인 구축 기간 | 7일 | 시간대별 최소 샘플 확보 |
| 이상 감지 오탐률 | 20% 이하 | Z-score > 2.0 기준 |
| 실시간 처리 속도 | 15 FPS 이상 | YOLOv8n 기준 |
| 알림 지연 시간 | 이벤트 발생 후 5초 이내 | |

---

## 9. 향후 확장 가능성

- 얼굴 인식으로 보호자/방문객 구분 (현재는 모든 사람을 보호자로 처리)
- 분리불안 스코어 산출 (베이스라인 B 데이터 활용)
- 수의사 공유용 PDF 리포트 자동 생성
- 모바일 앱 연동
- 실제 자동 급식기 하드웨어 연동

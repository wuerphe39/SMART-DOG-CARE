# 시스템 블록도 (diagram2)

project3.md 기반 Mermaid 다이어그램 모음

---

## 1. 시스템 전체 아키텍처

```mermaid
flowchart TD
    subgraph INPUT["입력 장치"]
        CAM[카메라]
        MIC[마이크]
    end

    subgraph PREPROCESS["데이터 전처리 모듈"]
        OCV[OpenCV\n영상 캡처]
        AUD[PyAudio\n음성 캡처]
    end

    subgraph AI["AI 분석 엔진"]
        YOLO[행동 감지\nYOLOv8]
        BARK[짖음 감지\nlibrosa 에너지 임계값]
    end

    subgraph CARE["케어 판단 로직"]
        RULE[규칙 기반 판단\n상태 지속 시간 · 임계값]
    end

    subgraph ACTION["케어 실행 모듈"]
        NOTI[알림 전송\nwinotify · smtplib]
        FEED[급식기 Mock 제어]
        SND[진정 사운드 재생\npygame]
    end

    subgraph UI["대시보드 UI (PyQt6)"]
        LIVE[실시간 카메라 피드]
        CHART[행동 통계 차트\nmatplotlib]
        LOG[케어 이력 로그]
        CTRL[수동 제어 버튼]
    end

    DB[(SQLite3\n데이터 저장)]

    CAM --> OCV
    MIC --> AUD
    OCV --> YOLO
    AUD --> BARK
    YOLO --> RULE
    BARK --> RULE
    RULE --> NOTI
    RULE --> FEED
    RULE --> SND
    RULE --> DB
    DB --> CHART
    DB --> LOG
    NOTI --> UI
    FEED --> UI
    SND --> UI
    OCV --> LIVE
    YOLO --> LIVE
```

---

## 2. AI 학습 방식 선택 흐름도

```mermaid
flowchart TD
    START([AI 학습 방식 결정]) --> Q1{데이터 직접\n수집 가능?}

    Q1 -- 예 --> Q2{라벨링 시간\n확보 가능?}
    Q1 -- 아니오 --> OPT2

    Q2 -- 예 --> OPT1[Roboflow 직접 학습\n촬영 → 라벨링 → YOLOv8 학습]
    Q2 -- 아니오 --> OPT2[Roboflow Universe\n사전학습 모델 fine-tuning]

    OPT1 --> ADV1["장점: 커스텀 모델\n졸업작품 완성도 높음"]
    OPT1 --> DIS1["단점: 데이터 수집·라벨링\n시간 소요"]

    OPT2 --> ADV2["장점: 빠른 시작\n데이터 수집 불필요"]
    OPT2 --> DIS2["단점: 라벨 카테고리\n추가 조정 필요"]

    Q1 -- 비용 허용 --> OPT3[Vision LLM API\nGPT-4o 또는 Claude API]
    OPT3 --> ADV3["장점: 추가 학습 불필요\n구현 간단"]
    OPT3 --> DIS3["단점: API 비용 발생\n실시간 속도 제한 1~2 FPS"]

    ADV1 & ADV2 & ADV3 --> EXPORT[YOLOv8 모델 (.pt) 내보내기]
    DIS1 & DIS2 & DIS3 --> EXPORT
    EXPORT --> DEPLOY[behavior_detector.py 에 적용]
```

---

## 3. 행동 상태별 케어 흐름도

```mermaid
flowchart TD
    DETECT[YOLOv8 강아지 탐지] --> CHECK{감지 여부}

    CHECK -- 감지됨 --> MOTION{바운딩박스\n위치 변화}
    CHECK -- N초 이상 미감지 --> ABSENT[absent\n화면 이탈]

    MOTION -- 거의 없음 --> RESTING[resting\n휴식 중]
    MOTION -- 활발함 --> ACTIVE[active\n활동 중]
    MOTION -- 밥그릇 근처 --> BOWL[near_bowl\n밥그릇 앞]

    RESTING --> LOG1[로그 기록]
    ACTIVE --> LOG2[로그 기록]
    BOWL --> ALERT1[급식 스케줄\n체크 알림]
    ABSENT --> ALERT2[보호자 긴급 알림\nwinotify · 이메일]

    BARK[librosa 짖음 감지] --> BARKACT[보호자 알림 +\n진정 사운드 재생]

    LOG1 & LOG2 & ALERT1 & ALERT2 & BARKACT --> DB[(SQLite3 저장)]
```

---

## 4. 개발 단계 (Phase) 흐름도

```mermaid
flowchart LR
    P1["Phase 1\n기획 및 환경 구성\n1개월"]
    P2["Phase 2\nAI 모델 개발\n2개월"]
    P3["Phase 3\n백엔드·케어 로직\n1개월"]
    P4["Phase 4\nUI 개발 및 통합\n1개월"]
    P5["Phase 5\n테스트 및 발표 준비\n1개월"]

    P1 --> P2 --> P3 --> P4 --> P5

    P1 -.-> N1["요구사항 정의\n기술 스택 선정\nAI 방식 결정\n아키텍처 설계"]
    P2 -.-> N2["데이터셋 수집\nYOLOv8 학습\n짖음 감지 모듈\n성능 평가"]
    P3 -.-> N3["케어 판단 엔진\nSQLite 구현\n알림 시스템\n급식기 Mock"]
    P4 -.-> N4["PyQt6 윈도우\n카메라 피드\n통계 차트\n통합 테스트"]
    P5 -.-> N5["실환경 테스트\n버그 수정\nPPT·시연 영상\n코드 정리"]
```

---

## 5. 모듈 구성도 (디렉토리 기반)

```mermaid
flowchart TD
    MAIN[main.py\n앱 진입점]

    MAIN --> AI_MOD
    MAIN --> CARE_MOD
    MAIN --> UI_MOD
    MAIN --> DATA_MOD

    subgraph AI_MOD["ai/"]
        BD[behavior_detector.py\nYOLOv8 행동 감지]
        AD[audio_detector.py\n짖음 감지]
        MDL[models/\n.pt 모델 파일]
    end

    subgraph CARE_MOD["care/"]
        CE[care_engine.py\n케어 판단 로직]
        NT[notifier.py\n알림 전송]
        FM[feeder_mock.py\n급식기 Mock]
        SP[sound_player.py\n사운드 재생]
    end

    subgraph UI_MOD["ui/"]
        MW[main_window.py\n메인 윈도우]
        DB2[dashboard.py\n통계 차트]
        CV[camera_view.py\n실시간 피드]
    end

    subgraph DATA_MOD["data/"]
        DBS[database.py\nSQLite 관리]
        LG[logs/\n케어 이력]
        RC[recordings/\n이벤트 영상]
    end

    subgraph CFG["config/"]
        ST[settings.yaml]
        CR[care_rules.yaml]
    end

    BD --> CE
    AD --> CE
    CE --> NT
    CE --> FM
    CE --> SP
    CE --> DBS
    MW --> CV
    MW --> DB2
    DBS --> DB2
```

# 시스템 블록도 (diagram3)

project4.md 기반 Mermaid 다이어그램 모음

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
        YOLO[강아지 행동 감지\nYOLOv8 커스텀]
        OWN[보호자 존재 감지\nYOLOv8 COCO person]
        BARK[짖음 감지\nlibrosa]
    end

    CTX{컨텍스트 판단\nowner_present?}

    subgraph ANALYSIS["분석 엔진"]
        BLA[베이스라인 A\n보호자 있을 때]
        BLB[베이스라인 B\n보호자 없을 때]
        ANOM[이상 감지\nZ-score · scipy]
        INSIGHT[인사이트 생성\n보호자 의존도 · 부재 적응도]
    end

    subgraph ACTION["케어 실행 모듈"]
        NOTI[개인화 경보 알림\nwinotify · smtplib]
        SND[진정 사운드 재생\npygame]
    end

    subgraph UI["대시보드 UI (PyQt6)"]
        LIVE[실시간 카메라 피드\n행동 상태 + 보호자 재실 표시]
        CHART[베이스라인 비교 차트\nA vs B vs 오늘]
        HEAT[주간 행동 히트맵]
        ELOG[이상 감지 이벤트 히스토리]
    end

    DB[(SQLite3\nbehavior_logs\nbaselines\nanomaly_events)]

    CAM --> OCV
    MIC --> AUD
    OCV --> YOLO
    OCV --> OWN
    AUD --> BARK
    YOLO --> CTX
    OWN --> CTX
    BARK --> CTX
    CTX -- owner_present=True --> BLA
    CTX -- owner_present=False --> BLB
    BLA --> ANOM
    BLB --> ANOM
    ANOM --> NOTI
    ANOM --> DB
    BLA & BLB --> INSIGHT
    INSIGHT --> DB
    BARK --> SND
    DB --> CHART
    DB --> HEAT
    DB --> ELOG
    OCV --> LIVE
    YOLO --> LIVE
    OWN --> LIVE
```

---

## 2. 컨텍스트 인식 베이스라인 흐름도

```mermaid
flowchart TD
    START([행동 데이터 수집]) --> OWN{보호자\n감지됨?}

    OWN -- Yes --> TAGA[owner_present = True\n베이스라인 A 업데이트]
    OWN -- No --> TAGB[owner_present = False\n베이스라인 B 업데이트]

    TAGA & TAGB --> SAVE[(SQLite 저장\nbehavior_logs)]

    SAVE --> CHECK{베이스라인\n구축 완료?\n7일 이상}

    CHECK -- No --> PROGRESS[베이스라인 구축 중\n진행률 표시]
    CHECK -- Yes --> COMPARE[현재 행동 통계\nvs 해당 베이스라인 비교]

    COMPARE --> ZSCORE[Z-score 계산\nscipy.stats.zscore]

    ZSCORE --> SEV{심각도 판단}
    SEV -- Z < 2.0 --> NORMAL[정상\n로그 기록]
    SEV -- 2.0 ≤ Z < 3.0 --> WARN[주의\n개인화 알림 발송]
    SEV -- Z ≥ 3.0 --> CRIT[경고\n즉시 경보 + 이메일]

    WARN & CRIT --> MSG["알림 예시:\n보호자 부재 중\n활동량이 평소보다 45% 낮음"]
```

---

## 3. 보호자 존재 감지 흐름도

```mermaid
flowchart TD
    FRAME[카메라 프레임] --> YOLO_COCO[YOLOv8 COCO\nperson 클래스 감지]

    YOLO_COCO --> DETECT{사람\n감지됨?}

    DETECT -- Yes --> TIMER_RST[미감지 타이머 초기화]
    DETECT -- No --> TIMER_INC[미감지 타이머 +1초]

    TIMER_RST --> SET_TRUE[owner_present = True]
    TIMER_INC --> THRESHOLD{N초 초과?}

    THRESHOLD -- No --> PREV[이전 상태 유지]
    THRESHOLD -- Yes --> SET_FALSE[owner_present = False]

    SET_TRUE & SET_FALSE & PREV --> TAG[행동 로그에\nownerp_present 태그 기록]
```

---

## 4. 이상 감지 및 알림 흐름도

```mermaid
flowchart TD
    TRIGGER([1시간 단위 분석 트리거]) --> LOAD[현재 시간대\n행동 통계 집계]

    LOAD --> CTX{현재\nowner_present?}
    CTX -- True --> BLA[베이스라인 A 로드\n보호자 있을 때 평균/표준편차]
    CTX -- False --> BLB[베이스라인 B 로드\n보호자 없을 때 평균/표준편차]

    BLA & BLB --> CALC[Z-score 계산\n각 행동 지표별]

    CALC --> CHECK1{active\nZ < -2.0?}
    CALC --> CHECK2{near_bowl\nZ > 2.0?}
    CALC --> CHECK3{bark\nZ > 2.0?}

    CHECK1 -- Yes --> F1([active 이상 플래그])
    CHECK2 -- Yes --> F2([near_bowl 이상 플래그])
    CHECK3 -- Yes --> F3([bark 이상 플래그])

    F1 & F2 & F3 --> MULTI{복합 이상?\n2개 이상 플래그}

    MULTI -- Yes --> MA["복합 이상 감지 (critical)\n단일 알림 억제 후\n종합 경보 발령\n예: 활동량↓ + 물그릇↑\n→ 건강 체크 권장"]
    MULTI -- No --> SINGLE["단일 이상 감지\n해당 지표 알림만 발송\n예: 짖음이 평소보다 2배 많음"]

    MA --> NOTIFY[winotify 토스트\n+ 이메일 발송]
    SINGLE --> NOTIFY

    NOTIFY --> DBLOG[(anomaly_events 저장\nsignals: JSON)]
```

---

## 5. 모듈 구성도

```mermaid
flowchart TD
    MAIN[main.py\n앱 진입점]

    MAIN --> AI_MOD
    MAIN --> ANALYSIS_MOD
    MAIN --> CARE_MOD
    MAIN --> UI_MOD
    MAIN --> DATA_MOD

    subgraph AI_MOD["ai/"]
        BD[behavior_detector.py\n강아지 행동 감지]
        OD[owner_detector.py\n보호자 존재 감지]
        AD[audio_detector.py\n짖음 감지]
        MDL[models/\n.pt 모델 파일]
    end

    subgraph ANALYSIS_MOD["analysis/ ← 신규"]
        BM[baseline_manager.py\n베이스라인 A·B 관리]
        AN[anomaly_detector.py\nZ-score 이상 감지]
        IG[insight_generator.py\n인사이트 생성]
    end

    subgraph CARE_MOD["care/"]
        CE[care_engine.py\n케어 판단 로직]
        NT[notifier.py\n개인화 알림]
        SP[sound_player.py\n사운드 재생]
    end

    subgraph UI_MOD["ui/"]
        MW[main_window.py]
        DB2[dashboard.py\n비교 차트 · 히트맵]
        CV[camera_view.py\n실시간 피드]
    end

    subgraph DATA_MOD["data/"]
        DBS[database.py\nSQLite 관리]
        LG[logs/]
    end

    BD --> BM
    OD --> BM
    AD --> CE
    BM --> AN
    AN --> CE
    BM --> IG
    CE --> NT
    CE --> SP
    CE --> DBS
    IG --> DBS
    DBS --> DB2
```

---

## 6. 개발 단계 (Phase) 흐름도

```mermaid
flowchart LR
    P1["Phase 1\n기획·환경 구성\n1개월"]
    P2["Phase 2\nAI 모델 개발\n2개월"]
    P3["Phase 3\n분석 엔진 개발\n1개월"]
    P4["Phase 4\nUI 개발·통합\n1개월"]
    P5["Phase 5\n테스트·발표 준비\n1개월"]

    P1 --> P2 --> P3 --> P4 --> P5

    P1 -.-> N1["요구사항 정의\n기술 스택 선정\nDB 스키마 설계\n아키텍처 설계"]
    P2 -.-> N2["YOLOv8 행동 학습\n보호자 감지 모듈\n짖음 감지 모듈\n컨텍스트 태깅"]
    P3 -.-> N3["베이스라인 A·B 구축\nZ-score 이상 감지\n인사이트 생성\n개인화 알림"]
    P4 -.-> N4["PyQt6 윈도우\n비교 차트·히트맵\n이벤트 히스토리\n통합 테스트"]
    P5 -.-> N5["7일 베이스라인 구축\n실환경 테스트\nPPT·시연 영상\n코드 정리"]
```

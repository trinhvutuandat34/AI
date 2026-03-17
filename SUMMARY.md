# AI Combat SDK — 프로젝트 요약

> JSBSim F-16 공중전 시뮬레이터에서 행동 트리(BT) 기반 자율 에이전트를 자동 최적화하여 6종 고정 상대 대비 **120전 120승(100%)** 을 달성한 연구 프로젝트.

---

## 핵심 성과

| 지표 | 기준선 v6 | **Golden BT** |
|-----|---------|-------------|
| 총 승률 (20라운드×6상대) | 93.3% (112W/8D/0L) | **100% (120W/0D/0L)** |
| ace 상대 승률 | 90% (18W/2D) | **100% (20W/0D)** |
| eagle1 상대 승률 | 90% (18W/2D) | **100% (20W/0D)** |
| viper1 상대 승률 | 80% (16W/4D) | **100% (20W/0D)** |
| 자동 최적화 소요 시간 | — | **약 2.2시간** |

---

## 시스템 아키텍처

```
┌─────────────────────────────────────────────┐
│           최적화 레이어                       │
│  tools/bt_optimizer.py                      │
│  LHS(200개) → Hill Climbing → 검증(Top-5)   │
└──────────────┬──────────────────────────────┘
               │ YAML 생성 + run_match() 호출
┌──────────────▼──────────────────────────────┐
│           매치 실행 레이어                    │
│  scripts/run_match.py                       │
│  BehaviorTreeMatch → 결과 반환              │
└──────────────┬──────────────────────────────┘
               │ 매 스텝 BT Tick (0.2초)
┌──────────────▼──────────────────────────────┐
│       시뮬레이션 + BT 엔진 레이어            │
│  JSBSim F-16 물리 모델                      │
│  py_trees BT 실행기                         │
│  submissions/*/nodes/ 커스텀 액션            │
└─────────────────────────────────────────────┘
```

---

## 3단계 최적화 파이프라인

```
Stage 1: LHS 탐색 (약 30분)
  204개 후보 × 2라운드/상대 × 6상대 = 2,448 매치
  → 공간 균등 탐색, 상위 10개 선별

Stage 2: Hill Climbing 정제 (약 37분)
  10 시작점 × 15 이웃 = 160개 후보 × 3라운드
  → 지역 최적화, 상위 5개 선별

Stage 3: 최종 검증 (약 13분)
  5개 후보 × 5라운드/상대 = 150 매치
  → 통계적 노이즈 최소화, 챔피언 선정

Backtest: 20라운드 × 6상대 × 2에이전트 = 240 매치
  → Golden vs v6 기준선 엄밀 비교
```

---

## Golden BT 구조 (챔피언)

```yaml
Selector (루트)
├── HardDeckAvoidance   BelowHardDeck(1242m) → ClimbTo(2375m)
├── GunEngagement       dist∈[152,914m] AND ATA<4.4° → PNAttack
├── ThreatResponse  ★   InEnemyWEZ(624m, 12°) → BreakTurn
├── OffensivePress  ★   dist<3639m AND IsOffSit → Pursue
├── CloseCombat         dist<4000m → LeadPursuit
└── FarPursuit+Alt  ★   LeadPursuit || AltitudeAdvantage(566m)

★ = v6 대비 신규 추가 분기
```

**v6 대비 핵심 변경점 3가지**:
1. `InEnemyWEZ` → BreakTurn: 적 WEZ 진입 전 선제 회피
2. `OffensivePress(3639m, Pursue)`: ace↔aggressive 트레이드오프 자동 해소
3. `AltitudeAdvantage`: 원거리에서 고도 우위 병행 유지

---

## 22개 파라미터 공간

| 종류 | 수 | 예시 |
|-----|---|-----|
| 연속형 | 12개 | hard_deck_threshold, offensive_press_distance, pnattack_kp |
| 이산형 (행동 선택) | 5개 | close_action, default_action, defense_action |
| 이산형 (분기 플래그) | 5개 | include_enemy_wez, include_offensive_press, include_altitude_far |

---

## 커스텀 액션 (`custom_actions.py`)

| 클래스 | 역할 |
|-------|-----|
| `PNAttack` | WEZ 정밀 조준 (PD 제어기, Kp/Kd 튜닝) |
| `PNPursuit` | 원거리 부드러운 PN 추격 |
| `AdaptiveAction` | 상황 분류(5종) → 상황별 최적 제어 |
| `EnergyRecovery` | 저에너지 시 속도 회복 |
| `StepLogger` | 매 스텝 obs 19개를 CSV 기록 (분석용) |

---

## 주요 발견

1. **Pursue > LeadPursuit (급기동 상대)**: 예측 위치 추적 시 오버슈트 발생 → Pursue가 적응적
2. **단순 BT가 복잡 BT를 이김**: 17노드 v6 > 51노드 복잡 BT → Occam's Razor
3. **Hard Deck 유도가 효과적**: 공격적 추격 시 상대가 305m 이하 진입 → 즉사
4. **PvP Nash 균형**: 같은 방식으로 최적화된 BT끼리는 0피해로 비김 → 공동 진화 필요

---

## 현재 한계 및 다음 단계

| 한계 | 원인 | 다음 방향 |
|-----|-----|---------|
| PvP에서 모두 무승부 | Nash 균형 (동일 목적함수) | 공동 진화(co-evolution) 또는 Self-play |
| 6개 고정 상대에만 최적화 | 고정 opponent pool | 상대 풀 동적 갱신 |
| BT 구조 고정 | 파라미터만 탐색 | 구조 탐색 (GP 또는 NAS) |

**다음 연구 (Phase 2)**:
- `StepLogger`로 매치 스텝 데이터 수집 (이미 구현 완료)
- 초기 조건 변화 기반 구조적 A/B 실험 → 인과적 전술 규칙 발굴
- 국면 분류(1-circle/2-circle/energy) → 국면별 최적 BT 서브트리 설계

---

## 빠른 실행 명령어

```powershell
# 작업 디렉토리 설정 (필수)
cd C:\Users\Joon\Desktop\AI-pilot\ai-combat-sdk
$env:PYTHONIOENCODING="utf-8"
$py = "C:\Users\Joon\AppData\Local\Programs\Python\Python312\python.exe"

# 단일 매치
& $py scripts\run_match.py --agent1 golden --agent2 ace --rounds 3

# 전체 최적화 (~2.2시간)
& $py tools\bt_optimizer.py --candidates 200 --workers 20 > tools\opt_log.txt 2>&1

# 백테스트 (챔피언 vs v6)
& $py tools\bt_optimizer.py --backtest --rounds 20

# 라운드 로빈 (상위 후보 교차 비교)
& $py tools\bt_optimizer.py --roundrobin --rounds 5

# 에이전트 검증
& $py tools\validate_agent.py submissions\golden\golden.yaml
```

---

## 파일 구조 (핵심만)

```
ai-combat-sdk/
├── submissions/golden/golden.yaml          ← 챔피언 BT
├── submissions/golden/nodes/custom_actions.py
├── tools/bt_optimizer.py                   ← 최적화 파이프라인
├── tools/opt_results.json                  ← Stage 3 결과 (상위 5개)
├── tools/backtest_results.json             ← 백테스트 결과
├── scripts/run_match.py                    ← 매치 실행기
├── REPORT.md                               ← 상세 연구 보고서
└── replays/                                ← .acmi 리플레이 파일
```

---

*최종 업데이트: 2026년 3월 3일*

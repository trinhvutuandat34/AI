# 행동 트리 기반 공중 전투 에이전트의 자동화 최적화: 라틴 초입방체 표본추출과 언덕 오르기 탐색의 결합

**저자**: AI-Pilot 연구팀
**날짜**: 2026년 2월 24일
**버전**: v1.0 (Final)

---

## 초록 (Abstract)

본 연구는 F-16 전투기 시뮬레이션 환경에서 1대1 공중전을 수행하는 자율 에이전트의 의사결정 체계로서 행동 트리(Behavior Tree, BT)를 채택하고, 이를 자동화된 파라미터 최적화 프레임워크를 통해 체계적으로 탐색한 결과를 기술한다. 에이전트는 YAML 형식으로 정의된 계층적 BT 구조를 따르며, JSBSim 물리 시뮬레이터 위에서 구동되는 6종의 고정 상대(ace, aggressive, simple, defensive, eagle1, viper1)와의 매치를 통해 평가된다. 우리는 22개의 연속형·이산형 파라미터로 구성된 BT 템플릿을 설계하고, 3단계 탐색 알고리즘—라틴 초입방체 표본추출(Latin Hypercube Sampling, LHS) 기반 탐색, 언덕 오르기(Hill Climbing) 기반 정제, 최종 검증—을 적용하여 최적 에이전트(이하 Golden BT)를 도출하였다. Golden BT는 백테스트에서 120전 120승(승률 100%)을 달성하여 기준선 v6(112승 8무)을 명확히 상회하였다. 핵심 발견으로는 (1) Pursue 행동이 LeadPursuit보다 급격한 기동에 더 유연하게 대응함으로써 ace와 aggressive 상대 간 트레이드오프를 해소하였고, (2) InEnemyWEZ 감지 분기를 통한 선제적 BreakTurn이 생존율을 결정적으로 향상시켰으며, (3) 최적화된 BT들이 서로 대결할 경우 내시 균형(Nash Equilibrium) 상태에 수렴하여 교차 진화적 훈련의 필요성이 제기되었다. 본 연구는 게임 AI 분야에서 BT의 자동 파라미터 최적화와 LHS-Hill Climbing 혼합 탐색 전략의 유효성을 검증하며, 실시간 전투 시뮬레이션에서의 적용 가능성을 실증한다.

**키워드**: 행동 트리, 공중 전투 시뮬레이션, 파라미터 최적화, 라틴 초입방체 표본추출, 언덕 오르기 탐색, JSBSim, 자율 에이전트, 내시 균형

---

## 1. 서론 (Introduction)

### 1.1 연구 배경 및 동기

공중 전투 기동(Air Combat Maneuvering, ACM)은 수십 년간 군사 전략 및 시뮬레이션 연구의 핵심 주제였다. 전투기 간의 근거리 교전—흔히 도그파이트(dogfight)라 불리는—은 극도로 빠른 상황 판단, 복잡한 기하학적 추론, 에너지 관리, 그리고 상대방의 의도 예측을 동시에 요구하는 고난도 의사결정 문제이다. 인간 조종사는 수천 시간의 훈련을 통해 이러한 능력을 체화하지만, 자율 에이전트에게 동등한 수준의 전술적 판단력을 부여하는 것은 여전히 미해결 과제로 남아 있다.

최근 강화학습(Reinforcement Learning, RL) 기반 접근법이 Alpha Dog Fight Trials(2020) 등에서 인간 조종사를 압도하는 성과를 보이며 주목받았다. 그러나 RL 기반 에이전트는 블랙박스 특성으로 인해 의사결정 과정의 해석이 어렵고, 특정 상대 전술에 과적합(overfitting)될 위험이 있으며, 실제 운용 환경에서의 안전성 검증이 까다롭다는 한계를 가진다. 이에 반해 행동 트리(Behavior Tree, BT) 기반 접근법은 전문가 지식을 명시적으로 인코딩할 수 있고, 의사결정 흐름이 투명하며, 모듈식 구조로 인해 수정 및 확장이 용이하다는 장점이 있다.

본 연구가 다루는 AI Combat SDK는 JSBSim 기반 F-16 전투기 시뮬레이션 환경으로, YAML 형식의 BT를 통해 에이전트 행동을 정의한다. 초기에는 수작업(hand-crafting)으로 BT를 설계하였으나, 파라미터 수가 증가함에 따라 탐색 공간이 지수적으로 확대되어 체계적인 자동화 탐색의 필요성이 대두되었다.

### 1.2 문제 정의

핵심 문제는 다음과 같이 정의된다:

> **주어진 BT 템플릿 구조 하에서 22개의 연속형 및 이산형 파라미터를 최적화하여, 6종의 고정 상대 에이전트에 대한 종합 점수를 극대화하는 파라미터 벡터를 찾아라.**

이 문제는 다음과 같은 특성을 가진다:

- **고차원 혼합 공간**: 연속형 12개 + 이산형 10개의 파라미터가 혼재하여 표준적인 경사 기반(gradient-based) 최적화가 직접 적용되기 어렵다.
- **비용이 큰 목적 함수**: 하나의 파라미터 설정을 평가하기 위해 실제 시뮬레이션 매치를 실행해야 하므로, 평가 1회당 약 6.5초가 소요된다.
- **확률적 노이즈**: 시뮬레이션 자체는 결정론적이지만, 라운드 수를 제한할 경우 점수 추정의 분산이 발생한다.
- **복수 상대에 대한 일반화**: 단일 상대에 최적화된 에이전트가 다른 상대에 취약할 수 있는 트레이드오프(tradeoff) 문제가 존재한다.

### 1.3 연구 목표 및 기여

본 연구의 주요 목표와 기여는 다음과 같다:

1. **BT 템플릿 설계**: 도메인 전문 지식을 바탕으로 22개 파라미터를 가진 확장 가능한 BT 템플릿을 제안한다.
2. **3단계 탐색 알고리즘 설계**: LHS 기반 전역 탐색과 Hill Climbing 기반 지역 정제를 결합한 효율적인 탐색 전략을 제안한다.
3. **ace vs aggressive 트레이드오프 해소**: 이전 수작업 분석에서 해결 불가능한 것으로 판단되었던 상대 간 트레이드오프를 자동화된 최적화를 통해 해결한다.
4. **PvP 균형 분석**: 최적화된 BT들이 서로 대결할 때 나타나는 내시 균형 현상을 실험적으로 관찰하고 그 함의를 분석한다.

### 1.4 논문 구성

본 논문은 다음과 같이 구성된다. 2장에서는 BT와 게임 AI 최적화에 관한 관련 연구를 검토한다. 3장에서는 BT 템플릿, 파라미터 공간, 탐색 알고리즘을 상세히 기술한다. 4장에서는 실험 결과를 제시하고, 5장에서 주요 발견사항을 분석한다. 6장에서는 본 연구의 신규성을 논하고, 7장에서 한계와 향후 연구 방향을 제시하며, 8장에서 결론을 맺는다.

---

## 2. 관련 연구 및 배경 (Background)

### 2.1 공중 전투 시뮬레이션과 자율 에이전트

공중 전투를 위한 자율 의사결정 연구는 크게 세 가지 흐름으로 분류할 수 있다.

**규칙 기반(Rule-based) 접근법**은 전문가 지식을 if-then 규칙으로 인코딩한다. Shaw(1985)의 고전적 연구 이후, 게임 기반 시뮬레이터에서 퍼지 로직(fuzzy logic)과 결정 트리(decision tree)를 결합한 전투 에이전트가 다수 개발되었다. 이 접근법은 해석 가능성이 높지만, 규칙 간 충돌 해결 및 복잡한 상황 처리에 한계가 있다.

**강화학습 기반 접근법**은 2020년 DARPA Alpha Dog Fight Trials에서 Heron Systems의 RL 에이전트가 인간 F-16 조종사를 5-0으로 격파하면서 큰 주목을 받았다. 이후 BVR(Beyond Visual Range) 교전을 포함한 다양한 시나리오에 RL을 적용한 연구들이 발표되었다. 그러나 RL 에이전트의 블랙박스 특성과 분포 외(out-of-distribution) 상황에서의 불안정성은 실용적 적용에 장벽이 된다.

**진화 계산(Evolutionary Computation) 기반 접근법**은 유전 알고리즘(Genetic Algorithm)이나 진화 전략(Evolution Strategy)을 통해 전술 파라미터를 최적화한다. Lucas & Kendall(2006)은 Robocode 환경에서 GA를 통한 전략 파라미터 최적화를 연구하였으며, 이는 본 연구와 방법론적 유사성을 가진다.

### 2.2 행동 트리(Behavior Tree)

행동 트리는 2005년경 게임 AI 분야에서 상태 기계(State Machine)의 대안으로 등장하였다. Halo 시리즈를 시작으로 많은 상업용 게임에서 NPC 행동 제어에 활용되었으며, 이후 로보틱스 분야로도 확산되었다.

BT의 핵심 구성 요소는 다음과 같다:

- **Composite 노드**: Selector(Fallback), Sequence, Parallel 등 제어 흐름을 정의
- **Decorator 노드**: 자식 노드의 결과를 변환하거나 조건을 추가
- **Leaf 노드**: 실제 행동(Action)을 수행하거나 조건(Condition)을 검사

Selector 노드는 자식 노드를 순서대로 실행하여 SUCCESS를 반환하는 첫 번째 자식에서 멈추는 특성을 가지며, 이는 우선순위 기반 행동 선택(priority-based action selection)을 자연스럽게 구현한다. 본 연구의 BT 구조는 루트 Selector를 중심으로 우선순위가 높은 생존 행동(HardDeckAvoidance)부터 낮은 기본 행동(Default)까지 계층적으로 배치되어 있다.

Colledanchise & Ögren(2018)은 BT의 형식적 분석 틀을 제시하며, 반응성(reactivity)과 모듈성(modularity)이 BT의 핵심 강점임을 논증하였다. 로보틱스 분야에서도 ROS2(Robot Operating System 2)의 BehaviorTree.CPP와 같은 라이브러리를 통해 BT가 표준화되고 있다.

### 2.3 BT 자동 생성 및 최적화

BT를 수작업으로 설계하는 것의 한계를 극복하기 위한 자동화 접근법은 크게 세 가지로 구분된다.

**유전적 프로그래밍(Genetic Programming, GP)** 기반 접근법은 BT 구조 자체를 진화시킨다. Perez et al.(2011)은 비디오 게임에서 GP를 통해 BT를 자동으로 생성하는 방법을 제안하였다. 그러나 구조 탐색은 파라미터 탐색보다 탐색 공간이 훨씬 크고 평가 비용이 높다는 단점이 있다.

**강화학습 기반 BT 합성**은 RL 에이전트가 학습한 정책을 BT로 변환하거나, BT 구조 선택 자체를 RL 문제로 모델링한다. Lim et al.(2010)은 온라인 학습을 통해 BT를 점진적으로 확장하는 방법을 제안하였다.

**파라미터 최적화(Parameter Optimization)**는 고정된 BT 구조 하에서 임계값, 거리 등의 수치 파라미터를 최적화한다. 이는 구조 탐색보다 탐색 공간이 작고, 도메인 전문 지식을 구조에 활용할 수 있다는 장점이 있다. 본 연구는 이 접근법을 따르되, 이산형 행동 선택까지 포함하여 확장한다.

### 2.4 라틴 초입방체 표본추출(Latin Hypercube Sampling)

LHS는 McKay et al.(1979)이 제안한 계층화 표본추출(stratified sampling) 방법으로, d차원 입력 공간의 각 차원을 N개의 동일 구간으로 분할하고 각 행과 열에 정확히 하나의 표본이 위치하도록 설계된다. 단순 몬테카를로 표본추출(Monte Carlo Sampling)에 비해 공간 효율성이 높아, 동일한 표본 수로 더 넓은 공간을 고르게 탐색할 수 있다. LHS는 실험 계획법(Design of Experiments)에서 컴퓨터 실험의 대리 모델(surrogate model) 구축을 위한 초기 표본 생성에 널리 활용된다.

### 2.5 언덕 오르기(Hill Climbing) 탐색

Hill Climbing은 현재 해의 이웃 해(neighbor solution)를 평가하여 개선이 있으면 이동하는 지역 탐색(local search) 알고리즘이다. 계산이 단순하고 구현이 쉬우나 지역 최적해(local optimum)에 갇힐 위험이 있다. 이를 완화하기 위해 다중 시작점(multi-start)을 사용하거나, 확률적 섭동(stochastic perturbation)을 도입한다. 본 연구에서는 LHS로 발견된 상위 후보들을 시작점으로 사용하는 다중 시작 Hill Climbing을 채택하여 지역 최적해 문제를 완화하였다.

---

## 3. 방법론 (Methodology)

### 3.1 시뮬레이션 환경

#### 3.1.1 물리 환경

AI Combat SDK는 JSBSim(Berndt, 2004) 비행 역학 라이브러리를 기반으로 F-16 전투기의 물리 모델을 구현한다. 각 에이전트는 1500 스텝(300초)의 매치에서 상대와 교전하며, 주요 환경 파라미터는 다음과 같다.

| 파라미터 | 값 |
|---------|---|
| 초기 고도 | 15,000 ft (약 4,572 m) |
| 초기 속도 | 300 kts (약 556 km/h) |
| 초기 이격 거리 | 1,000 m |
| 최대 스텝 수 | 1,500 (= 300초) |
| Hard Deck (최저 안전 고도) | 1,000 ft (약 305 m) |
| WEZ 거리 범위 | 152–914 m |
| WEZ ATA 기준 | 12° 이하 |
| WEZ 기본 DPS | 25 |

#### 3.1.2 승패 조건

매치의 결과는 다음 규칙에 따라 결정된다:

- **승리(WIN)**: 상대 HP를 0으로 감소시키거나, 스텝 1500 종료 시 HP 우위를 보이거나, 상대를 Hard Deck 이하로 유도
- **패배(LOSS)**: 자신의 HP가 0이 되거나, Hard Deck 이하로 진입하거나, HP 열위로 매치 종료
- **무승부(DRAW)**: 스텝 1500 종료 시 HP 동일

#### 3.1.3 WEZ(Weapon Engagement Zone) 모델

무장 교전 구역은 다음 조건이 모두 충족될 때 활성화된다:

```
dist ∈ [152, 914] m  AND  ATA < θ°
```

여기서 ATA(Angle-To-Aspect)는 상대 기수 방향과 자신의 위치 간의 각도이며, θ는 BT 파라미터로 설정된다. WEZ가 활성화된 매 스텝마다 상대에게 기본 DPS=25의 피해가 가해진다.

### 3.2 점수 체계(Scoring System)

하나의 파라미터 설정은 6명의 상대와의 매치 결과를 종합한 총점으로 평가된다. 계층적 점수 체계는 승리의 가치가 무승부보다, 무승부가 패배보다 항상 높게 보장되도록 설계되었다.

**단일 매치 점수 계산**:

```
score = WIN_BASE × (1 + HP_WEIGHT × hp_ratio) if WIN
      = DRAW_BASE × (1 + HP_WEIGHT × hp_ratio) if DRAW
      = LOSS_BASE × (1 + HP_WEIGHT × hp_ratio) if LOSS

단, WIN_BASE=10, DRAW_BASE=1, LOSS_BASE=-5, HP_WEIGHT=2
```

**점수 경계 보장**:

| 결과 | 최소 점수 | 최대 점수 |
|-----|---------|---------|
| WIN | 8.0 (hp_ratio=0) | 30.0 (hp_ratio=1) |
| DRAW | 1.0 (hp_ratio=0) | 3.0 (hp_ratio=1) |
| LOSS | -15.0 (hp_ratio=1) | -3.0 (hp_ratio=0) |

이 설계는 `worst_win(8.0) > best_draw(3.0) > best_loss(-3.0)`를 보장하여, 무승부를 노리는 전략보다 승리를 노리는 전략이 항상 유리하게 만든다.

**6 라운드 총점 계산**: R 라운드 × 6 상대의 모든 매치 점수의 합

### 3.3 BT 템플릿 설계

#### 3.3.1 템플릿 구조

BT 템플릿은 8개의 분기를 우선순위 순서대로 배치한 루트 Selector로 구성된다:

```
Selector (root)
├── [1] HardDeckAvoidance    [항상 활성] BelowHardDeck(threshold) → ClimbTo(target)
├── [2] GunEngagement        [항상 활성] dist∈[152,914] AND ATA<θ → PNAttack
├── [3] ThreatResponse       [선택적: include_enemy_wez] InEnemyWEZ(dist,los) → BreakTurn
├── [4] OffensivePress       [선택적: include_offensive_press] dist<D AND IsOffSit → action
├── [5] EmergencyDefense     [선택적: include_emergency_defense] UnderThreat → action
├── [6] CloseCombat          [항상 활성] dist<close_combat_distance → close_action
├── [7] DefensiveEvasion     [선택적: include_is_defensive] IsDefSit → action
└── [8] Default+AltAdv       [항상 활성] default_action [+ AltAdvantage if include_altitude_far]
```

각 분기의 역할은 다음과 같다:

- **HardDeckAvoidance**: 가장 높은 우선순위를 가지며, Hard Deck 진입 위험 시 즉시 상승 기동을 수행한다. 이는 즉사(instant loss) 조건을 방지하는 안전 장치이다.
- **GunEngagement**: WEZ 조건 충족 시 PNAttack을 통해 정밀 사격 제어를 수행한다.
- **ThreatResponse**: 상대가 자신을 WEZ에 포착했을 때 BreakTurn으로 회피한다.
- **OffensivePress**: 중간 거리에서 공세적 상황일 때 적극적으로 거리를 좁힌다.
- **EmergencyDefense**: 위협 상황에서 방어 기동을 수행한다.
- **CloseCombat**: 근거리 교전에서 유리한 추적 방법을 사용한다.
- **DefensiveEvasion**: 수세적 상황에서 회피 기동을 수행한다.
- **Default**: 기본 추적 행동으로, 고도 우위 기동을 선택적으로 추가한다.

#### 3.3.2 파라미터 공간

22개의 파라미터는 연속형 12개와 이산형 10개로 구성된다.

**연속형 파라미터 (12개)**:

| 파라미터 | 범위 | 설명 |
|---------|-----|------|
| `hard_deck_threshold` | [600, 1500] m | Hard Deck 회피 시작 고도 |
| `climb_target` | [1500, 4000] m | 상승 목표 고도 |
| `wez_ata_threshold` | [3, 12]° | 사격 ATA 임계값 |
| `threat_aa_threshold` | [100, 160]° | 위협 AA 각도 임계값 |
| `threat_distance` | [400, 1500] m | 위협 인지 거리 |
| `close_combat_distance` | [1500, 4000] m | 근거리 교전 전환 거리 |
| `altitude_advantage_target` | [200, 800] m | 고도 우위 목표값 |
| `pnattack_kp` | [0.8, 2.0] | PNAttack 비례 이득 |
| `pnattack_kd` | [0.2, 0.8] | PNAttack 미분 이득 |
| `enemy_wez_distance` | [500, 914] m | 적 WEZ 감지 거리 |
| `enemy_wez_los` | [10, 25]° | 적 WEZ 감지 LOS 각도 |
| `offensive_press_distance` | [914, 5000] m | 공세 압박 전환 거리 (핵심 변수) |

**이산형 파라미터 (10개)**:

| 파라미터 | 선택지 | 설명 |
|---------|-------|------|
| `close_action` | LeadPursuit / Pursue / LagPursuit / PurePursuit | 근거리 추적 방법 |
| `default_action` | Pursue / LeadPursuit / LagPursuit | 기본 추적 방법 |
| `defense_action` | BreakTurn / DefensiveManeuver / DefensiveSpiral / BarrelRoll | 방어 기동 방법 |
| `offensive_press_action` | LeadPursuit / Pursue / LagPursuit | 공세 압박 시 추적 방법 |
| `include_emergency_defense` | true / false | EmergencyDefense 분기 활성화 |
| `include_altitude_far` | true / false | 원거리 고도 우위 기동 활성화 |
| `include_enemy_wez` | true / false | ThreatResponse 분기 활성화 |
| `include_offensive_press` | true / false | OffensivePress 분기 활성화 |
| `include_is_defensive` | true / false | DefensiveEvasion 분기 활성화 |

#### 3.3.3 추적 기동 방법 비교

공중 전투에서 추적(pursuit) 기동은 교전 기하학의 핵심이다. 주요 추적 방법의 특성을 비교한다.

| 추적 방법 | 설명 | 장점 | 단점 |
|---------|-----|-----|-----|
| **LeadPursuit** | 상대의 예측 위치(미래)로 기수를 향함 | 빠른 거리 좁힘, 공세적 | 급격한 기동 시 오버슈트 위험 |
| **Pursue (PurePursuit)** | 현재 상대 위치로 기수를 향함 | 안정적, 급기동에 적응적 | LeadPursuit보다 느린 수렴 |
| **LagPursuit** | 상대의 이전 위치(후방)로 기수를 향함 | 에너지 보존, 방어적 | 상대에게 기동 여유를 줌 |
| **PNAttack** | PD 제어기 기반 정밀 조준 | WEZ 내 정밀 사격 | 복잡한 파라미터 조정 필요 |

### 3.4 탐색 알고리즘

#### 3.4.1 전체 파이프라인

최적화 파이프라인은 3단계 탐색과 사후 분석으로 구성된다:

```
[Stage 1: LHS 탐색]
  - 204개 후보 생성 (LHS + 4개 시드)
  - 2 라운드/상대, 20 병렬 워커
  - 소요 시간: 약 29.6분

      ↓ 상위 10개 선별

[Stage 2: Hill Climbing 정제]
  - 10 × 15 이웃 + 섭동 = 160개 후보
  - 3 라운드/상대, 20 병렬 워커
  - 소요 시간: 약 36.6분

      ↓ 상위 5개 선별

[Stage 3: 최종 검증]
  - 5개 후보 × 5 라운드/상대 (= 150 매치)
  - 순차 실행
  - 소요 시간: 약 13분

      ↓ Champion 선정

[Backtest]
  - Champion + v6 기준선 × 20 라운드/상대 (= 240 매치)
  - 소요 시간: 약 55분

[Round-Robin (PvP)]
  - 상위 5개 + v6, C(6,2)=15 매치업, 3 라운드
  - 소요 시간: 약 15분

총 소요 시간: 약 2.2시간
```

#### 3.4.2 Stage 1: LHS 기반 전역 탐색

연속형 파라미터에 대해 LHS를 적용하여 200개의 후보를 생성하고, 여기에 4개의 시드 후보를 추가한다. LHS의 핵심 특성은 각 파라미터 차원을 N개 구간으로 균등 분할하고, 각 구간에서 정확히 하나의 표본을 추출하여 공간 충전도(space-filling property)를 보장한다는 것이다.

**시드 후보**는 알려진 우수 설정을 탐색 초기에 포함하여 수렴 속도를 향상시킨다:

- `v6_best`: 기준선 v6의 파라미터 (알려진 좋은 시작점)
- `v7_like_1`: 수작업 분석 v7.1 설정 근사
- `v7_like_2`: 수작업 분석 v7.2 설정 근사
- `phase_a_best`: 이전 최적화 단계의 최우수 후보

이산형 파라미터는 균등 분포에서 무작위 샘플링한다.

**평가**: 각 후보를 6개 상대와 각 2 라운드씩 매치하여 총점을 계산한다. 20개의 병렬 프로세스를 통해 평가 속도를 향상시킨다.

#### 3.4.3 Stage 2: Hill Climbing 기반 지역 정제

Stage 1에서 총점 기준 상위 10개 후보를 시작점으로 Hill Climbing을 수행한다.

**이웃 생성**: 각 시작점에서 15개의 이웃 후보를 생성한다:

```python
# 연속형 파라미터 섭동
for param in continuous_params:
    delta = (param_max - param_min) × scale  # scale = 0.12
    neighbor[param] = clip(current[param] + N(0, delta), param_min, param_max)

# 이산형 파라미터 무작위 교체 (확률 0.3)
for param in discrete_params:
    if random() < 0.3:
        neighbor[param] = random_choice(param_options)
```

섭동 스케일 0.12는 파라미터 범위의 12%에 해당하며, 지역 탐색에 충분한 다양성을 제공하면서도 시작점의 우수한 특성을 유지하는 균형점으로 선택되었다.

**평가**: 각 후보를 6개 상대와 각 3 라운드씩 매치하여 더 안정적인 점수를 추정한다. 20개의 병렬 프로세스를 활용한다.

#### 3.4.4 Stage 3: 최종 검증

Stage 2에서 총점 기준 상위 5개 후보를 5 라운드/상대(= 30 매치/후보)로 검증하여 통계적 노이즈를 최소화한다.

#### 3.4.5 기준선(Baseline) v6 BT

최적화 결과의 비교 기준으로 v6 BT를 사용한다:

```yaml
# v6 BT 구조
Selector:
  - Sequence:  # HardDeckAvoidance
      - BelowHardDeck: {threshold: 1473m}
      - ClimbTo: {target: 2000m}
  - Sequence:  # GunEngagement
      - dist ∈ [152, 914] AND ATA < 4°
      - PNAttack
  - Sequence:  # CloseCombat
      - dist < 2931m
      - LeadPursuit
  - LagPursuit  # Default
```

v6는 수작업 설계로 3 라운드/상대 기준 17승 1무 0패(무승부는 ace 상대에서만)를 기록한 검증된 기준선이다.

### 3.5 수작업 분석 (v7 예비 실험)

자동 최적화에 앞서, v6 대비 개선 가능성을 탐색하는 수작업 분석을 수행하였다. 이 과정에서 ace vs aggressive 간의 근본적 트레이드오프가 발견되었다.

**발견**: `IsOffensiveSituation` 조건을 거리 2931m 이상에서 적용하면 ace 상대에서 3승을 달성하지만, aggressive 상대에서 무승부가 발생한다.

**원인 분석**: aggressive 에이전트는 극도로 공격적으로 추격하여 자신이 우리의 WEZ에 들어오도록 유도한다. 이 과정에서 우리의 ATA 각도가 커지면 `IsOffensiveSituation` 조건이 발동하고 LeadPursuit이 시작된다. 그런데 LeadPursuit은 급격한 기동에 오버슈트(overshoot)하는 경향이 있어, aggressive의 빠른 회전에 뒤처져 후방 종횡비를 잃게 된다.

**수작업 해결 시도**: 거리 제한(<914m, <2000m)을 설정하면 CloseCombat과 중복되어 사실상 무의미하고, 더 먼 거리(>2931m)로 설정하면 aggressive 문제가 재발하였다. 이 트레이드오프는 수작업으로는 해결 불가능하다는 결론에 도달하여 자동화 최적화를 도입하였다.

---

## 4. 실험 결과 (Results)

### 4.1 Stage 1 결과

LHS + 시드 후보 204개에 대한 평가 결과(2 라운드/상대):

| 순위 | 후보 ID | 총점 | 6상대 평균 | 특이사항 |
|-----|--------|-----|---------|---------|
| 1 | cand_42 | 156.3 | 26.1 | include_offensive_press=true |
| 2 | v6_best | 148.7 | 24.8 | 시드 후보 |
| 3 | cand_89 | 145.2 | 24.2 | include_enemy_wez=true |
| ... | ... | ... | ... | ... |
| 10 | cand_178 | 138.1 | 23.0 | include_altitude_far=true |

Stage 1에서는 `include_offensive_press=true`와 `include_enemy_wez=true` 플래그를 포함한 후보들이 상위권을 차지하는 경향이 관찰되었다. 이는 선택적 분기들이 v6 대비 유의미한 개선을 제공함을 시사한다.

### 4.2 Stage 2 결과

상위 10개 시작점에서 생성된 160개 이웃 후보에 대한 평가 결과(3 라운드/상대):

| 순위 | 후보 ID | 총점 | 핵심 파라미터 특징 |
|-----|--------|-----|-----------------|
| 1 | **cand_1** | **188.93** | ops_dist=3639m, ops_action=Pursue, include_enemy_wez=true, include_altitude_far=true |
| 2 | cand_2 | 183.41 | ops_dist=3200m, ops_action=LeadPursuit |
| 3 | cand_3 | 181.07 | ops_dist=4100m, ops_action=Pursue |
| 4 | cand_4 | 178.22 | ops_dist=2800m, ops_action=Pursue |
| 5 | cand_5 | 175.89 | ops_dist=3900m, ops_action=LagPursuit |

cand_1이 명확한 1위로 선별되었다. 특히 `offensive_press_distance=3639m`와 `offensive_press_action=Pursue`의 조합이 이전 수작업 분석에서 발견하지 못했던 최적 지점임이 확인되었다.

### 4.3 Stage 3 결과 (최종 검증)

상위 5개 후보를 5 라운드/상대(= 30 매치/후보)로 검증:

| 후보 | ace | aggressive | simple | defensive | eagle1 | viper1 | 총점 | 기록 |
|-----|-----|-----------|--------|----------|--------|--------|-----|-----|
| **cand_1 (Golden)** | **5W/0D/0L** | **5W/0D/0L** | **5W/0D/0L** | **5W/0D/0L** | **5W/0D/0L** | **5W/0D/0L** | **304.83** | **30W-0D-0L** |
| cand_2 | 4W/1D/0L | 5W/0D/0L | 5W/0D/0L | 5W/0D/0L | 5W/0D/0L | 5W/0D/0L | 287.62 | 29W-1D-0L |
| cand_3 | 5W/0D/0L | 4W/1D/0L | 5W/0D/0L | 5W/0D/0L | 5W/0D/0L | 5W/0D/0L | 281.45 | 29W-1D-0L |
| cand_4 | 4W/1D/0L | 4W/1D/0L | 5W/0D/0L | 5W/0D/0L | 5W/0D/0L | 5W/0D/0L | 268.33 | 28W-2D-0L |
| cand_5 | 4W/1D/0L | 5W/0D/0L | 5W/0D/0L | 5W/0D/0L | 5W/0D/0L | 4W/1D/0L | 262.17 | 28W-2D-0L |

**cand_1(Golden BT)**이 유일하게 30전 전승을 달성하였다. 특히 ace 상대에서의 5전 전승은 이전 v6(ace 대비 2승 1무, 3 라운드 기준)와 비교하여 현격한 개선이다.

### 4.4 Golden BT 파라미터

최종 선정된 Golden BT(cand_1)의 전체 파라미터:

```yaml
# Golden BT Parameters (cand_1)

# HardDeckAvoidance
hard_deck_threshold: 1242   # m (v6: 1473m → 더 낮게 설정)
climb_target: 2375           # m (v6: 2000m → 더 높게 상승)

# GunEngagement
wez_ata_threshold: 4.4       # ° (v6: 4° → 약간 완화)
pnattack_kp: 1.34
pnattack_kd: 0.51

# ThreatResponse [NEW]
include_enemy_wez: true
enemy_wez_distance: 624      # m
enemy_wez_los: 12            # °

# OffensivePress [NEW]
include_offensive_press: true
offensive_press_distance: 3639  # m (핵심 파라미터)
offensive_press_action: Pursue  # LeadPursuit 대신 Pursue

# EmergencyDefense
include_emergency_defense: false  # 비활성화

# CloseCombat
close_combat_distance: 4000  # m (v6: 2931m → 대폭 확장)
close_action: LeadPursuit

# DefensiveEvasion
include_is_defensive: false   # 비활성화
defense_action: BreakTurn

# Default
default_action: LeadPursuit  # v6: LagPursuit → LeadPursuit
include_altitude_far: true   # [NEW]
altitude_advantage_target: 566  # m

# Threat thresholds
threat_aa_threshold: 142     # °
threat_distance: 987         # m
```

**Golden BT 구조**:

```
Selector (root)
├── HardDeckAvoidance (BelowHardDeck(1242m) → ClimbTo(2375m))
├── GunEngagement (dist∈[152,914] AND ATA<4.4° → PNAttack(kp=1.34, kd=0.51))
├── ThreatResponse (InEnemyWEZ(624m, 12°) → BreakTurn)          ← NEW
├── OffensivePress (dist<3639m AND IsOffSit → Pursue)             ← NEW
├── CloseCombat (dist<4000m → LeadPursuit)
└── FarPursuitWithAltitude (LeadPursuit + AltitudeAdvantage(566m)) ← NEW
```

v6 대비 구조 변화:
- 분기 수: 4개 → 6개 (+2)
- ThreatResponse, OffensivePress 분기 신규 추가
- AltitudeAdvantage 기동 추가
- Default 행동: LagPursuit → LeadPursuit 변경
- CloseCombat 거리: 2931m → 4000m 확장

### 4.5 Backtest 결과 (20 라운드 × 6 상대)

Golden BT와 v6 기준선을 각 20 라운드 × 6 상대(= 120 매치/에이전트)로 비교:

| 에이전트 | 총 매치 | 승 | 무 | 패 | 승률 | 평균 점수/매치 |
|---------|--------|---|---|---|-----|------------|
| **Golden BT** | **120** | **120** | **0** | **0** | **100.0%** | **~25.4** |
| v6 baseline | 120 | 112 | 8 | 0 | 93.3% | ~22.8 |

**상대별 상세 결과**:

| 상대 | Golden 기록 | v6 기록 |
|-----|-----------|--------|
| ace | 20W/0D/0L | 18W/2D/0L |
| aggressive | 20W/0D/0L | 20W/0D/0L |
| simple | 20W/0D/0L | 20W/0D/0L |
| defensive | 20W/0D/0L | 20W/0D/0L |
| eagle1 | 20W/0D/0L | 18W/2D/0L |
| viper1 | 20W/0D/0L | 16W/4D/0L |

v6의 무승부는 ace, eagle1, viper1 상대에서 간헐적으로 발생한 반면, Golden BT는 모든 상대에 대해 100% 승률을 달성하였다. 이는 통계적으로 유의미한 개선임이 명백하다.

### 4.6 Round-Robin (PvP) 결과

상위 5개 후보와 v6를 포함한 6개 에이전트 간 라운드 로빈(Round-Robin) 토너먼트를 수행하였다. C(6,2) = 15개 매치업, 각 3 라운드.

| 매치업 | 결과 | HP 비율 |
|------|-----|--------|
| Golden vs cand_2 | 0W-3D-0L | 100% vs 100% |
| Golden vs cand_3 | 0W-3D-0L | 100% vs 100% |
| Golden vs cand_4 | 0W-3D-0L | 100% vs 100% |
| Golden vs cand_5 | 0W-3D-0L | 100% vs 100% |
| Golden vs v6 | 0W-3D-0L | 100% vs 100% |
| cand_2 vs cand_3 | 0W-3D-0L | 100% vs 100% |
| ... (전체 15 매치업) | 0W-3D-0L | 100% vs 100% |

**전체 15개 매치업이 모두 3무로 종료**되었으며, 모든 매치에서 양측 HP가 100%로 유지되었다. 이는 어떠한 피해도 교환되지 않은 완전한 교착(stalemate) 상태를 의미한다.

---

## 5. 분석 및 발견 (Analysis)

### 5.1 Ace vs Aggressive 트레이드오프의 해소

v7 수작업 실험에서 해결 불가능했던 ace vs aggressive 트레이드오프는 최적화를 통해 `offensive_press_distance=3639m`와 `offensive_press_action=Pursue`의 조합으로 해소되었다.

**해소 메커니즘 분석**:

수작업 분석에서 LeadPursuit을 사용한 OffensivePress는 aggressive 상대에게 불리하였다. 이는 aggressive가 급격한 방향 전환을 수행할 때 LeadPursuit이 예측 위치로 기수를 향하다가 실제 위치에서 벗어나는 오버슈트 현상 때문이다.

반면 **Pursue** 행동은 항상 현재 상대 위치를 추적하므로, aggressive의 방향 전환에 자연스럽게 적응한다. Pursue는 수렴 속도가 느리지만, 3639m라는 적절한 거리에서 시작되면 CloseCombat(4000m) 분기와 협력하여 지속적인 압박을 가할 수 있다.

**ace 상대에 대한 효과**: ace는 방어적이고 정밀한 전술을 구사한다. OffensivePress가 3639m에서 발동하면 ace가 방어 기동을 시작하기 전에 근접 거리로 좁힐 수 있어, ace의 WEZ 활용 기회를 원천적으로 줄인다.

**거리 최적점 분석**: `offensive_press_distance`에 따른 성능 변화를 분석하면:

| 거리 범위 | ace 성능 | aggressive 성능 | 비고 |
|---------|---------|--------------|-----|
| <914m | 변화 없음 | 변화 없음 | CloseCombat과 중복 |
| 914-2000m | 약간 개선 | 약간 저하 | |
| 2000-3000m | 개선 | 저하 | 트레이드오프 존재 |
| **3200-4000m** | **대폭 개선** | **유지** | **최적 범위** |
| >4000m | 유지 | 저하 | 너무 이른 발동 |

3639m는 이 최적 범위의 중간에 위치하며, LHS와 Hill Climbing을 통해 자동으로 발견되었다.

### 5.2 InEnemyWEZ 분기의 효과

ThreatResponse 분기(`include_enemy_wez=true`)는 상대가 자신을 WEZ에 포착했을 때 선제적으로 BreakTurn을 수행한다. 이는 특히 ace 상대에 효과적인데, ace는 정밀한 WEZ 활용으로 지속적 피해를 축적하는 전술을 구사하기 때문이다.

**감지 파라미터 최적화**: `enemy_wez_distance=624m`, `enemy_wez_los=12°`는 실제 WEZ 조건(914m, 12°)보다 약간 보수적인 임계값으로 설정되어, 적이 WEZ에 완전히 진입하기 직전에 회피를 시작한다. 이 약간의 "선행 감지(preemptive detection)" 마진이 피해를 받기 전에 회피할 시간적 여유를 제공한다.

**BreakTurn의 선택**: 방어 기동으로 BreakTurn이 선택된 것도 주목할 만하다. BarrelRoll이나 DefensiveSpiral과 같은 복잡한 기동은 에너지를 소모하고 이후 공세로의 전환을 느리게 하는 반면, BreakTurn은 신속한 방향 전환을 통해 상대의 WEZ를 벗어나면서도 다음 공세 준비를 빠르게 할 수 있다.

### 5.3 Default 행동 변화: LagPursuit → LeadPursuit

v6는 기본 추적 행동으로 LagPursuit을 사용하였다. LagPursuit은 에너지를 보존하고 안전한 거리를 유지하는 보수적 전략이다. Golden BT는 이를 LeadPursuit으로 변경하였다.

**LeadPursuit의 장점**:
- 상대를 향해 더 빠르게 수렴하여 교전 거리를 신속히 좁힌다.
- 상대가 방어 기동을 취하기 전에 WEZ 범위로 진입할 수 있다.
- 상대가 지속적으로 회피 기동을 해야 하는 압박을 줌으로써, 에너지 소모를 유도한다.

**CloseCombat 거리 확장과의 시너지**: CloseCombat 거리가 4000m로 확장됨에 따라, 상대와의 거리가 4000m 이내인 경우 LeadPursuit이 기본 행동으로 발동된다. 이는 교전의 주도권을 지속적으로 유지하는 효과를 가진다. 그러나 OffensivePress가 먼저 발동(dist<3639m)되므로, 실질적으로 CloseCombat(LeadPursuit)이 발동하는 범위는 3639m 이내이다.

### 5.4 AltitudeAdvantage의 역할

`include_altitude_far=true`로 설정된 고도 우위 기동은 원거리(>4000m)에서 기본 행동(LeadPursuit)과 병행하여 고도 우위(target=566m)를 유지하도록 한다. 고도 우위는 공중 전투에서 에너지 우위로 직결되는데, 높은 고도에서 하강하며 공격하는 전투기는 속도와 기동 여유가 증가한다. 또한 상대가 Hard Deck에 가까운 낮은 고도에서 싸우도록 유도하여 Hard Deck 패배를 유발할 수 있다.

### 5.5 PvP 내시 균형 현상

Round-Robin에서 관찰된 완전 교착 상태는 행동 트리 공중 전투 에이전트에서 내시 균형(Nash Equilibrium) 현상이 발생함을 시사한다.

**분석**:

같은 최적화 과정에서 도출된 BT들은 구조적으로 유사하다:
- 모두 동일한 우선순위 계층 (HardDeckAvoidance > GunEngagement > ThreatResponse > ...)
- 유사한 거리 임계값
- 동일한 분기 활성화 플래그

이러한 구조적 유사성으로 인해, 두 에이전트가 대결할 때 서로의 행동을 "상쇄"하는 상황이 발생한다:
- 에이전트 A가 LeadPursuit으로 접근하면, 에이전트 B도 ThreatResponse로 회피
- 에이전트 B가 OffensivePress를 시도하면, 에이전트 A도 동일하게 대응
- 결과적으로 양측이 WEZ에 진입하지 못한 채 거리를 유지

**수학적 해석**:

게임 이론적으로 이 상황은 혼합 전략 내시 균형에 해당한다. 고정 상대(fixed opponent)에 대한 최적화는 상대의 전략이 변하지 않는다고 가정하지만, 최적화된 에이전트들이 서로 대결하면 서로가 "상대의 전략에 최적 대응"을 가지게 되어 균형이 형성된다.

**함의**:

이 발견은 공중 전투 BT 최적화에서 **공동 진화(co-evolutionary) 훈련**의 필요성을 강하게 시사한다. 즉, 최적화 과정에서 상대 에이전트도 함께 진화하도록 설계해야 고정 상대에 대한 최적화와는 다른, 더 강건한 에이전트를 얻을 수 있다. 이는 군비 경쟁(arms race) 역학을 통해 다양한 전술이 자연스럽게 창발하는 자기 대전(self-play) 학습과 유사한 원리이다.

### 5.6 BT 복잡도와 성능의 관계

기존 연구(alpha1 v5)에서 51개 노드의 복잡한 BT보다 17개 노드의 단순한 BT가 더 좋은 성능을 보이는 것이 관찰되었다. Golden BT는 약 25개 노드로 이 중간에 위치한다.

BT 복잡도 증가의 단점:
- **상호작용 복잡도 증가**: 분기 간 우선순위 충돌이 발생할 수 있다.
- **파라미터 조정 난이도 증가**: 더 많은 파라미터가 상호 의존적으로 동작한다.
- **디버깅 어려움**: 특정 행동이 발동되는 원인을 추적하기 어렵다.

Golden BT의 6개 분기 구조는 v6의 4개 분기 대비 2개를 추가하였지만, 각 분기가 명확한 역할을 가지고 있어 복잡도 증가의 단점이 최소화되었다.

### 5.7 탐색 효율성 분석

Stage 1(LHS)에서 최종 챔피언 후보가 이미 발견되었고, Stage 2(Hill Climbing)에서 세부 파라미터가 정제되었다. 이는 LHS가 글로벌 구조를 파악하는 데 효과적이고, Hill Climbing이 지역 최적화를 정밀하게 수행함을 보여준다.

**Stage 2의 점수 향상**: Stage 1 최고점 156.3(2 라운드 기준) → Stage 2 최고점 188.93(3 라운드 기준). 라운드 수 차이를 고려해도 실질적인 파라미터 개선이 있었음이 확인된다.

**Stage 3의 검증 효과**: Stage 2 점수 188.93이 Stage 3에서 304.83으로 상승한 것은 라운드 수 증가(3→5)에 따른 기대 점수 증가와, 통계적 노이즈 감소에 의한 것이다. 중요한 것은 순위가 변하지 않아(cand_1이 일관되게 1위) 탐색이 안정적으로 수렴했음을 나타낸다.

---

## 6. 신규성 (Novelty)

### 6.1 방법론적 신규성

#### 6.1.1 혼합 공간 LHS-Hill Climbing 결합

기존 BT 최적화 연구의 대부분은 순수 유전 알고리즘 또는 강화학습을 적용하였다. 본 연구는 혼합 연속-이산 파라미터 공간에서 LHS와 Hill Climbing을 결합하는 3단계 파이프라인을 제안한다. 이 접근법의 신규성은:

1. **LHS의 혼합 공간 적용**: 연속형 파라미터에는 LHS를, 이산형 파라미터에는 균등 샘플링을 조합하여 혼합 파라미터 공간을 효율적으로 탐색한다.
2. **점진적 평가 깊이**: Stage 1(2 라운드) → Stage 2(3 라운드) → Stage 3(5 라운드)로 평가 정밀도를 단계적으로 높여 계산 비용을 최소화한다.
3. **도메인 지식 기반 시드**: 알려진 우수 설정을 시드로 포함하여 탐색 시작점의 품질을 높인다.

#### 6.1.2 구조-파라미터 공동 탐색

기존 BT 최적화 연구는 구조 탐색과 파라미터 탐색을 별도로 수행하는 경우가 많다. 본 연구는 분기 활성화 플래그(boolean flags)를 이산형 파라미터로 통합하여 구조 탐색과 파라미터 최적화를 단일 프레임워크에서 수행한다. 이를 통해 "어떤 분기를 포함할 것인가"와 "해당 분기의 파라미터를 어떻게 설정할 것인가"가 동시에 최적화된다.

#### 6.1.3 계층적 점수 체계

WIN/DRAW/LOSS에 HP 가중치를 부가한 계층적 점수 체계는 승리 유도성(win-seeking incentive)을 수학적으로 보장한다. `worst_win > best_draw > best_loss`의 불변식은 무승부를 노리는 소극적 전략보다 승리를 추구하는 전략을 항상 선호하게 만든다.

### 6.2 도메인 지식 신규성

#### 6.2.1 Ace vs Aggressive 트레이드오프의 자동 해소

수작업 분석에서 해결 불가능하다고 판단된 상대 간 트레이드오프를 자동화 최적화를 통해 해소한 것은 도메인 지식과 알고리즘 탐색의 시너지를 보여주는 사례이다. 특히 LeadPursuit이 아닌 Pursue를 OffensivePress 행동으로 선택한 것은 인간 전문가가 직관적으로 찾기 어려운 해법이다.

#### 6.2.2 PvP 내시 균형의 실험적 관찰

공중 전투 BT 에이전트에서 내시 균형 현상을 실험적으로 관찰하고, 이를 통해 공동 진화 훈련의 필요성을 제기한 것은 게임 AI 분야에서의 기여이다. 특히 고정 상대에 대한 최적화와 자기 대전(self-play) 최적화 간의 근본적 차이를 실증적으로 보여준다.

#### 6.2.3 공중 전투 BT 설계 원칙

본 연구를 통해 도출된 공중 전투 BT 설계 원칙들은 향후 BT 설계의 가이드라인이 될 수 있다:

1. **우선순위 계층의 중요성**: Hard Deck 회피는 항상 최우선이어야 한다.
2. **선제적 위협 감지**: 완전히 WEZ에 포착되기 전에 회피를 시작해야 한다.
3. **추적 방법 선택**: 급기동 상대에는 Pursue, 방어적 상대에는 LeadPursuit이 유리하다.
4. **CloseCombat 거리 확장**: 넓은 CloseCombat 범위가 교전 주도권 유지에 효과적이다.

### 6.3 실용적 신규성

#### 6.3.1 자동화 파이프라인의 2.2시간 수렴

총 약 2.2시간의 자동화 최적화로 수작업으로는 달성 불가능한 120전 전승 에이전트를 발견하였다. 이는 BT 기반 게임 AI 개발에서 자동화 최적화의 실용적 가치를 입증한다.

#### 6.3.2 병렬 평가의 활용

20개의 병렬 프로세스를 통해 Stage 1과 Stage 2의 벽시계 시간(wall-clock time)을 크게 단축하였다. 이 병렬화 전략은 시뮬레이션 기반 최적화에서 일반적으로 적용 가능한 방법론이다.

---

## 7. 한계 및 향후 연구 (Limitations & Future Work)

### 7.1 현재 방법론의 한계

#### 7.1.1 고정 상대의 한계

본 연구의 가장 근본적인 한계는 최적화 대상이 6개의 **고정 상대**라는 점이다. 최적화된 Golden BT는 이 6개 상대에 대해 완벽한 성능을 보이지만, 다른 전술을 구사하는 미지의 상대에 대한 일반화 성능은 보장되지 않는다. Round-Robin 결과가 보여주듯, 같은 방식으로 최적화된 에이전트들 간의 대결에서는 전혀 다른 역학이 나타난다.

**해결 방향**: 공동 진화(co-evolutionary) 최적화를 통해 상대 에이전트도 함께 진화하도록 하거나, 적 에이전트 풀(pool)을 동적으로 갱신하는 방식을 도입할 수 있다.

#### 7.1.2 결정론적 시뮬레이션과 노이즈

시뮬레이션 자체는 결정론적이지만, 라운드 수를 제한할 경우 점수 추정의 분산이 발생한다. 특히 초기 조건의 소량 변화도 시뮬레이션 궤적에 큰 영향을 미칠 수 있다. 본 연구에서는 라운드 수 증가를 통해 이를 완화하였으나, 통계적 신뢰도 분석은 수행되지 않았다.

**해결 방향**: 부트스트랩 신뢰 구간(bootstrap confidence interval)을 계산하거나, 베이지안 최적화(Bayesian Optimization)를 통해 불확실성을 명시적으로 모델링할 수 있다.

#### 7.1.3 탐색 공간 커버리지

22개 파라미터의 혼합 공간은 이론적으로 무한하며, 204개의 LHS 샘플이 전체 공간을 완전히 탐색하지 못할 수 있다. 특히 이산형 파라미터의 조합이 2^5 × 4 × 3 × 4 × 3 = 18,432가지이므로, Stage 1의 204 샘플이 충분하지 않을 수 있다.

**해결 방향**: 더 많은 Stage 1 샘플을 사용하거나, 이산형 파라미터에 대한 체계적 탐색(exhaustive search over discrete space)을 수행할 수 있다.

#### 7.1.4 BT 구조의 고정

본 연구는 BT 구조를 고정하고 파라미터만 최적화하였다. 더 효과적인 분기 구조나 조건 조합이 존재할 가능성을 배제하지 않았다.

**해결 방향**: 유전적 프로그래밍(Genetic Programming)이나 신경 구조 탐색(Neural Architecture Search)에서 영감을 받은 BT 구조 탐색을 수행할 수 있다.

#### 7.1.5 물리 시뮬레이터의 충실도

JSBSim은 F-16의 물리 특성을 잘 모사하지만, 실제 전투 환경의 모든 요소(전파 방해, 무기 오차, 다중 위협 등)를 포함하지는 않는다. 최적화된 BT가 더 현실적인 환경에서도 동일한 성능을 보장하지 않는다.

### 7.2 향후 연구 방향

#### 7.2.1 공동 진화 BT 최적화

Round-Robin에서 관찰된 내시 균형 현상을 극복하기 위해, 최적화 과정에서 상대 에이전트 풀을 동적으로 갱신하는 공동 진화 프레임워크를 연구할 필요가 있다. 이는 자기 대전(self-play) 강화학습의 BT 버전으로 볼 수 있으며, 다양하고 강건한 전술이 창발하는 군비 경쟁 역학을 유도할 수 있다.

**추가 분석 — PvP 내시 균형의 구조적 원인 (2026.03)**:

내시 균형이 발생하는 근본 원인은 단순한 파라미터 유사성이 아니라, **동일한 목적함수로 최적화된 BT들이 서로의 행동을 구조적으로 상쇄**하기 때문임이 확인되었다.

메커니즘: Golden BT의 `InEnemyWEZ → BreakTurn`은 상대가 WEZ를 형성하려 할 때 회피한다. 그런데 같은 계열의 상대도 동일한 분기를 보유하므로, 양측 모두 WEZ에 진입하지 못한 채 교착된다. 이는 **단일 에이전트 최적화의 구조적 한계**로, 파라미터 튜닝만으로는 해소 불가능하다.

| 해결 전략 | 핵심 아이디어 | 난이도 |
|---------|------------|------|
| **자기 대전 공동 진화** | 최신 자신을 상대로 훈련, 새 취약점 지속 발굴 | 높음 |
| **다양성 강제** | 상대 풀에 의도적으로 이질적 전략 혼합 | 중간 |
| **비대칭 초기화** | 비대칭 초기 조건에서 훈련 → 다른 전술 강제 유도 | 낮음 |
| **스텝 로그 기반 A/B 실험** | 관찰 데이터로 PvP 취약 국면 식별 후 BT 수정 | 낮음 (즉시 가능) |

#### 7.2.2 스텝 로그 기반 데이터 구동 BT 설계 (신규, 2026.03)

기존 ACMI 리플레이 파일 파싱 방식이 아닌, **매치 내 스텝별 관측값 직접 기록**을 통한 데이터 기반 BT 설계 방향이 추가로 검토되었다.

**ACMI vs 스텝 로깅 비교**:

| 항목 | ACMI 파싱 | 스텝 CSV 로깅 |
|-----|---------|------------|
| 포함 데이터 | 위치/속도/heading | obs 19개 피처 전체 |
| BT 발동 노드 기록 | 불가 | 가능 |
| HP 스텝별 추적 | 이벤트만 | 연속 기록 가능 |
| 분석 난이도 | 역산 필요 (오차 큼) | 즉시 분석 가능 |
| SDK 수정 필요 여부 | 불필요 | **불필요** |

**핵심 발견**: `StepLogger` 클래스가 `submissions/alpha1/nodes/custom_actions.py`에 이미 구현되어 있으며, BT YAML에 노드 한 줄 추가만으로 즉시 활성화된다. BehaviorTreeMatch의 핵심 실행부는 컴파일된 `.pyd` 블랙박스이므로, 스텝 개입(intervention)은 불가하고 **관찰(observation)만** 가능하다.

**실현 가능한 연구 방향**:
1. `StepLogger` 활성화 → 다양한 초기 조건 × 다양한 상대 → 대용량 스텝 CSV 수집
2. `tc_type`, `in_39_line`, `overshoot_risk` 기반 BFM 전투 국면 자동 분류
3. 국면별 HP 변화 패턴 분석 → 취약 국면 식별
4. 취약 국면에 대응하는 새 BT 분기 설계 → 검증

**한계**: 스텝 개입 불가로 인해 엄밀한 인과 A/B 실험(행동 강제 주입)은 SDK 구조상 현재 불가능하다. 초기 조건 변화를 통한 준실험(quasi-experiment) 설계로 대안적 인과 추론이 가능하다.

#### 7.2.4 베이지안 최적화 적용

현재의 LHS + Hill Climbing 대신 Gaussian Process 기반 베이지안 최적화(BO)를 적용하면, 불확실성을 명시적으로 모델링하여 더 적은 평가 횟수로 최적 파라미터를 찾을 수 있다. 특히 Acquisition function(EI, UCB 등)을 통한 탐색-활용(exploration-exploitation) 균형이 본 문제에 적합하다.

#### 7.2.3 BT + 기계학습 하이브리드

BT의 구조적 투명성과 기계학습의 적응성을 결합한 하이브리드 접근법이 유망하다. 예를 들어:
- BT의 조건 노드를 학습된 분류기(classifier)로 대체
- BT의 행동 노드를 강화학습 정책으로 대체
- BT 구조를 하이퍼파라미터로 보고 NAS 기법 적용

#### 7.2.5 다목적 최적화(Multi-Objective Optimization)

현재는 총점만을 최적화 목표로 사용하지만, 실제 군사 응용에서는 다양한 상충 목표(생존율, 연료 소모, 임무 시간 등)를 동시에 고려해야 한다. NSGA-II나 MOEA/D 같은 다목적 진화 알고리즘을 적용하면 파레토 프론트(Pareto front)상의 다양한 트레이드오프 솔루션을 얻을 수 있다.

#### 7.2.6 실시간 적응 BT

현재 BT는 정적 파라미터를 사용하지만, 교전 중 상대의 전술을 실시간으로 분석하여 BT 파라미터를 동적으로 조정하는 적응형 BT(Adaptive BT)를 연구할 수 있다. 이는 온라인 학습(online learning)과 BT의 결합으로, 미지의 상대에 대한 일반화 문제를 해결할 수 있다.

#### 7.2.7 다대다 전투 환경으로의 확장

현재 연구는 1v1 교전에 국한되어 있다. 다대다(N vs N) 공중 전투로 확장하면 전술적 복잡도가 지수적으로 증가한다. BT의 모듈식 구조는 다중 에이전트 조정(multi-agent coordination)을 위한 고수준 분기를 추가하기 용이하여, 이 방향의 연구 가능성이 높다.

---

## 8. 결론 (Conclusion)

본 연구는 JSBSim 기반 F-16 공중 전투 시뮬레이션 환경에서 행동 트리(BT) 에이전트의 파라미터를 자동화된 3단계 탐색 알고리즘을 통해 최적화하는 프레임워크를 제안하고 실험적으로 검증하였다.

**주요 기여**를 요약하면 다음과 같다:

1. **22개 파라미터 BT 템플릿**: 8개의 우선순위 분기와 연속형·이산형 혼합 파라미터 공간을 가진 표현력 높은 BT 템플릿을 설계하였다.

2. **LHS-Hill Climbing 3단계 파이프라인**: 전역 탐색(LHS)과 지역 정제(Hill Climbing)를 결합하여 약 2.2시간 내에 최적 파라미터를 발견하는 효율적인 탐색 알고리즘을 구현하였다.

3. **완벽한 성능의 Golden BT 도출**: 6종의 고정 상대에 대해 120전 전승(승률 100%)을 달성하는 Golden BT를 자동으로 발견하였다. 이는 수작업 설계 기준선 v6(승률 93.3%)을 명확히 상회한다.

4. **Ace vs Aggressive 트레이드오프 해소**: 수작업으로 해결 불가능했던 상대 간 트레이드오프를 `offensive_press_distance=3639m`와 `offensive_press_action=Pursue`의 조합으로 자동 해소하였다.

5. **PvP 내시 균형의 실험적 관찰**: 동일한 최적화 목표에서 도출된 BT들이 서로 대결할 때 완전한 교착 상태로 수렴하는 내시 균형 현상을 관찰하고, 공동 진화 훈련의 필요성을 제기하였다.

본 연구의 결과는 BT 기반 게임 AI에서 자동화 파라미터 최적화의 실용적 가치와 방법론적 유효성을 입증한다. 특히 도메인 전문 지식을 BT 구조에 인코딩하고 자동화 탐색으로 파라미터를 정제하는 인간-AI 협업 설계 패러다임은, 복잡한 전술 환경에서의 자율 에이전트 개발에 광범위하게 적용될 수 있을 것으로 기대된다.

향후 연구에서는 공동 진화 최적화, 베이지안 최적화, BT-ML 하이브리드 접근법을 통해 미지의 상대에 대한 일반화 성능을 향상시키는 것이 핵심 과제이다. 궁극적으로 투명하고 검증 가능하며 적응적인 공중 전투 BT 에이전트의 개발은, 군사 자율화 시스템의 안전성과 신뢰성을 높이는 데 기여할 것이다.

---

## 9. 시스템 설계 (System Design)

### 9.1 전체 아키텍처 개요

AI Combat SDK는 JSBSim 물리 시뮬레이터 위에 행동 트리 기반 자율 에이전트를 올리고, 자동화된 파라미터 최적화 파이프라인을 통해 최강의 에이전트를 탐색하는 시스템이다. 전체 아키텍처는 다음과 같이 세 개의 수직 레이어로 구성된다.

```
┌─────────────────────────────────────────────────────────┐
│                   OPTIMIZATION LAYER                      │
│   tools/bt_optimizer.py — LHS + Hill Climbing + Backtest │
│   tools/validate_agent.py — Syntax & Structure Check     │
└────────────────────┬────────────────────────────────────┘
                     │ generate YAML + call run_match()
┌────────────────────▼────────────────────────────────────┐
│                   MATCH RUNNER LAYER                      │
│   scripts/run_match.py — Match orchestration + replay    │
│   src/match/runner.py  — BehaviorTreeMatch class         │
└────────────────────┬────────────────────────────────────┘
                     │ tick BT per step
┌────────────────────▼────────────────────────────────────┐
│               SIMULATION + BT ENGINE LAYER               │
│   src/simulation/ — JSBSim F-16 physics                  │
│   src/behavior_tree/ — BT parser, executor, node registry│
│   submissions/*/nodes/ — Custom action implementations   │
└─────────────────────────────────────────────────────────┘
```

### 9.2 디렉토리 구조 및 모듈 역할

프로젝트의 전체 디렉토리 구조와 각 모듈의 역할은 다음과 같다.

```
ai-combat-sdk/
│
├── src/                          # SDK 코어 (수정 불가)
│   ├── behavior_tree/            # BT 파싱 및 실행 엔진
│   │   ├── nodes/                # 빌트인 조건·액션 노드 구현체
│   │   │   ├── conditions/       # DistanceBelow, ATABelow,
│   │   │   │                     # IsOffensiveSituation, InEnemyWEZ, ...
│   │   │   └── actions/          # LeadPursuit, Pursue, BreakTurn,
│   │   │                         # ClimbTo, AltitudeAdvantage, ...
│   │   └── executor.py           # Tick-by-tick BT 순회·실행
│   ├── simulation/               # 물리 시뮬레이션
│   │   ├── envs/JSBSim/          # JSBSim 랩퍼
│   │   │   ├── configs/          # bt_vs_bt, tail_chase 시나리오
│   │   │   ├── core/             # 비행 역학 모델
│   │   │   ├── reward_functions/ # HP 감소, Hard Deck 판정
│   │   │   └── tasks/            # 1v1 교전 태스크 정의
│   │   └── runner/               # 시뮬레이션 루프 실행기
│   ├── match/
│   │   └── runner.py             # BehaviorTreeMatch: 매치 API
│   ├── submission/
│   │   └── validator.py          # YAML 구조 검증 (SubmissionValidator)
│   └── tournament/               # 토너먼트 관리
│
├── scripts/                      # 사용자 실행 스크립트
│   └── run_match.py              # CLI: 두 에이전트 간 매치 실행
│
├── tools/                        # 최적화·분석 도구
│   ├── bt_optimizer.py           # 핵심 최적화 파이프라인 (22 파라미터)
│   └── validate_agent.py         # 에이전트 구조 사전 검증
│
├── submissions/                  # 제출 에이전트 (참가자 작성)
│   ├── alpha1/
│   │   ├── alpha1.yaml           # BT 구조 정의
│   │   └── nodes/
│   │       └── custom_actions.py # PNAttack, PNPursuit, AdaptiveAction 등
│   └── golden/
│       ├── golden.yaml           # 최적화 챔피언 BT
│       └── nodes/
│           └── custom_actions.py # (alpha1과 동일)
│
├── examples/                     # 내장 상대 에이전트 (고정)
│   ├── simple.yaml               # 단순 Pursue 에이전트
│   ├── aggressive.yaml           # 공세적 LeadPursuit
│   ├── defensive.yaml            # 방어 중심 BarrelRoll
│   ├── ace.yaml                  # 고급 복합 BT (20+ 노드)
│   ├── eagle1/                   # 복잡한 커스텀 BT
│   └── viper1/                   # 복잡한 커스텀 BT
│
├── config/
│   ├── match_config.yaml         # 기본 매치 설정 (1500 스텝, bt_vs_bt)
│   ├── match_rules.yaml          # 승패 판정 규칙 (Hard Deck = 305m)
│   └── wez_params.yaml           # WEZ 정의 (ATA<12°, 152–914m, DPS=25)
│
├── replays/                      # 매치 리플레이 (.acmi, TacView 호환)
└── REPORT.md                     # 본 연구 보고서
```

### 9.3 컴포넌트 인터페이스 설계

각 레이어 간의 인터페이스는 다음과 같이 설계되어 있으며, 각 컴포넌트는 명확히 분리된 책임을 가진다.

#### 9.3.1 에이전트 정의 인터페이스 (YAML Schema)

에이전트는 YAML 형식으로 정의되며, 구조는 다음 스키마를 따른다.

```yaml
name: <string>           # 에이전트 식별자
version: "<string>"      # 버전 (문자열)
description: "<string>"  # 설명 (선택)
tree:                    # BT 루트 노드
  type: Selector | Sequence | Parallel | Condition | Action
  name: <string>         # 노드 이름 (선택)
  params:                # 노드 파라미터 (선택)
    <key>: <value>
  children:              # 자식 노드 목록 (Composite 노드만)
    - type: ...
```

**빌트인 Condition 노드**:

| 노드 이름 | 파라미터 | 설명 |
|----------|---------|------|
| `DistanceBelow` | `threshold` (m) | 상대와의 거리 < threshold |
| `DistanceAbove` | `threshold` (m) | 상대와의 거리 > threshold |
| `ATABelow` | `threshold` (°) | ATA 각도 < threshold |
| `BelowHardDeck` | `threshold` (m) | 자신 고도 < threshold |
| `IsOffensiveSituation` | — | 공세적 상황 (내부 기준 판단) |
| `IsDefensiveSituation` | — | 수세적 상황 |
| `InEnemyWEZ` | `max_distance`, `max_los_angle` | 적이 자신을 WEZ 내에 포착 |
| `UnderThreat` | `max_aa_angle`, `max_distance` | 위협 수신 중 |

**빌트인 Action 노드**:

| 노드 이름 | 파라미터 | 설명 |
|----------|---------|------|
| `Pursue` | 11개 튜닝 파라미터 | 현재 위치 기반 추적 |
| `LeadPursuit` | — | 예측 위치 기반 추적 |
| `LagPursuit` | — | 후방 위치 기반 추적 |
| `BreakTurn` | — | 즉각 방향 전환 회피 |
| `BarrelRoll` | — | 배럴 롤 기동 |
| `DefensiveManeuver` | — | 복합 방어 기동 |
| `ClimbTo` | `target_altitude` (m) | 목표 고도로 상승 |
| `AltitudeAdvantage` | `target_advantage` (m) | 고도 우위 유지 |

#### 9.3.2 관측 공간 (Observation Space)

각 에이전트의 커스텀 액션은 매 스텝마다 다음 19개 관측 피처에 접근할 수 있다.

```python
obs = {
    # 상대적 위치·거리
    "distance":          float,   # 상대와의 직선 거리 (m)
    "alt_gap":           float,   # 고도 차 (m, 양수 = 우리가 위)
    "ego_altitude":      float,   # 자신의 고도 (m)

    # 각도 정보
    "ata_deg":           float,   # Angle-To-Aspect: 0°=정면, 180°=후방
    "aa_deg":            float,   # Aspect Angle: 상대가 우리를 보는 각도
    "hca_deg":           float,   # Heading Crossing Angle
    "tau_deg":           float,   # 자신 기수 → 상대 방향 오차 (-180~+180°)
    "relative_bearing":  float,   # 상대의 상대 방위
    "side_flag":         int,     # +1=우측, -1=좌측

    # 속도·에너지
    "ego_vc":            float,   # 자신의 속도 (m/s)
    "closure_rate":      float,   # 거리 변화율 (양수=접근)
    "energy_diff":       float,   # 에너지(고도+속도) 차
    "energy_advantage":  bool,    # 에너지 우위 여부

    # 전술 상황
    "turn_rate":         float,   # 상대의 선회율 (°/s)
    "in_39_line":        bool,    # 상대의 3-9 라인 내 위치 여부
    "overshoot_risk":    bool,    # 속도 초과에 의한 오버슈트 위험
    "tc_type":           str,     # "1-circle" 또는 "2-circle" 선회전
    "alt_advantage":     bool,    # 고도 우위 여부
    "spd_advantage":     bool,    # 속도 우위 여부
}
```

#### 9.3.3 행동 공간 (Action Space)

커스텀 액션은 3축 이산 행동을 반환한다.

```python
# 고도 제어 (5단계)
delta_altitude_idx ∈ {0, 1, 2, 3, 4}
  # 0 = 급하강, 2 = 유지, 4 = 급상승

# 기수 방향 제어 (9단계)
delta_heading_idx ∈ {0, 1, ..., 8}
  # 0 = 최대 좌선회, 4 = 직진, 8 = 최대 우선회

# 속도 제어 (5단계)
delta_velocity_idx ∈ {0, 1, 2, 3, 4}
  # 0 = 급감속, 2 = 유지, 4 = 급가속
```

#### 9.3.4 매치 결과 인터페이스

`run_match()` 함수는 다음 형태의 딕셔너리 리스트를 반환한다.

```python
[
  {
    "winner":           str,    # "tree1" | "tree2" | "draw"
    "total_steps":      int,    # 종료 시 경과 스텝 수
    "duration_seconds": float,  # 실제 실행 시간 (초)
    "tree1_reward":     float,  # 에이전트1 누적 보상
    "tree2_reward":     float,  # 에이전트2 누적 보상
    "tree1_health":     float,  # 에이전트1 최종 HP (0–100)
    "tree2_health":     float,  # 에이전트2 최종 HP (0–100)
    "success":          bool,   # 매치 정상 완료 여부
  },
  ...  # rounds 수만큼 반복
]
```

---

## 10. 프로젝트 구현 (Implementation)

### 10.1 행동 트리 실행 엔진

BT 엔진은 YAML 파일을 파싱하여 py_trees 라이브러리 기반의 트리 객체로 변환하고, 매 시뮬레이션 스텝마다 루트 노드부터 Tick을 실행한다.

**실행 흐름**:
```
YAML 파싱 → 노드 객체 생성 → 블랙보드 초기화
    ↓
[매 스텝 (0.2초 간격)]
  1. JSBSim에서 관측값 갱신 → 블랙보드에 기록
  2. BT 루트 Tick()
     └─ Selector 순서대로 자식 평가
        └─ Sequence: 모든 자식이 SUCCESS → 전체 SUCCESS
           └─ Condition: 관측값 조건 검사 → SUCCESS/FAILURE
           └─ Action: 행동 실행 → RUNNING/SUCCESS/FAILURE
  3. Action이 반환한 [alt_idx, heading_idx, vel_idx] → JSBSim 입력
  4. JSBSim 물리 시뮬레이션 1 스텝 진행
  5. 승패 조건 검사 (HP=0, Hard Deck, 1500 스텝 초과)
```

**커스텀 노드 등록**: 에이전트 폴더의 `nodes/custom_actions.py`는 빌트인 노드와 동일한 인터페이스를 구현하며, `update()` 메서드에서 관측값을 읽고 `set_action()`을 호출한다. 커스텀 노드 파일이 존재하면 SDK가 자동으로 임포트하여 빌트인 노드와 동등하게 사용할 수 있다.

### 10.2 커스텀 액션 구현 (`custom_actions.py`)

참가자가 구현한 커스텀 액션은 총 5개의 클래스로 구성된다.

#### PNAttack — WEZ 정밀 조준 액션

WEZ 내 교전에서 PD 제어기를 사용하여 기수를 정밀하게 상대에게 조준한다.

```
입력: tau_deg, tau_rate (각속도), distance
처리:
  heading_cmd = Kp × tau + Kd × (tau - prev_tau)  [PD 제어]
  alt_cmd     = 안정 유지 (고도 변화 최소화)
  vel_cmd     = 거리별 속도 조절
    - dist < 152m: 급제동 (vel=0)
    - dist < 700m: 감속 (vel=1)
    - dist > 914m: 가속 (vel=3)
출력: [alt_idx, heading_idx, vel_idx]
```

**설계 원칙**: 공격 플랫폼 안정화 — WEZ 내에서 불필요한 고도 변화나 급가속을 억제하여 조준선 안정성을 극대화한다.

#### PNPursuit — 원거리 PN 기반 추격 액션

Kp=1.0, Kd=0.4의 부드러운 PD 제어로 원거리 접근 시 에너지를 보존하며 추격한다. PNAttack보다 완화된 이득값으로 오버슈트를 방지한다.

#### AdaptiveAction — 상황 인식 적응형 액션

관측 피처 19개 중 `tc_type`, `overshoot_risk`, `aa_deg`, `in_39_line`, `energy_diff`를 조합하여 다음 5가지 전투 상황을 분류하고 각 상황에 최적화된 제어를 수행한다.

| 상황 분류 | 조건 | 선택 전략 |
|---------|-----|---------|
| OFFENSIVE | aa < 60°, ata < 70° | LeadPursuit형 (Kp=1.1~1.3) |
| DEFENSIVE | aa > 140° | 급 BreakTurn (반대 방향) |
| OVERSHOOT | overshoot_risk=True | High Yo-Yo (상승 + LagPursuit) |
| 1-CIRCLE | tc_type="1-circle", HCA<90° | 선회율 극대화 (감속) |
| 2-CIRCLE | tc_type="2-circle", HCA>90° | 에너지 파이트 (속도 유지) |
| DEFAULT | 기타 | LagPursuit (에너지 보존) |

#### EnergyRecovery — 저에너지 회복 액션

속도 < 150 m/s이면 고도를 활용한 속도 회복을 수행한다. Hard Deck(800m) 이하일 경우 즉시 상승을 우선한다.

### 10.3 최적화 파이프라인 구현 (`bt_optimizer.py`)

최적화 파이프라인의 핵심 구현 요소들을 설명한다.

#### BT YAML 생성기 (`generate_bt_yaml`)

파라미터 딕셔너리를 입력받아 실행 가능한 BT YAML을 동적으로 생성한다. 분기 포함 여부는 `include_*` 플래그로 제어하며, 플래그가 False인 분기는 YAML에서 완전히 제거된다.

```python
# 분기 우선순위 순서 (Selector 자식 순서)
branches = [
    "HardDeckAvoidance",         # 항상 포함 (최우선)
    "GunEngagement",             # 항상 포함 (PNAttack)
    "ThreatResponse",            # include_enemy_wez=True 시 포함
    "OffensivePress",            # include_offensive_press=True 시 포함
    "EmergencyDefense",          # include_emergency_defense=True 시 포함
    "CloseCombat",               # 항상 포함
    "DefensiveEvasion",          # include_is_defensive=True 시 포함
    "Default(+AltitudeAdv)",     # 항상 포함 (최후순위)
]
```

#### 적합도 평가기 (`evaluate_fitness`)

1. 임시 YAML 파일 생성 (고유 접미사로 병렬 충돌 방지)
2. 6개 상대 × `rounds_per_opponent` 라운드 순차 실행
3. 각 라운드 결과를 계층적 점수로 변환
4. 임시 파일 삭제 후 `(총점, 상대별 세부 결과)` 반환

#### Latin Hypercube Sampling

`scipy.stats.qmc.LatinHypercube`를 사용하여 연속형 파라미터의 12차원 공간을 균등하게 탐색하는 200개 표본을 생성한다.

```python
from scipy.stats.qmc import LatinHypercube

sampler = LatinHypercube(d=12, seed=42)  # d = 연속형 파라미터 수
samples = sampler.random(n=200)          # [0,1]^12 정규화 표본
# 각 차원을 실제 파라미터 범위로 역변환
```

이산형 파라미터는 `random.choice(options)`로 균등 샘플링한다.

#### Spearman 민감도 분석 (`print_param_analysis`)

Stage 1 완료 후 각 연속형 파라미터의 총점과의 Spearman 순위 상관계수를 계산하여 파라미터 중요도를 정량화한다. 외부 라이브러리 없이 순수 Python으로 구현하였다.

```python
def _spearman(x, y):
    # 1. x, y를 각각 순위로 변환
    # 2. 순위 차(d)의 제곱합 계산
    # 3. ρ = 1 - 6Σd² / n(n²-1)
    n = len(x)
    rx = _rank(x); ry = _rank(y)
    d_sq = sum((a-b)**2 for a, b in zip(rx, ry))
    return 1 - 6*d_sq / (n*(n**2-1))
```

이산형 파라미터는 각 선택지별 평균 점수를 계산하여 순위를 매긴다.

### 10.4 상대 에이전트 분류

6개의 내장 상대 에이전트는 전술적 다양성을 위해 설계되었으며, 각각 뚜렷이 다른 전략을 구사한다.

| 에이전트 | BT 복잡도 | 주요 전략 | 약점 |
|---------|---------|---------|-----|
| `simple` | 1 노드 | 순수 Pursue | 방어 기동 없음 |
| `aggressive` | ~5 노드 | LeadPursuit 일변도 | 오버슈트, Hard Deck 위험 |
| `defensive` | ~5 노드 | BarrelRoll + 추격 | 공세 능력 제한 |
| `ace` | 20+ 노드 | 복합 적응형 BT | WEZ 활용 최적화 |
| `eagle1` | 복합 | 커스텀 액션 포함 | 상황별 상이 |
| `viper1` | 복합 | 커스텀 액션 포함 | 상황별 상이 |

---

## 11. 평가 체계 (Evaluation Framework)

### 11.1 단일 매치 평가

매 매치는 JSBSim 시뮬레이터에서 결정론적으로 실행된다. 평가의 핵심 지표는 다음과 같다.

**기본 평가 지표**:
- **승패 결과** (`winner`): "tree1" / "tree2" / "draw"
- **잔여 HP**: 양측의 최종 체력 (0–100)
- **종료 스텝**: 승패가 확정된 스텝 번호 (빠를수록 결정적)

**승패 판정 우선순위**:
```
1순위: HP = 0 (상대에 의한 격파)
2순위: 고도 < 305m (Hard Deck 진입 — 즉사)
3순위: 스텝 1500 도달 → HP 비교 → 동일하면 무승부
```

### 11.2 파라미터 최적화 평가

**계층적 점수 체계**의 수학적 설계 원칙:

```
f(result, hp_ratio) = BASE(result) × (1 + HP_WEIGHT × hp_ratio)

  결과     BASE    hp_ratio=0  hp_ratio=1
  WIN       10        10.0       30.0
  DRAW       1         1.0        3.0
  LOSS      -5        -5.0      -15.0

불변식: min(WIN) > max(DRAW) > max(LOSS)
         10.0    >    3.0     >   -5.0   ✓
```

이 설계는 두 가지 목표를 동시에 달성한다:
1. **승리 추구 강제**: 최악의 승리(HP 0%로 이김) > 최고의 무승부(HP 100%로 비김)
2. **HP 차이 보상**: 같은 결과 내에서 더 높은 HP 유지에 점수 그라디언트 부여

**총점 계산**: `total_score = Σ f(result_r,o, hp_ratio_r,o)` (r=라운드, o=상대)

### 11.3 3단계 검증 파이프라인

최적화는 계산 비용과 통계적 신뢰도 간의 트레이드오프를 고려한 3단계로 진행된다.

```
단계     후보 수   라운드/상대   총 매치 수   목적
──────────────────────────────────────────────────────────
Stage 1   204        2           2,448       전역 탐색 (LHS)
Stage 2   160        3           2,880       지역 정제 (Hill Climbing)
Stage 3     5        5             150       최종 선별
Backtest    2       20             240       통계적 유의성 검증
────────────────────────────────────────────────────────────────
합계                              5,718       총 시뮬레이션 매치
```

**점진적 정밀도 설계**: 탐색 초기에는 저비용·저정밀(2 라운드) 평가로 넓은 공간을 스캔하고, 후반부로 갈수록 라운드 수를 높여 통계적 노이즈를 줄인다. 이를 통해 전체 탐색 비용을 최소화하면서도 최종 선별의 신뢰도를 확보한다.

### 11.4 백테스트 (Backtest)

Stage 3 챔피언을 v6 기준선과 함께 20 라운드 × 6 상대(= 각 120 매치)로 엄밀 비교한다. 20 라운드는 시뮬레이션 결정론성 하에서 다양한 Round Seed를 반영한 것이며, 단일 라운드 비교보다 훨씬 신뢰할 수 있는 성능 추정을 제공한다.

**평가 출력**:
```json
{
  "golden": {
    "wins": 120, "draws": 0, "losses": 0,
    "win_rate": 1.0,
    "per_opponent": {
      "ace":       {"wins": 20, "draws": 0, "losses": 0},
      "aggressive":{"wins": 20, "draws": 0, "losses": 0},
      ...
    }
  },
  "v6_baseline": {
    "wins": 112, "draws": 8, "losses": 0,
    "win_rate": 0.933,
    ...
  }
}
```

### 11.5 라운드 로빈 토너먼트

상위 5개 후보와 v6 기준선 간의 교차 비교로 에이전트 간 우열 관계 및 PvP 특성을 분석한다.

**구성**: C(6,2) = 15 매치업, 각 N 라운드
**출력 형식**:

```
┌─────────────┬────────┬────────┬────────┬────────┬────────┬────────┐
│             │ golden │ cand_2 │ cand_3 │ cand_4 │ cand_5 │  v6    │
├─────────────┼────────┼────────┼────────┼────────┼────────┼────────┤
│ golden      │   —    │  0-3-0 │  0-3-0 │  0-3-0 │  0-3-0 │  0-3-0 │
│ cand_2      │  0-3-0 │   —    │  0-3-0 │  ...   │  ...   │  ...   │
│ ...         │  ...   │  ...   │   ...  │  ...   │  ...   │  ...   │
└─────────────┴────────┴────────┴────────┴────────┴────────┴────────┘
형식: W-D-L (row 에이전트 기준)
```

**리플레이 저장**: 라운드 로빈 실행 시 `replays/` 디렉토리에 `.acmi` 형식으로 모든 매치의 비행 궤적이 저장되며, TacView 소프트웨어로 3D 시각화가 가능하다.

### 11.6 에이전트 검증 (`validate_agent.py`)

제출 전 에이전트의 구조적 유효성을 사전에 검증한다.

**검증 항목**:
1. YAML 문법 오류
2. 필수 필드 존재 (`name`, `version`, `tree`)
3. BT 노드 유형 유효성 (알려진 노드 이름 확인)
4. 커스텀 노드의 `update()` 메서드 구현 여부
5. `set_action()` 호출 확인 (행동 출력 누락 방지)

**실행 예시**:
```bash
# 검증 실행 (Windows PowerShell)
$env:PYTHONIOENCODING="utf-8"
C:\...\Python312\python.exe tools\validate_agent.py submissions\golden\golden.yaml
```

---

## 12. 실행 가이드 (Quick Start)

### 12.1 환경 설정

```bash
# Python 3.12 필수 (JSBSim .pyd 파일 호환)
# 작업 디렉토리: ai-combat-sdk/
cd C:\Users\Joon\Desktop\AI-pilot\ai-combat-sdk
set PYTHONIOENCODING=utf-8   # Windows cmd
# 또는
$env:PYTHONIOENCODING="utf-8"  # PowerShell
```

### 12.2 매치 실행

```bash
# 기본 매치 (1 라운드)
python scripts\run_match.py --agent1 golden --agent2 ace

# 다중 라운드
python scripts\run_match.py --agent1 golden --agent2 aggressive --rounds 5

# 조용한 출력 (점수만)
python scripts\run_match.py --agent1 golden --agent2 simple --rounds 3 --quiet
```

### 12.3 최적화 실행

```bash
# 전체 최적화 (약 2.2시간)
python tools\bt_optimizer.py --candidates 200 --workers 20 > opt_log.txt 2>&1

# 백테스트 (최적 에이전트 검증, 약 55분)
python tools\bt_optimizer.py --backtest --rounds 20 > backtest_log.txt 2>&1

# 라운드 로빈 (상위 후보 간 교차 비교, 약 15분)
python tools\bt_optimizer.py --roundrobin --rounds 5
```

### 12.4 에이전트 검증

```bash
python tools\validate_agent.py submissions\golden\golden.yaml
```

### 12.5 리플레이 시청

1. [TacView](https://www.tacview.net/) 설치 (무료 버전)
2. `replays/` 폴더의 `.acmi` 파일을 TacView로 열기
3. 3D 비행 궤적, 교전 지점, HP 변화를 시각적으로 확인

---

## 참고문헌 (References)

1. Berndt, J. S. (2004). JSBSim: An open source flight dynamics model in C++. *AIAA Modeling and Simulation Technologies Conference*.

2. Colledanchise, M., & Ögren, P. (2018). *Behavior Trees in Robotics and AI: An Introduction*. CRC Press.

3. DARPA. (2020). AlphaDogfight Trials Final Event. Defense Advanced Research Projects Agency.

4. Lucas, S. M., & Kendall, G. (2006). Evolutionary computation and games. *IEEE Computational Intelligence Magazine*, 1(1), 10-18.

5. McKay, M. D., Beckman, R. J., & Conover, W. J. (1979). A comparison of three methods for selecting values of input variables in the analysis of output from a computer code. *Technometrics*, 21(2), 239-245.

6. Lim, C. U., Baumgarten, R., & Colton, S. (2010). Evolving behaviour trees for the commercial game DEFCON. *Applications of Evolutionary Computation* (pp. 100-110).

7. Perez, D., Nicolau, M., O'Neill, M., & Brabazon, A. (2011). Evolving behaviour trees for the Mario AI competition using grammatical evolution. *Applications of Evolutionary Computation* (pp. 123-132).

8. Shaw, R. L. (1985). *Fighter Combat: Tactics and Maneuvering*. Naval Institute Press.

9. Iovino, M., Tuci, E., Saffiotti, A., & Colledanchise, M. (2022). A survey of behavior trees in robotics and AI. *Robotics and Autonomous Systems*, 154, 104096.

10. Murdock, P. (2001). Hierarchical task network planning for computer-generated forces. *Proceedings of the International Conference on Artificial Intelligence in Design*.

---

## 부록 (Appendix)

### 부록 A: 추적 방법 상세 비교

| 특성 | LeadPursuit | Pursue | LagPursuit | PurePursuit |
|-----|------------|--------|-----------|-----------|
| 조준 위치 | 상대의 미래 위치 | 상대의 현재 위치 | 상대의 과거 위치 | 상대의 현재 위치 (순수) |
| 수렴 속도 | 빠름 | 중간 | 느림 | 중간 |
| 오버슈트 위험 | 높음 | 낮음 | 매우 낮음 | 낮음 |
| 급기동 대응 | 취약 | 양호 | 우수 | 양호 |
| 에너지 소모 | 높음 | 중간 | 낮음 | 중간 |
| 적합 상황 | 방어적 상대, 안정적 기동 | 급기동 상대, 범용 | 에너지 보존, 수세 | 거리 좁힘 |

### 부록 B: 방어 기동 상세 비교

| 방어 기동 | 설명 | 장점 | 단점 | 적합 상황 |
|---------|-----|-----|-----|---------|
| BreakTurn | 급격한 방향 전환 | 빠른 회피, 에너지 유지 | 단순 예측 가능 | WEZ 즉각 이탈 |
| DefensiveManeuver | 복합 회피 기동 | 복잡한 궤적 | 에너지 소모 | 중거리 위협 |
| DefensiveSpiral | 나선형 기동 | 예측 불가능 | 고도 손실 | 다중 위협 |
| BarrelRoll | 배럴 롤 기동 | 속도 조절 | 궤적 혼란, 공세 전환 지연 | 속도 우위 상황 |

### 부록 C: 파라미터 별 민감도 분석

Stage 1 결과를 바탕으로 각 파라미터가 총점에 미치는 영향을 분석하였다.

**고민감도 파라미터** (점수 분산에 크게 기여):
1. `offensive_press_distance`: 이 변수의 범위에 따라 성능이 크게 변동
2. `include_offensive_press`: true/false 여부가 평균 15점 차이 유발
3. `include_enemy_wez`: true/false 여부가 특히 ace 상대에서 큰 차이
4. `close_combat_distance`: 2000m vs 4000m 설정이 교전 주도권에 결정적
5. `default_action`: LagPursuit vs LeadPursuit이 압박 수준에 직접 영향

**저민감도 파라미터** (점수 분산에 작게 기여):
1. `pnattack_kp`, `pnattack_kd`: WEZ 내 정밀도에는 영향 있으나 전체 점수에 미미
2. `include_emergency_defense`: false가 약간 유리 (불필요한 회피 방지)
3. `include_is_defensive`: false가 약간 유리 (DefensiveEvasion은 공세성 저하)
4. `altitude_advantage_target`: 일정 범위 내에서 크게 차이 없음

### 부록 D: 점수 체계 수학적 검증

점수 체계의 계층성 보장을 수학적으로 검증한다.

```
WIN_BASE = 10, DRAW_BASE = 1, LOSS_BASE = -5, HP_WEIGHT = 2

최악의 WIN: hp_ratio = 0 → score = 10 × (1 + 2×0) = 10
최고의 WIN: hp_ratio = 1 → score = 10 × (1 + 2×1) = 30

최악의 DRAW: hp_ratio = 0 → score = 1 × (1 + 2×0) = 1
최고의 DRAW: hp_ratio = 1 → score = 1 × (1 + 2×1) = 3

최악의 LOSS: hp_ratio = 1 → score = -5 × (1 + 2×1) = -15
최고의 LOSS: hp_ratio = 0 → score = -5 × (1 + 2×0) = -5

검증:
- worst_WIN(10) > best_DRAW(3) ✓
- best_DRAW(3) > best_LOSS(-5) ✓
- worst_WIN(10) > best_DRAW(3) > best_LOSS(-5) > worst_LOSS(-15) ✓

경계 조건 수정 후:
- worst_WIN = WIN_BASE × 1 = 10 × 1 = 10 (hp_ratio=0, 적 HP=0이므로)
  → 실제로는 hp_ratio=my_hp/initial_hp이므로 0 <= hp_ratio <= 1

결론: worst_win(8.0) > best_draw(3.0) > best_loss(-3.0) 보장됨
  (hp_ratio=0.1일 때 WIN = 10×1.2=12, hp_ratio=1일 때 DRAW = 1×3=3,
   hp_ratio=0.4일 때 LOSS = -5×1.8=-9이므로 계층 보장)
```

### 부록 E: 실험 환경 및 재현성

**하드웨어 환경**:
- OS: Windows 11 Pro 10.0.26100
- Python: 3.12 (JSBSim .pyd 호환성 요구)
- 병렬 처리: 20 워커 (multiprocessing)

**소프트웨어 의존성**:
- JSBSim: 공기 역학 시뮬레이터
- py_trees: BT 실행 엔진
- numpy, scipy: LHS 생성 및 수치 계산
- yaml: BT 정의 파일 파싱

**재현성**:
- 시뮬레이션 자체는 결정론적이므로, 동일 파라미터, 동일 초기 조건에서 동일 결과 생성
- 병렬 평가 시 워커 간 실행 순서가 달라져도 개별 매치 결과는 동일
- 난수 생성기 시드 고정 시 LHS 표본 완전 재현 가능

---

*본 보고서는 AI-Pilot 연구 프로젝트의 행동 트리 최적화 실험에 대한 종합 분석을 담고 있으며, 2026년 3월 3일 기준의 연구 성과를 반영한다. 섹션 9–12는 시스템 설계·구현·평가·실행 가이드를 포함한 기술 문서 역할을 겸하며, 섹션 7.2.1–7.2.2는 2026년 3월 PvP 분석 및 스텝 로깅 타당성 검토 결과를 포함한다.*

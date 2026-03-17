# 🎯 Golden State 학습 가이드

## 핵심 아이디어

SDK의 **상세 로깅 데이터**를 활용하여 **Golden BT의 성공 패턴**을 학습하고, 이를 자신의 BT 개발에 적용하는 방법론입니다.

### 왜 SDK 로깅이 적합한가?

#### ✅ SDK CSV 로그의 강점

**49개 컬럼의 풍부한 데이터**:
```
step, agent_id, tree_name,
ego_altitude_ft, ego_vc_kts, ego_vx_kts, ego_vy_kts, ego_vz_kts,
roll_deg, pitch_deg, specific_energy_ft, ps_fts,
distance_ft, ata_deg, aa_deg, hca_deg, tau_deg, relative_bearing_deg,
alt_gap_ft, closure_rate_kts, turn_rate_degs,
in_39_line, overshoot_risk, tc_type,
ata_lead_deg, tau_lead_deg, side_flag,
energy_advantage, energy_diff_ft, alt_advantage, spd_advantage,
bfm_situation,
ego_health, enm_health, ego_damage_dealt, enm_damage_dealt,
in_wez, enm_in_wez, reward,
action_altitude, action_heading, action_velocity,
aileron, elevator, rudder, throttle,
active_node, active_nodes_path  ← 핵심!
```

**기존 콜백 로그 대비 장점**:
- ✅ **BT 실행 경로**: `active_nodes_path` (전체 노드 경로)
- ✅ **비행역학 상세**: 속도 벡터, 롤/피치, 에너지 상태
- ✅ **전술 기하**: ATA, AA, HCA, Tau, 3-9 Line, Overshoot Risk
- ✅ **에너지 관리**: `specific_energy_ft`, `ps_fts`, `energy_diff_ft`
- ✅ **미래 예측**: `ata_lead_deg`, `tau_lead_deg` (1초 후 예측값)

---

## 학습 프레임워크: (time, node, action) → golden_state

### 개념

**Golden BT의 로그**를 분석하여 다음 패턴을 학습:

```
시간(step) + 활성 노드(active_node) + 액션(action) 
    ↓
Golden State (성공적인 전투 상태)
```

### Golden State의 정의

**성공적인 전투 상태**의 특징:
1. **WEZ 진입 성공**: `in_wez=True` + `ego_damage_dealt` 증가
2. **에너지 우위 유지**: `energy_advantage=True` + `specific_energy_ft` 높음
3. **BFM 우위**: `bfm_situation=OBFM` (공격 유리)
4. **안전한 위치**: `enm_in_wez=False` + `altitude > hard_deck`
5. **효율적 기동**: `ps_fts > 0` (에너지 증가 중)

---

## 구체적인 학습 방법

### 1. Golden BT 로그 수집

```bash
# Golden BT vs 모든 상대 (상세 로그)
python tools/test_agent.py golden --all-opponents --rounds 10 --log-csv golden_logs
```

**결과**: `golden_logs/` 폴더에 60개 CSV 파일 (6 상대 × 10 라운드)

### 2. 성공 패턴 추출

#### Python 분석 스크립트

```python
import pandas as pd
import glob
from pathlib import Path

# 모든 Golden BT 로그 로드
golden_files = glob.glob("golden_logs/*_golden_vs_*.csv")
all_data = []

for file in golden_files:
    df = pd.read_csv(file)
    # Golden BT만 필터링
    golden_data = df[df['tree_name'] == 'golden'].copy()
    all_data.append(golden_data)

golden_df = pd.concat(all_data, ignore_index=True)

# ========================================
# Golden State 정의: WEZ 진입 성공 순간
# ========================================

golden_states = golden_df[
    (golden_df['in_wez'] == True) &
    (golden_df['ego_damage_dealt'] > 0) &
    (golden_df['enm_in_wez'] == False)
].copy()

print(f"총 Golden State 발견: {len(golden_states)} / {len(golden_df)} ({len(golden_states)/len(golden_df)*100:.1f}%)")

# ========================================
# 패턴 분석 1: 어떤 노드가 Golden State를 만드는가?
# ========================================

node_success = golden_states['active_node'].value_counts()
print("\n=== Golden State를 만드는 노드 Top 10 ===")
print(node_success.head(10))

# ========================================
# 패턴 분석 2: Golden State의 전술 기하 특성
# ========================================

print("\n=== Golden State의 평균 전술 파라미터 ===")
print(f"평균 거리: {golden_states['distance_ft'].mean():.0f} ft")
print(f"평균 ATA: {golden_states['ata_deg'].mean():.1f}°")
print(f"평균 AA: {golden_states['aa_deg'].mean():.1f}°")
print(f"평균 고도 차이: {golden_states['alt_gap_ft'].mean():.0f} ft")
print(f"평균 에너지 차이: {golden_states['energy_diff_ft'].mean():.0f} ft")

# ========================================
# 패턴 분석 3: Golden State 진입 전 노드 시퀀스
# ========================================

# Golden State 5스텝 전 노드 경로 분석
golden_states['prev_5_steps'] = None
for idx in golden_states.index:
    if idx >= 5:
        prev_nodes = golden_df.loc[idx-5:idx-1, 'active_node'].tolist()
        golden_states.at[idx, 'prev_5_steps'] = ' -> '.join(prev_nodes)

print("\n=== Golden State 진입 전 노드 시퀀스 Top 10 ===")
print(golden_states['prev_5_steps'].value_counts().head(10))

# ========================================
# 패턴 분석 4: 시간대별 Golden State 분포
# ========================================

import matplotlib.pyplot as plt

plt.figure(figsize=(12, 6))
plt.hist(golden_states['step'], bins=50, alpha=0.7, edgecolor='black')
plt.xlabel('Step (시간)')
plt.ylabel('Golden State 발생 빈도')
plt.title('Golden BT의 WEZ 진입 성공 시점 분포')
plt.grid(True, alpha=0.3)
plt.savefig('golden_state_timeline.png', dpi=150, bbox_inches='tight')
print("\n그래프 저장: golden_state_timeline.png")

# ========================================
# 패턴 분석 5: 상대별 Golden State 전략 차이
# ========================================

# 파일명에서 상대 추출
golden_states['opponent'] = golden_states.index.map(
    lambda idx: golden_files[idx // len(golden_df) * len(golden_files) // len(all_data)].split('_vs_')[1].split('.csv')[0]
)

print("\n=== 상대별 주요 활성 노드 ===")
for opponent in ['ace', 'aggressive', 'defensive', 'simple', 'eagle1', 'viper1']:
    opp_states = golden_states[golden_states['opponent'] == opponent]
    if len(opp_states) > 0:
        top_node = opp_states['active_node'].value_counts().head(1)
        print(f"{opponent:12s}: {top_node.index[0]:20s} ({top_node.values[0]}회)")
```

### 3. 학습된 패턴을 BT에 적용

#### 발견된 패턴 예시

**가정: 분석 결과**
```
Golden State를 만드는 노드 Top 3:
1. PNAttack (342회) - 거리 700ft, ATA 2.1°
2. LeadPursuit (189회) - 거리 1200ft, ATA 8.5°
3. Pursue (87회) - 거리 2500ft, ATA 15.2°

Golden State 진입 전 시퀀스 Top 3:
1. LeadPursuit -> LeadPursuit -> PNAttack (78회)
2. Pursue -> LeadPursuit -> PNAttack (45회)
3. BreakTurn -> LeadPursuit -> LeadPursuit -> PNAttack (32회)
```

#### BT 설계에 반영

```yaml
tree:
  type: Selector
  children:
    # 1. Hard Deck 회피 (필수)
    - type: Sequence
      children:
        - type: Condition
          name: BelowHardDeck
        - type: Action
          name: ClimbTo
          params:
            target_altitude_ft: 2000

    # 2. WEZ 정밀 사격 (Golden State 패턴 1)
    - type: Sequence
      children:
        - type: Condition
          name: DistanceBelow
          params:
            threshold_ft: 914  # WEZ 최대 거리
        - type: Condition
          name: DistanceAbove
          params:
            threshold_ft: 500  # WEZ 최소 거리
        - type: Condition
          name: ATABelow
          params:
            threshold_deg: 4  # Golden State 평균 2.1° + 여유
        - type: Action
          name: PNAttack  # ← Golden State 최빈 노드!

    # 3. 근접 추적 (Golden State 패턴 2)
    - type: Sequence
      children:
        - type: Condition
          name: DistanceBelow
          params:
            threshold_ft: 1500  # Golden State 평균 1200ft + 여유
        - type: Action
          name: LeadPursuit  # ← Golden State 2위 노드

    # 4. 기본 추적 (Golden State 패턴 3)
    - type: Action
      name: Pursue
```

---

## 고급 분석: 시계열 패턴 학습

### 노드 전환 확률 학습

```python
# Golden BT의 노드 전환 확률 계산
from collections import defaultdict

transitions = defaultdict(lambda: defaultdict(int))

for i in range(len(golden_df) - 1):
    current_node = golden_df.loc[i, 'active_node']
    next_node = golden_df.loc[i+1, 'active_node']
    transitions[current_node][next_node] += 1

# 확률로 변환
transition_probs = {}
for current, nexts in transitions.items():
    total = sum(nexts.values())
    transition_probs[current] = {
        next_node: count / total 
        for next_node, count in nexts.items()
    }

# 예: LeadPursuit 다음에 어떤 노드가 오는가?
print("\n=== LeadPursuit 다음 노드 확률 ===")
for next_node, prob in sorted(
    transition_probs.get('LeadPursuit', {}).items(), 
    key=lambda x: x[1], 
    reverse=True
)[:5]:
    print(f"{next_node:20s}: {prob*100:.1f}%")
```

### 상태-액션 매핑 학습

```python
# 특정 전술 상황에서 Golden BT가 선택한 액션
def analyze_state_action(df, distance_range, ata_range, bfm_situation=None):
    """
    특정 상태에서 Golden BT의 액션 분포
    """
    mask = (
        (df['distance_ft'] >= distance_range[0]) &
        (df['distance_ft'] <= distance_range[1]) &
        (df['ata_deg'] >= ata_range[0]) &
        (df['ata_deg'] <= ata_range[1])
    )
    
    if bfm_situation:
        mask &= (df['bfm_situation'] == bfm_situation)
    
    state_data = df[mask]
    
    if len(state_data) == 0:
        return None
    
    action_dist = state_data['active_node'].value_counts()
    
    return {
        'count': len(state_data),
        'actions': action_dist.to_dict(),
        'avg_damage': state_data['ego_damage_dealt'].mean(),
        'wez_rate': (state_data['in_wez'] == True).sum() / len(state_data)
    }

# 예: 근접전 (500-1000ft, ATA < 10°, OBFM)에서 Golden BT의 선택
result = analyze_state_action(
    golden_df,
    distance_range=(500, 1000),
    ata_range=(0, 10),
    bfm_situation='BFMSituation.OBFM'
)

print("\n=== 근접 공세 상황 (500-1000ft, ATA<10°, OBFM) ===")
print(f"발생 횟수: {result['count']}")
print(f"평균 데미지: {result['avg_damage']:.1f}")
print(f"WEZ 진입률: {result['wez_rate']*100:.1f}%")
print("\n주요 액션:")
for action, count in sorted(result['actions'].items(), key=lambda x: x[1], reverse=True)[:5]:
    print(f"  {action:20s}: {count:4d}회 ({count/result['count']*100:.1f}%)")
```

---

## 실전 활용 워크플로우

### Phase 1: Golden BT 데이터 수집

```bash
# 1. Golden BT vs 모든 상대 (충분한 샘플)
python tools/test_agent.py golden --all-opponents --rounds 20 --log-csv golden_analysis

# 2. 특정 상대 집중 분석 (예: ace)
python scripts/run_match.py --agent1 golden --agent2 ace --rounds 50 --log-csv golden_vs_ace
```

### Phase 2: 패턴 분석 및 인사이트 도출

```bash
# Python 분석 스크립트 실행
python analyze_golden_patterns.py
```

**출력 예시**:
```
=== Golden State 분석 결과 ===

총 Golden State: 1,847 / 18,000 (10.3%)

주요 노드:
  PNAttack      : 34.2%
  LeadPursuit   : 28.5%
  Pursue        : 15.7%

평균 전술 파라미터:
  거리: 782 ft
  ATA: 3.8°
  고도 차이: +245 ft (우위)
  에너지 차이: +1,234 ft (우위)

핵심 발견:
  1. PNAttack은 거리 600-900ft, ATA < 5°에서 최고 효율
  2. LeadPursuit → PNAttack 전환이 가장 성공적
  3. 에너지 우위 유지가 WEZ 진입률 2배 향상
```

### Phase 3: 자신의 BT에 적용

```yaml
# 학습된 패턴을 반영한 BT
tree:
  type: Selector
  children:
    # Golden Pattern 1: 정밀 사격 (PNAttack at 600-900ft, ATA<5°)
    - type: Sequence
      children:
        - type: Condition
          name: DistanceBelow
          params:
            threshold_ft: 900
        - type: Condition
          name: DistanceAbove
          params:
            threshold_ft: 600
        - type: Condition
          name: ATABelow
          params:
            threshold_deg: 5
        - type: Action
          name: PNAttack

    # Golden Pattern 2: 근접 추적 (LeadPursuit at 1000-1500ft)
    - type: Sequence
      children:
        - type: Condition
          name: DistanceBelow
          params:
            threshold_ft: 1500
        - type: Action
          name: LeadPursuit
    
    # ... 나머지 분기
```

### Phase 4: 검증 및 반복

```bash
# 1. 새 BT 테스트
python tools/test_agent.py my_learned_bt --all-opponents --rounds 10 --log-csv my_bt_test

# 2. Golden BT와 비교
python compare_bt_patterns.py golden_analysis/ my_bt_test/

# 3. 개선점 발견 → BT 수정 → 재테스트
```

---

## 머신러닝 확장 (선택사항)

### 지도 학습: Golden State 예측 모델

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split

# 특징 추출
features = [
    'distance_ft', 'ata_deg', 'aa_deg', 'alt_gap_ft', 
    'energy_diff_ft', 'closure_rate_kts', 'turn_rate_degs'
]

X = golden_df[features].fillna(0)
y = (golden_df['in_wez'] == True) & (golden_df['ego_damage_dealt'] > 0)

# 학습
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
model = RandomForestClassifier(n_estimators=100, max_depth=10)
model.fit(X_train, y_train)

# 평가
accuracy = model.score(X_test, y_test)
print(f"Golden State 예측 정확도: {accuracy*100:.1f}%")

# 중요 특징
importances = pd.DataFrame({
    'feature': features,
    'importance': model.feature_importances_
}).sort_values('importance', ascending=False)

print("\n=== Golden State 예측에 중요한 특징 ===")
print(importances)
```

---

## 핵심 장점 요약

### 1. **BT 구조 친화적**
- `active_nodes_path`로 정확한 노드 실행 경로 추적
- 어떤 분기가 성공으로 이어지는지 명확히 파악

### 2. **세부 전술 데이터**
- 49개 컬럼의 풍부한 전투 상태 정보
- 비행역학, 에너지, 전술 기하 모두 포함

### 3. **시계열 패턴 학습**
- 노드 전환 시퀀스 분석
- 성공적인 전투 흐름 발견

### 4. **상대별 맞춤 전략**
- 각 상대에 대한 Golden BT의 다른 접근법 학습
- 적응적 BT 설계 가능

### 5. **정량적 검증**
- 학습된 패턴의 실제 효과 측정
- 데이터 기반 의사결정

---

## 다음 단계

1. **Golden BT 로그 수집** (20+ 라운드)
2. **패턴 분석 스크립트 실행**
3. **핵심 인사이트 도출**
4. **자신의 BT에 반영**
5. **검증 및 반복 개선**

이 방법론은 **수작업 시행착오**를 **데이터 기반 학습**으로 전환하여, Golden BT의 성공 비결을 체계적으로 습득할 수 있게 합니다! 🎯

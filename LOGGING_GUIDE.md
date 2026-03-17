# 🔍 로깅 시스템 완벽 가이드

## 📊 두 가지 로깅 방식

AI Combat SDK는 **두 가지 독립적인 로깅 시스템**을 제공합니다:

### 1️⃣ SDK 내장 CSV 로깅 (`--log-csv`)
- **목적**: 전체 시뮬레이션 데이터 상세 기록
- **컬럼 수**: 49개 (모든 관측값 + 비행역학 데이터)
- **용도**: 심층 분석, 디버깅, 연구

### 2️⃣ 콜백 기반 커스텀 로깅 (`--callback-log`)
- **목적**: 핵심 정보만 간결하게 기록 + 실시간 콘솔 출력
- **컬럼 수**: 21개 (전술 분석에 필요한 핵심만)
- **용도**: 빠른 리뷰, 실시간 모니터링, 커스텀 메트릭

---

## 🚀 명령어 사용법

### 기본 매치 (로깅 없음)
```bash
python scripts/run_match.py --agent1 eagle1 --agent2 simple
```

### CSV 로깅만 (전체 데이터)
```bash
python scripts/run_match.py --agent1 eagle1 --agent2 simple --log-csv
```
- 저장 위치: `logs/YYYYMMDD_HHMMSS_agent1_vs_agent2.csv`
- 파일 크기: 큼 (모든 데이터 포함)

### 콜백 로깅만 (실시간 출력 + 요약 파일)
```bash
python scripts/run_match.py --agent1 eagle1 --agent2 simple --callback-log
```
- 저장 위치: `logs/YYYYMMDD_HHMMSS_agent1_vs_agent2_callback.csv`
- 콘솔에 실시간 출력 ✅
- 파일 크기: 작음 (핵심만)

### 🔥 동시 사용 (추천!)
```bash
python scripts/run_match.py --agent1 eagle1 --agent2 simple --log-csv --callback-log
```
- **CSV**: 전체 데이터 → 나중에 심층 분석
- **콜백**: 실시간 모니터링 → 즉시 전술 파악

### 커스텀 로그 경로 지정
```bash
# 특정 폴더에 저장
python scripts/run_match.py --agent1 eagle1 --agent2 simple --log-csv my_logs --callback-log my_logs
```

---

## 📋 로그 파일 비교

### SDK 내장 CSV (`*_vs_*.csv`)
```csv
step,agent_id,tree_name,ego_altitude_ft,ego_vc_kts,ego_vx_kts,ego_vy_kts,ego_vz_kts,
roll_deg,pitch_deg,specific_energy_ft,ps_fts,distance_ft,ata_deg,aa_deg,hca_deg,
tau_deg,relative_bearing_deg,alt_gap_ft,closure_rate_kts,turn_rate_degs,
in_39_line,overshoot_risk,tc_type,ata_lead_deg,tau_lead_deg,side_flag,
energy_advantage,energy_diff_ft,alt_advantage,spd_advantage,bfm_situation,
ego_health,enm_health,ego_damage_dealt,enm_damage_dealt,ego_damage_received,
enm_damage_received,in_wez,enm_in_wez,reward,action_altitude,action_heading,
action_velocity,aileron,elevator,rudder,throttle,active_node,active_nodes_path
```

**주요 컬럼**:
- **비행역학**: `ego_altitude_ft`, `ego_vc_kts`, `roll_deg`, `pitch_deg`, `specific_energy_ft`
- **전술 기하**: `distance_ft`, `ata_deg`, `aa_deg`, `hca_deg`, `tau_deg`
- **에너지**: `energy_advantage`, `energy_diff_ft`, `ps_fts`
- **전투 상황**: `bfm_situation`, `tc_type`, `in_39_line`, `overshoot_risk`
- **WEZ**: `in_wez`, `enm_in_wez`
- **체력**: `ego_health`, `enm_health`, `ego_damage_dealt`
- **노드 경로**: `active_nodes_path` (전체 실행 경로)

### 콜백 CSV (`*_callback.csv`)
```csv
step,agent_id,bfm_situation,ego_health,enm_health,ego_damage_dealt,enm_damage_dealt,
in_wez,enm_in_wez,reward,distance,ata_deg,action_altitude,action_heading,
action_velocity,aileron,elevator,rudder,throttle,active_node,active_nodes_count
```

**주요 컬럼**:
- **핵심 전술**: `distance`, `ata_deg`, `bfm_situation`
- **WEZ 상태**: `in_wez`, `enm_in_wez`
- **체력**: `ego_health`, `enm_health`, `ego_damage_dealt`
- **액션**: `action_altitude`, `action_heading`, `action_velocity`
- **활성 노드**: `active_node` (마지막 성공 노드만)

---

## 💡 Golden BT 도출을 위한 활용 방안

### 1. 전술 패턴 분석
```bash
# 여러 라운드 실행하여 패턴 수집
python scripts/run_match.py --agent1 my_bt --agent2 ace --rounds 10 --callback-log analysis
```

**콜백 로그에서 확인할 사항**:
- ✅ **WEZ 진입 빈도**: `in_wez=True` 비율
- ✅ **데미지 효율**: `ego_damage_dealt` 증가 속도
- ✅ **BFM 상황 전환**: `OBFM` → `DBFM` 전환 시점
- ✅ **활성 노드 분포**: 어떤 노드가 가장 많이 실행되는가?

### 2. 파라미터 튜닝 검증
```bash
# 파라미터 변경 전후 비교
python scripts/run_match.py --agent1 my_bt_v1 --agent2 aggressive --rounds 5 --log-csv tuning/v1
python scripts/run_match.py --agent1 my_bt_v2 --agent2 aggressive --rounds 5 --log-csv tuning/v2
```

**CSV 로그로 비교 분석**:
- `ata_deg` 분포 → WEZ 각도 임계값 최적화
- `distance_ft` 분포 → 근접전 거리 임계값 조정
- `energy_diff_ft` 추이 → 에너지 관리 전략 평가
- `active_nodes_path` → 실제 실행된 분기 확인

### 3. 상대별 약점 파악
```bash
# 각 상대와 대전하며 로그 수집
for opponent in simple aggressive defensive ace eagle1 viper1; do
    python scripts/run_match.py --agent1 my_bt --agent2 $opponent --rounds 3 --callback-log analysis/$opponent
done
```

**분석 포인트**:
- 어느 상대에게 `in_wez` 진입이 어려운가?
- 어느 상대가 `enm_in_wez=True`를 자주 만드는가?
- `bfm_situation` 분포 차이 (OBFM vs DBFM 비율)

### 4. 자동 최적화와 연계
```bash
# bt_optimizer.py 실행 후 상위 후보 상세 분석
python tools/bt_optimizer.py --candidates 50 --workers 10

# 상위 후보들의 상세 로그 수집
python scripts/run_match.py --agent1 submissions/alpha1/_temp_opt_12345 --agent2 ace --log-csv --callback-log
```

**최적화 후 검증**:
- 실제 전투에서 파라미터가 의도대로 작동하는가?
- 예상치 못한 분기 실행이 있는가?
- WEZ 진입률이 개선되었는가?

### 5. 실시간 디버깅
```bash
# 콘솔 출력으로 실시간 관찰
python scripts/run_match.py --agent1 my_bt --agent2 ace --callback-log
```

**콘솔 출력 예시**:
```
[   0] A0100 | BFM=BFMSituation.HABFM | HP=100.0/100.0 | Dmg= 0.0 | WEZ=False | Dist=3298m ATA= 90.0deg | Act=[2, 0, 3] | Node=LeadPursuit
[  50] A0100 | BFM=BFMSituation.OBFM  | HP=100.0/ 95.3 | Dmg= 4.7 | WEZ=True  | Dist= 850m ATA=  3.2deg | Act=[2, 1, 3] | Node=PNAttack
[ 100] A0100 | BFM=BFMSituation.DBFM  | HP= 92.1/100.0 | Dmg= 0.0 | WEZ=False | Dist=1200m ATA= 65.0deg | Act=[3, 2, 4] | Node=BreakTurn
```

---

## 🎯 GUIDE.md 분석 결과

### ✅ 잘 작성된 부분
1. **기본 튜토리얼** (섹션 5): 환경 설정부터 첫 매치까지 단계별 설명 ✓
2. **노드 레퍼런스** (섹션 4): 사용 가능한 노드 목록 및 파라미터 ✓
3. **전략 가이드** (섹션 6): 승리 전략 및 BFM 개념 ✓
4. **검증 체크리스트** (섹션 8): 제출 전 확인 사항 ✓
5. **최적화 과정** (섹션 10): Golden BT 도출 과정 설명 ✓

### ❌ 부족한 부분

#### 1. **로깅 시스템 설명 누락** (Critical)
- `--log-csv`, `--callback-log` 옵션 설명 없음
- 두 로깅 방식의 차이 설명 없음
- 로그 파일 분석 방법 없음

#### 2. **개발 스크립트 가이드 부족** (High)
- `bt_optimizer.py` 사용법 간략히만 언급 (섹션 10)
- 실제 명령어 및 옵션 설명 없음
- 최적화 결과 해석 방법 없음
- `test_agent.py` 언급 없음

#### 3. **고급 분석 기법 부족** (Medium)
- CSV 로그 데이터 분석 예제 없음
- Python/Excel로 로그 분석하는 방법 없음
- 통계적 성능 평가 방법 없음

#### 4. **토너먼트 실행 가이드 누락** (Medium)
- `run_tournament.py` 사용법 없음
- Round-robin 대전 방법 없음

#### 5. **커스텀 콜백 작성 가이드 부족** (Low)
- `examples/full_logger_callback.py` 수정 방법 없음
- 자신만의 메트릭 추가 방법 없음

---

## 📝 권장 개선 사항

### 우선순위 1: 로깅 시스템 섹션 추가
GUIDE.md에 새로운 섹션 추가:
```markdown
## 7. 로깅 및 분석

### 7.1 로깅 옵션
- --log-csv: 전체 데이터 로깅
- --callback-log: 실시간 모니터링 + 요약 로깅
- 동시 사용 권장

### 7.2 로그 파일 분석
- CSV 파일 구조 설명
- 주요 컬럼 의미
- 분석 예제 (Python/Excel)

### 7.3 전술 패턴 분석
- WEZ 진입률 계산
- BFM 상황 분포 확인
- 활성 노드 빈도 분석
```

### 우선순위 2: 개발 도구 섹션 보강
```markdown
## 9. 개발 도구 활용

### 9.1 자동 최적화 (bt_optimizer.py)
- 기본 사용법
- 파라미터 공간 정의
- 결과 해석

### 9.2 에이전트 테스트 (test_agent.py)
- 여러 상대와 자동 대전
- 통계 수집

### 9.3 토너먼트 실행
- Round-robin 대전
- 순위 계산
```

### 우선순위 3: 실전 예제 추가
```markdown
## 11. 실전 개발 워크플로우

### 11.1 초기 개발
1. 기본 BT 작성
2. validate_agent.py로 검증
3. simple 상대로 테스트

### 11.2 반복 개선
1. 로그 수집 (--log-csv --callback-log)
2. 약점 분석
3. 파라미터 조정
4. 재테스트

### 11.3 최종 최적화
1. bt_optimizer.py 실행
2. 상위 후보 상세 분석
3. 수동 미세 조정
4. 전체 상대 검증
```

---

## 🎓 학습 곡선 고려사항

### 초보자 (1-2일차)
- ✅ 기본 매치 실행
- ✅ 간단한 BT 작성
- ❌ 로깅 시스템 (설명 부족)
- ❌ 로그 분석 (가이드 없음)

### 중급자 (3-7일차)
- ✅ 복잡한 BT 설계
- ✅ 파라미터 수동 튜닝
- ❌ 자동 최적화 활용 (명령어 예제 부족)
- ❌ 통계적 분석 (방법론 없음)

### 고급자 (8일차+)
- ✅ 커스텀 노드 작성
- ❌ 대규모 실험 설계 (가이드 부족)
- ❌ 커스텀 메트릭 개발 (예제 없음)

---

## 💻 즉시 사용 가능한 명령어 모음

### 빠른 테스트
```bash
# 기본 대전
python scripts/run_match.py --agent1 my_bt --agent2 simple

# 실시간 모니터링
python scripts/run_match.py --agent1 my_bt --agent2 ace --callback-log

# 전체 데이터 수집
python scripts/run_match.py --agent1 my_bt --agent2 ace --log-csv --callback-log
```

### 성능 평가
```bash
# 여러 라운드 테스트
python scripts/run_match.py --agent1 my_bt --agent2 ace --rounds 10 --callback-log

# 전체 상대 테스트 (Windows PowerShell)
$opponents = @("simple", "aggressive", "defensive", "ace", "eagle1", "viper1")
foreach ($opp in $opponents) {
    python scripts/run_match.py --agent1 my_bt --agent2 $opp --rounds 5 --log-csv results/$opp
}
```

### 자동 최적화
```bash
# 빠른 탐색 (50개 후보, 약 30분)
python tools/bt_optimizer.py --candidates 50 --workers 10

# 전체 최적화 (200개 후보, 약 2시간)
python tools/bt_optimizer.py --candidates 200 --workers 20

# 결과 검증
python tools/bt_optimizer.py --validate --rounds 10
```

---

*이 가이드는 AI Combat SDK의 로깅 시스템을 완벽히 활용하여 Golden BT를 도출하는 데 필요한 모든 정보를 담고 있습니다.*

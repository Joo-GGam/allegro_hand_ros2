# Local Changes vs Wonik 원본 (origin/main)

- 원본: https://github.com/Wonikrobotics-git/allegro_hand_ros2.git
- 기준 커밋: `5600ffd` (Merge pull request #9 from voidrobot/main)
- 상태: 아래 변경은 모두 **커밋되지 않은 로컬 수정** (`git diff`로 확인 가능)
- 최종 정리일: 2026-07-31

우리 변경은 크게 두 가지다:
**(1) per-joint torque limit을 YAML로 설정하는 파이프라인 추가**,
**(2) PD 게인을 학습/replay 분석 기반 값으로 재튜닝**.

---

## 1. Torque limit YAML 설정 파이프라인 (신규 기능)

원본 드라이버는 joint별 최대 토크(정규화 duty `[0, 1]`, `1.0` = TORQUE2PWM(1200) 포화)를
`v4_hardware_interface.cpp:594-599`에 **하드코딩된 `default_torque_limit_table`** 로만 초기화한다.
런타임에 ROS 파라미터 `2_torque_limit/<Finger>/<joint>`로 바꿀 수는 있지만, 시작값을 설정 파일로
지정할 방법이 없었다. 이를 `pd_gains.yaml`에서 지정할 수 있도록 3단계를 뚫었다.

### 1-1. `allegro_hand_hardwares/v4/description/config/pd_gains.yaml`
`torque_limits:` 섹션 신설. 현재 16개 joint 전부 **0.8** 사용.
(파일 하단에 이전 실험값 0.512–0.738 세트와 드라이버 기본값 세트가 주석으로 보존돼 있음.)

### 1-2. `allegro_hand_bringup/config/v4/{single_hand,dual_hand}/allegro_hand.ros2_control.xacro`
- `pd_gains.yaml`에서 `torque_limits`를 xacro property로 로드
- ros2_control `<hardware>` 블록에 `torque_joint00` ~ `torque_joint33` 16개 파라미터로 전달
- single/dual 두 파일에 동일한 수정

### 1-3. `allegro_hand_hardwares/common/utility/src/ros_utility.cpp`
`HardwareInfoParser::_load_initial_pd_gains()`에서 `p_`/`d_` 파라미터를 읽는 것과 같은 방식으로
`torque_<joint>` 하드웨어 파라미터를 읽어 `torque_limit_table_`을 덮어쓴다.
파라미터가 없으면 원본 그대로 하드코딩 기본값 유지 (하위 호환).

### 값의 흐름
```
pd_gains.yaml (torque_limits)
  → xacro (torque_joint** hardware param)
  → HardwareInfoParser::_load_initial_pd_gains()  [ros_utility.cpp — 우리 수정]
  → torque_limit_table_
  → 런타임 ROS 파라미터 2_torque_limit/<Finger>/<joint> 초기값 (v4_hardware_interface.cpp:421)
  → v4_sdk.cpp:407 에서 출력 토크 클램프
```

---

## 2. PD 게인 재튜닝 — `pd_gains.yaml`

100 Hz replay 분석(`Debug_results/replay_no_loag(100hz)`) 기반 joint별 차등 튜닝.
학습 nominal은 Kp=3.0 / Kd=0.1.

| 항목 | Wonik 원본 | 우리 값 |
|---|---|---|
| Kp | 0.417 – 0.833 | 1.5 – 3.0 (joint별 차등) |
| Kd | 0.021 – 0.046 | 0.095 – 0.14 (joint별 차등) |

튜닝 원칙 (yaml 헤더 주석 참조):
- **Kd**: overshoot 강도별로 0.10 – 0.16 차등 (예: joint00=0.14, joint10=0.135, joint21=0.13)
- **Kp**: plateau gravity sag가 큰 3개 joint만 상향 — joint00(thumb roll)=3.0, joint21(middle near)=2.8, ※주석에는 joint01도 언급되나 실제 값은 1.6
- 나머지는 학습 nominal 근처 유지

원본 게인은 우리 값 대비 약 1/3 – 1/5 수준으로, 정책 실행 시 추종이 무름.

---

## 3. 빌드 제외 — `COLCON_IGNORE` (미추적 파일)

- `allegro_hand_hardwares/gazebo/plugin/COLCON_IGNORE`
- `allegro_hand_hardwares/plexus/hardware/COLCON_IGNORE`

Gazebo 플러그인과 Plexus 핸드 하드웨어는 사용하지 않으므로 colcon 빌드에서 제외.

---

## 4. 기타

- `allegro_hand_hardwares/v4/hardware/readme.md`: joint 확인용 메모 추가
  (`ros2 run joint_state_publisher_gui joint_state_publisher_gui`)
- `allegro_hand_hardwares/v4/description/config/pd_gains.yaml.bak.1777878308`:
  게인 튜닝 전 백업 (미추적, 삭제해도 무방)

---

## 사용처

이 드라이버 위에서 배포 노드
`~/workspace/allegro_inhand_rotation/hora_allegro/ros2_node.py`가
`/allegro_hand_position_controller/commands`로 목표 위치를 발행한다.
게인/토크 리밋 수정 후에는 `colcon build` + 컨트롤러 재기동 필요
(yaml → xacro는 기동 시점에 로드됨).

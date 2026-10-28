---
title: "구조부터 정했다 — ReAct는 멈춰서 판단하고 TIC-VLA는 움직이며 일한다"
date: 2026-10-27 18:00:00 +0900
summary: "L0~L4 층 정의, marker·LiDAR 정지·VLM LoRA 결정, 수집 에피소드 4종"
tags: [아키텍처, ReAct, LiDAR, 데이터수집, 심부름]
category: ai-agent
---

261026 결과를 바탕으로 ReAct와 TIC-VLA의 역할을 L0~L4로 나눠 정의하고, 목표 단어·도착 판정·심부름 입력·수집 방식을 결정했다.
**구조와 결정은 정했다. 새로 정한 층(L2 실행·L4 심부름·터치 UI·LiDAR)은 하나도 구현되지 않았다.**

---

## 1. 배경

- 261026: 실제 `<think>`로 ReAct live 연동을 시도했으나 goto가 한 번도 나오지 않았다. 관측 품질과 관측 방식 설계가 동시에 문제였다.
- ReAct가 언제 쓰이고 TIC-VLA가 언제 쓰이는지 구조상 정의가 없어서, 도착 판정·심부름 이벤트·데이터 수집이 어디에 속하는지 정할 수 없었다.
- 그래서 기능을 더 붙이기 전에 구조부터 정했다.

---

## 2. 결정 사항

| # | 항목 | 결정 | 이유 |
|---|---|---|---|
| D1 | 목표 단어 | 물체는 그대로(A4 높이 정도), 부르는 단어만 `cooler` → `marker` | 0818 명사 교체 실험에서 cooler 반응이 가장 좋아 썼지만, 실제 쿨러가 아니므로 Gemini의 상식 추론과 충돌한다 |
| D2 | 관측 개선 | VLM LoRA 학습 (상위 블록 중심, 풀 파인튜닝 아님) | 261026에서 VLM이 실내 장면을 서술하지 못함. 지시문 변경만으로 해결된다는 근거 없음 |
| D3 | 학습 순서 | VLM LoRA → Action Expert 재학습·재측정 | Action Expert는 1001 시점 VLM 출력에 맞춰 학습됨. VLM이 바뀌면 입력이 달라진다(구조상 추정, 미확인) |
| D4 | goto 도착 판정 | LiDAR는 **정지에만** 사용 + 카메라 확인(verify) | 교수님 승인. LiDAR만으로는 "앞에 무엇이 있다"까지만 알 수 있어 벽·문 앞 정지를 구분하지 못한다 |
| D5 | 심부름 이벤트 | 적재 완료 / 수령 완료 / 받을 사람 없음 / 취소 → 서봇 화면 터치 UI | 261023 방식(PyQt5, Bero 선례) 유지 |
| D6 | 조이스틱 | 수동 조작·비상정지만 | ReAct 도착 신호는 D4로 대체. **261023 2장 표의 "ReAct 도착 신호 = 조이스틱" 행과 261008 3장 "조이스틱 선례" 근거는 이 문서로 대체한다** |
| D7 | 수령 무응답 | `no_response`로 따로 기록, 성공으로 세지 않음 | Bero는 300초 초과 시 성공 처리. 연구 기록에서는 "탐색 성공 + 사람 없음"을 실패와 구분해야 한다 |

---

## 3. 시스템 구조

원칙: **ReAct는 멈춰 있을 때만 판단하고, TIC-VLA는 움직이는 동안만 일한다.**

```
┌ L4  심부름 관리자 (상태 머신) ◀──▶ 터치 UI (PyQt5, 서봇 모니터)
│     /errand/event 수신, /errand/state 발행, 미션 UUID로 매칭
├ L3  ReAct (Gemini) ── 어디로 갈지 판단 (스킬이 끝났을 때만)
├ L2  스킬 실행기 ───── goto/rotate 수행, 종료 판정 (LiDAR + verify)
├ L1  TIC-VLA ──────── goto 동안 주행 (<think> + waypoint)
├ 관측 모듈 ────────── 장면 → 텍스트 (교체 가능)
└ L0  안전·제어 ────── 변환 → controller → 속도 제한 → watchdog → mux → /cmd_vel
                       ▲ 조이스틱(수동·비상정지), LiDAR 비상정지
```

![TIC-VLA × ReAct 심부름 로봇 시스템 구조도. Jetson Orin NX 온보드 안에 터치 UI와 L4 심부름 관리자, L3 ReAct 상위 에이전트, 관측 모듈, L2 스킬 실행기, L1 TIC-VLA(VLM InternVL3-1B → Action Expert → waypoints), L0 안전·제어(좌표 변환 → controller → 속도 제한 → watchdog → mux → /cmd_vel)가 위에서 아래로 놓이고, 바깥에 사람, 카메라, 클라우드 Gemini, LiDAR, 조이스틱, SerBot II 구동부가 연결된다]({{ '/assets/img/261027/ticvla_react_architecture.png' | relative_url }}){: style="max-width:100%;height:auto"}

*그림 1. 시스템 구조 (261027 초안)*

| 층 | 하는 일 | 하지 않는 일 |
|---|---|---|
| 터치 UI | 이벤트 발행 | 모터 명령 |
| L4 | 에피소드 시작·중단, 심부름 결과 기록 | 주행 관여 |
| L3 | 관측 텍스트로 다음 행동 결정 | 이미지·바퀴 직접 접근 |
| L2 | 스킬 실행, 끝났는지 판정 | 탐색 방향 결정 |
| L1 | 보이는 대상 쪽 waypoint 생성 | 목표 선택 |
| L0 | 항상 켜진 안전 장치 | — |

### 한 번의 탐색 흐름

```
① 관측 모듈: 장면 → 텍스트
② ReAct: 목표가 관측에 있나?
   없음 → think / goto[보이는 랜드마크] / rotate
   있음 → goto[목표]
③ L2가 스킬 실행 (goto면 TIC-VLA 주행, L0 계속 감시)
④ L2 종료 판정 → 결과 + 새 관측을 ReAct에 반환 → ①
⑤ goto[목표] 후 arrived → done[found] → L4 도착 단계
```

목표가 보여도 자동 직행하지 않는다. ReAct가 `goto[목표]`를 내야 한다. 알고리즘 비교(A~D)를 공정하게 하기 위함이다.

---

## 4. 스킬 정의

### ReAct가 쓰는 행동

| 스킬 | 의미 | 실행 | 끝나는 조건 | 반환 |
|---|---|---|---|---|
| `think[...]` | 생각만 | ReAct 내부 | 즉시 | "OK." |
| `goto[대상]` | 보이는 대상 쪽으로 이동. 대상은 직전 관측에 있는 이름만 | L2 → TIC-VLA → L0 | 아래 표 | 결과 + 새 관측 |
| `rotate[left/right]` | 제자리 회전 (각도 미정) | L2 → L0 (TIC-VLA 미사용) | 회전 완료 | 새 관측 |
| `done[found/fail]` | 탐색 종료 | ReAct → L4 | 즉시 | 심부름 상태 전환 |

### goto 종료 판정 (L2)

| 결과 | 조건 |
|---|---|
| `arrived` | LiDAR 전방 d 이내 + verify "대상 맞음" |
| `blocked` | LiDAR 전방 d 이내 + verify "대상 아님" (벽·문 등) |
| `timeout` | T초 안에 위 조건 없음 |

ReAct에 노출하지 않는 내부 기능: `observe()`(장면 → 텍스트), `verify(대상)`(정면 물체가 대상인가).
`blocked`는 오탐이 아니라 재계획 정보로 ReAct에 전달된다.

---

## 5. 심부름 상태 머신

| 상태 | 화면 | 활성 버튼 | 다음 |
|---|---|---|---|
| ① 적재 대기 | "서류를 올려주세요" | 적재 완료, 취소 | 적재 완료 → ② |
| ② 이동 중 | 진행 상태 | **없음** (주행 중 터치 부적절, 정지는 조이스틱) | done[found] → ③ / done[fail] → ⑤ |
| ③ 도착 | "서류를 받아주세요" | 수령 완료, 받을 사람 없음 | 수령 완료 → ④ / 받을 사람 없음·무응답 → ⑤ |
| ④ 완료 | — | — | 종료 |
| ⑤ 실패 | 사유 | — | 종료 |

| 토픽 | 방향 | 내용 | QoS |
|---|---|---|---|
| `/errand/event` | UI → L4 | `{uuid, event: loaded / received / no_receiver / cancel, ts}` | transient_local, depth 1 |
| `/errand/state` | L4 → UI | `{uuid, state, reason}` | transient_local, depth 1 |

Qt 메인 스레드와 ROS spin 스레드 분리(Bero 방식). 터치 UI T1~T3(화면 표시·터치 좌표·한글)는 아직 로봇에서 확인하지 않았다(261023).

---

## 6. 구현 현황 (261027 기준)

![TIC-VLA × ReAct 시스템 구현 현황도. 구조도와 같은 L0~L4 배치에 각 블록의 상태 배지를 붙였다. 구현됨: L3 ReAct, Gemini 연동, L1 TIC-VLA, L0 안전·제어, 카메라, 조이스틱. 부분: 관측 모듈, L2 스킬 실행기. 미구현(점선): 터치 UI, L4 심부름 관리자, LiDAR 연동, VLM LoRA]({{ '/assets/img/261027/ticvla_status.png' | relative_url }}){: style="max-width:100%;height:auto"}

*그림 2. 구현 현황 — ✓ 구현 / ◐ 부분 / ✗ 미구현 (점선)*

| 구분 | 항목 | 근거 |
|---|---|---|
| 구현됨 | L3 ReAct 루프 + 그라운딩 체크, Gemini 연동(503·429 처리, 종료 코드 0/1/2) | 1008 dry-run, 1026 live |
| 구현됨 | L1 TIC-VLA 온보드 추론, 느린/빠른 루프 분리, Action Expert 학습본 | 0804, 0806, 1001 |
| 구현됨 | L0 안전 관문 실차 주행 | 0813. mux 구성까지 실제 존재하는지는 코드로 미확인 |
| 부분 | 관측 모듈: `observe()` = 중립 지시문 `<think>`만. `verify()` 없음 | 1026 |
| 부분 | L2: 어댑터 인터페이스만. `goto`/`rotate`는 추론만, `AutoAdapter`는 `NotImplementedError` | 1008, 1026 |
| 미구현 | goto 실제 주행 연결, 종료 판정, LiDAR 연동(도착·비상정지), rotate 직접 명령 | — |
| 미구현 | L4 심부름 관리자, `/errand` 토픽, 터치 UI | 261023 설계만 |
| 미구현 | VLM LoRA, Action Expert 재학습, `<think>` 라벨 | — |

---

## 7. 데이터 수집

### 오늘

기존 방식(`collect/joystick_colormarker.py`, 대시보드에서 layout·색 주입) 그대로 72개를 수집한다.
이 데이터는 이후 `episode_type=target_visible`로 분류되어 Action Expert 재학습에 다시 쓰인다.
색만 다른 물체 배치이므로 색 구분 능력은 기대하지 않는다(1007, 정답색 우위율 0.216, 우연 0.333).

### 추가할 에피소드 종류

장소·운전 방식·궤적 기록 형식은 그대로. 출발 방향과 도착지만 다르다.

| episode_type | 지시문 | 운전 | 용도 |
|---|---|---|---|
| target_visible | Go to the marker. | 기존과 동일 | Action Expert |
| target_hidden_start | Go to the marker. | 목표가 안 보이는 방향에서 출발, 처음 보인 시점에 버튼 5 | 보인 이후: Action Expert / 이전: VLM 부정 예시 |
| landmark | Go to the &lt;door/desk/shelf/chair/wall&gt;. | 랜드마크로 이동 | Action Expert (중간 목표 주행) |
| stop_negative | Go to the marker. | 목표 아닌 것 앞에서 정지 | verify·VLM 전용. **Action Expert 학습 제외** |

stop_negative를 Action Expert에 넣으면 "marker로 가라"는 지시문으로 벽까지 간 궤적을 배운다.

### Claude Code 사전 확인 결과

| 항목 | 내용 |
|---|---|
| 수집 스크립트 | `collect/joystick_colormarker.py` (0930 데이터셋과 일치). `collect/collect_gotoobject.py`는 `frames/000000.jpg` 파일명으로 `get_numeric_key`를 깨뜨렸던 폐기 버전 — 베이스로 쓰지 않는다 |
| 저장 구조 | `ep{NNNN}_L{layout}_{color}/rgb/rgb_%05d.jpg`, `trajectory.csv`(time,x,y,z,qx,qy,qz,qw), `episode.json`(기존 존재) |
| 전처리 게이트 | `train/02_prepare_dataset.py`가 `ESSENTIAL_FIELDS`(layout, target_color, target_side, instruction, success, final_distance, notes), 색 red/green/blue, layout 1~3, 지시문 색 단어를 강제 → 신규 에피소드는 전부 탈락 |
| 지시문 | `INSTRUCTIONS` 딕셔너리 하드코드 |
| 조이스틱 | 버튼 5가 수집 스크립트 3개 전체에서 미사용 |
| 시작 조건 | 대시보드가 `/collect/set_config`로 layout·색을 주입해야 시작 가능 (`joystick_colormarker.py:625-627`) |

### 구현 방침 (Claude Code에 전달)

| 항목 | 결정 | 이유 |
|---|---|---|
| episode_type 선택 | CLI 인자. 주어진 경우에만 set_config 게이트 우회 | 기존 3색 수집·대시보드 무변경 |
| 전처리 게이트 | episode_type 유무로 분기. 기존 에피소드 검증 규칙은 완화하지 않음 | 0930에서 잡은 데이터 오류를 계속 잡기 위함 |
| 학습 라우팅 | 위 표의 "용도"대로. 제외분은 버리지 않고 별도 목록 기록 | VLM·verify 데이터셋은 이번 범위 밖 |
| 허용 목록 | 스크립트 내 Python 상수 (`TARGET_WORD`, `LANDMARKS`) | 수집 스크립트의 기존 관행(`INSTRUCTIONS`)을 따름 |
| 기존 72개 | 원본 무수정, 파생 메타데이터에 `instruction_v2`(cooler→marker) | 수집 데이터 삭제·수정 금지 규칙 |

권장 비율(필수 아님): 10회 중 target_visible 5 / hidden_start 2 / landmark 2 / stop_negative 1. 마커와 출발 위치는 매번 바꾼다.

---

## 8. 지금 건드리지 않기로 한 것

- **대시보드에 episode_type 추가** — 웹 UI 변경이 들어가 범위가 커진다. 수집이 안정된 뒤 검토한다.
- **VLM·verify용 데이터셋 생성과 `<think>` 라벨링** — 수집이 쌓인 뒤 별도 작업.
- **복귀 방법** — 맵이 없으므로 미정. 수동 복귀 또는 범위 밖.
- **MCP 연결** — 261008 판단 유지(도구 4개, 같은 프로세스 함수 호출).

---

## 9. 과잉해석 경계

**검증된 것:** 없음. 이 문서는 설계와 결정의 기록이다.
**검증되지 않은 것:** 이 구조로 탐색이 실제로 되는가.

- Action Expert가 VLM 출력 특징에 의존한다는 것은 구조상 추정이다. LoRA 후 궤적이 얼마나 바뀌는지 모른다
- LiDAR 사양(토픽·장착 높이·시야각)을 확인하지 않았다. A4 높이 물체가 잡힌다는 것은 사용자 판단이다
- 열린 문 같은 랜드마크는 LiDAR에 잡히지 않아 `arrived`가 나지 않는다. 랜드마크 도착 규칙이 없다
- 연구실 공간이 미가시 객체 탐색에 적합한지 모른다. 초기 속도 제한 0.02 m/s에서 5 m 이동은 4분 이상 걸린다
- 구현 현황 표의 L0 "구현됨"은 0813 기록 기준이며 mux 구성은 코드로 확인하지 않았다

**이 문서로 "구조가 맞다"고 말할 수 없다.** 말할 수 있는 것은 "층별 책임과 스킬 계약을 정했다"까지다.

---

## 10. 다음

### 블로커 — 관측 품질

L2·L4를 먼저 만들어도 관측이 장면을 반영하지 않으면 ReAct가 goto를 내지 못한다(261026). VLM LoRA 전에 기준선을 잡는다.

1. **관측 방식 비교** (원격 가능, 낮 프레임 3~5장 + dark 대조군): 현재 중립 지시문 / 물체 나열 지시문 / InternVL3-1B 기본 VQA. marker 대상 호칭도 같이 확인.
2. 수집 코드 수정 (프롬프트 A: episode_type + 게이트 분기 + 라우팅, 프롬프트 B: 기존 72개 instruction_v2)
3. 신규 4종 에피소드 수집 → `<think>` 라벨 규격 정의 → VLM LoRA

### 사전 점검 (관측 방식 비교, LoRA 전후 동일 적용)

| # | 점검 | 통과 기준 |
|---|---|---|
| P1 | 공간 구조 서술 | 50% 이상 |
| P2 | 호칭 일관성 (지시문에 대상 이름 없는 조건) | 70% 이상 |
| P3 | 부재 물체 존재 단정률 | 20% 이하 (261026: gray bin 4/5) |
| P4 | dark 프레임 장면 서술 비율 | 기준 미정, 261026: 20/20 |

**P3이 가장 위험하다.** LoRA 라벨에 "목표 안 보임" 부정 예시가 빠지면 지시문 되읊기가 그대로 남는다.

### 정해야 할 값

| 항목 | 상태 |
|---|---|
| LiDAR 토픽·높이·시야각 | 미확인 |
| 도착 거리 d, goto 시간초과 T, rotate 각도 | 미정 |
| 랜드마크 도착 규칙 | 미정 |
| 실험 속도 제한 상향 여부 | 미정 |
| 별건: `~/.gemini_env` 자동 로드, 로그에 모델명 기록 | 미수정 (261026) |

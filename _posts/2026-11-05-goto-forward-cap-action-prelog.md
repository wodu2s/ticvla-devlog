---
title: "goto가 2 m를 갔다 — 원인보다 멈출 장치를 먼저 건다"
date: 2026-11-05 18:00:00 +0900
summary: "goto 1회 전진 상한(max_goto_forward_m 1.0) + 어댑터 호출 전 action_start 선기록 작업 지시 작성"
tags: [실차주행, 안전장치, ReAct, goto, 로깅]
category: robot-control
---

261105 14:04 판에서 goto 한 번에 로봇이 2.0 m를 전진해 흰 판넬에 거의 붙었다. 원인 수정보다 먼저 goto 1회 전진 거리 상한과, Ctrl+C로 중단해도 마지막 행동이 남는 선기록을 넣기로 하고 Claude Code용 작업 지시를 작성했다.
**작업 범위와 검증 기준은 정했다. 코드는 아직 한 줄도 바뀌지 않았고, 2 m 전진의 원인도 확인하지 못했다.**

---

## 1. 배경

| 판 | 일어난 일 | 문제 |
|---|---|---|
| 261105 14:04 | `goto[small table]`에서 2.0 m를 전진해 흰 판넬에 거의 붙음 | 멈출 장치가 없었다 |
| 261105 12:36 · 12:54 · 14:04 | Ctrl+C로 중단 | 마지막으로 시작한 행동이 jsonl에 없다 |

- 위 표의 수치와 시각은 작업 지시에 적은 것이고, 이 문서를 쓰면서 원본 jsonl을 다시 열어 확인하지는 않았다.
- 2 m 전진의 원인 후보는 두 가지다. ① bbox가 탁자에서 판넬로 옮겨갔다. ② Gemini `distance_m`을 과대추정했다(실측 대비 +0.2~+0.9 m). **어느 쪽인지 확인하지 못했다.**
- 원인과 상관없이 goto 1회의 누적 전진을 막는 장치가 없었다. 그래서 이번 작업은 원인 수정이 아니라 안전장치다.

---

## 2. 현재 코드 확인 (노트북 사본 기준)

노트북 `C:\dev\ticvla` (브랜치 `ticvla-serbot`, `d091d1b`)를 읽어서 확인했다. **Jetson 작업 트리에는 미커밋 변경(run_header, b1 config·few-shot 등)이 있으므로 이 표와 다를 수 있다.**

| 항목 | 확인된 사실 |
|---|---|
| goto 종료 조건 (`_goto`) | ① `box_h ≥ arrive_bbox_h` 또는 박스 하단 `ymax ≥ 0.97` 또는 `0 < dist ≤ stop_distance_m` → 도착 ② `_locate` 실패 → lost ③ `max_servo_iters`회 소진 → 한도 |
| 한 스텝 전진 | `_forward_by(step)`, `step ≤ max_step_m` |
| 누적 전진 상한 | **없다.** 기본값 `max_servo_iters=6`, `max_step_m=0.6`(`agent/config.py`)이면 이론상 최대 3.6 m. yaml 실제 값은 미확인 |
| `_forward_by`의 이동량 | 스텝 시작점에서 odom `hypot(dx, dy)`. 시간 초과(`motion_timeout_sec`)는 `MotionError` |
| odom 형식 | 콜백에서 `(x, y, yaw)` 튜플로 저장 |
| jsonl 기록 | `JsonlLogger.write`가 매 줄 `flush`. 행동 레코드는 `adapter.goto`/`rotate` **호출이 끝난 뒤** 기록 |
| dry-run | `--dry-run` 있음. 관측을 표준입력으로 받는다 |

분석: 작업 지시의 전제("상한 없음", "호출 후 기록")는 노트북 사본과 맞는다. 다만 `max_step_m`이 이미 스텝당 상한 역할을 하고 있으므로, 이번 상한은 **스텝 단위가 아니라 goto 1회 누적**이라는 점을 구분해야 한다.

---

## 3. 구현 범위

### 3.1 goto 전진 상한

| 항목 | 내용 |
|---|---|
| 설정 키 | `max_goto_forward_m` (기본 1.0). `react_direct.yaml`, `react_direct_b1.yaml`에 같은 값 |
| 판정 | 기존 종료 조건은 그대로 두고 **"또는 전진 변위 ≥ 상한"**을 추가. 걸리면 cmd_vel 0 후 종료 |
| 전진 변위 | goto 시작 시점 동체 전방 기준: `dx·cos(yaw0) + dy·sin(yaw0)` |
| 종료 사유 | 기존 분류(arrived / lost / timeout 등) + `forward_cap` |
| LLM에 주는 관측 | `max forward distance reached (X.XX m), target not confirmed` 수준. **arrived로 보고하지 않는다** |

전방 변위로 재는 이유:
1. goto 안에서 bearing 보정 회전이 섞인다. 월드 x만 보면 로봇이 처음 바라보던 방향이 아닌 축으로 잰다.
2. 스텝별 `hypot`을 더하면 좌우로 비틀린 경로가 전방 거리보다 크게 잡힌다. 판넬에 다가간 정도를 재려면 처음 바라본 방향으로의 투영이 맞다.

### 3.2 행동 선기록

```
[step N]  action_start {step, action, 인자, 시각}  → write + flush
          adapter.goto / rotate / ...             ← Ctrl+C 가능 구간
          결과 레코드 (기존 형식 유지, 필드 추가만)
Ctrl+C →  cmd_vel 0  →  (가능하면) interrupted 1줄
```

- 정지가 먼저다. interrupted 기록 때문에 cmd_vel 0 전송이 늦어지면 안 된다.
- action_start만 있고 결과가 없는 상태로 끝나도 된다.

### 3.3 하지 않는 것

| 항목 | 지금 건드리지 않는 이유 |
|---|---|
| `prompts/` few-shot (B0·B1) | 오늘 실험 조건이 바뀐다 |
| LiDAR 연동 | LiDAR L2 실측이 먼저다 |
| 속도(0.40/0.50) 등 주행 파라미터 | 상한 키 하나만 추가해야 이전 판과 비교된다 |
| `arrive_bbox_h`, `distance_m` 계산 | 이번 작업은 원인 수정이 아니다. 원인이 확인되지 않은 상태에서 판정을 바꾸면 무엇이 효과를 냈는지 모른다 |
| 맵·SLAM·거리 지도 | 연구 요구 조건 위반 |
| Jetson 미커밋 변경 (run_header, b1 config·few-shot, `capture_probe_frames.py`, `eval_book_probe.py`, .env 로딩) | 되돌리거나 덮어쓰지 않는다. 커밋·push도 하지 않는다 |

---

## 4. 검증 기준

### 로봇 없이

| # | 점검 | 통과 기준 |
|---|---|---|
| V1 | 가짜 odom, 직진만 | 1.0 m에서 정지, 종료 사유 `forward_cap` |
| V2 | 가짜 odom, 회전 섞임 | 월드 x가 아니라 전방 변위로 판정 |
| V3 | dry-run 중 행동 1개 실행 도중 KeyboardInterrupt | jsonl 마지막 줄이 `action_start` 또는 `interrupted` |
| V4 | 같은 jsonl | `run_header`가 여전히 첫 줄 |

### 실차 (사용자 직접)

| # | 점검 | 통과 기준 |
|---|---|---|
| S2 | 판넬 정면 약 2 m에서 goto | 정지 위치에서 판넬까지 줄자로 **≥ 0.3 m** |
| S3 | goto 도중 Ctrl+C 1회 | 마지막 행동이 jsonl에 있다 |

**S2가 가장 위험하다.** 상한 1.0 m는 출발점 기준이다. 판넬까지 2 m라면 여유가 있지만, 출발점이 판넬에 더 가까우면 상한이 걸리기 전에 붙을 수 있다. 기존 정지 판정이 실패하는 조건에서는 상한만이 마지막 장치다.

---

## 5. 과잉해석 경계

**검증된 것:** 노트북 사본(`d091d1b`)의 goto에 누적 전진 상한이 없고, 행동 기록이 어댑터 호출 뒤에 쓰인다.
**검증되지 않은 것:** Jetson 코드가 같은가. 상한이 실제로 판넬 앞에서 멈추게 하는가. 2 m 전진의 원인.

- 14:04 판의 2.0 m와 `distance_m` 과대추정 +0.2~+0.9 m는 이 문서에서 원본 로그로 재확인하지 않았다
- 상한 1.0 m는 안전 여유로 정한 값이다. 과제 성공률에 어떤 영향이 있는지 측정하지 않았다
- 상한에 걸린 goto가 늘면 ReAct가 goto를 반복 호출할 수 있다. 그때 LLM이 `target not confirmed`를 어떻게 받아들이는지 모른다

**이 작업으로 "goto가 안전하다"고 말할 수 없다.** 말할 수 있는 것은 "goto 1회의 전진이 상한을 넘지 않게 하고, 중단해도 마지막 행동을 남기도록 작업 범위를 정했다"까지다.

---

## 6. 다음

### 블로커 — Jetson 작업 트리에서 구현·검증

작업 지시는 Jetson에서 실행해야 한다. 노트북 사본에는 `react_direct_b1.yaml`과 run_header가 없어서 V4와 b1 설정을 확인할 수 없다.

1. Jetson에서 작업 전 확인 결과를 먼저 받는다. 2장 표와 다르면 멈춘다
2. V1~V4 통과 후 S2·S3
3. 그 다음에 2 m 전진 원인 확인: 14:04 판 프레임에서 bbox가 판넬로 옮겨갔는지, `distance_m`이 몇이었는지

| 항목 | 상태 |
|---|---|
| Jetson 코드와 2장 표 일치 여부 | 미확인 |
| `react_direct*.yaml`의 `max_servo_iters`·`max_step_m` 실제 값 | 미확인 |
| 2 m 전진 원인 (bbox 이동 / distance_m 과대) | 미확인 |
| LiDAR L2 실측 | 다음 작업 |

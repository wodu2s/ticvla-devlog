---
title: "진짜 &lt;think&gt;를 넣었더니 ReAct가 한 번도 움직이지 않았다"
date: 2026-10-26 18:00:00 +0900
summary: "보이는 쿨러는 말하지 않고, 없는 회색 통은 있다고 했다"
tags: [ReAct, VLM, grounding, 환각, 관측]
---

로봇을 움직이지 않은 채 원격으로 실제 TIC-VLA `<think>`를 처음 뽑고, ReAct 루프에 live로 연결했다.
**연동 경로는 동작한다. 그러나 `<think>`는 장면 속 물체를 서술하지 않았고, 없는 것을 있다고 했다 — 현재 관측으로는 ReAct가 goto를 낼 수 없다.**

---

## 1. 배경

261008에서 ReAct 루프를 dry-run(사람이 쓴 관측)으로 검증했다. 남은 블로커는 실제 `<think>` 덤프였다.

이번 작업 조건:

| 항목 | 값 |
|---|---|
| 접속 | 원격 SSH (학교, 로봇 옆에 사람 없음) |
| 로봇 구동 | 없음. `/cmd_vel` 실행 전후 "Unknown topic"(publisher 0) 확인 |
| 실행 방식 | Claude Code용 프롬프트 A(덤프) / B(ReAct live) |
| 가용 메모리 | 15Gi 중 8.1Gi (실행 전) |

---

## 2. 준비

### 설정

- `config/react_agent.yaml`의 `ticvla.model_path` / `ckpt_path` / `frame_dir`가 전부 null이었다 → `--dry-run` 없이 실행하면 ValueError.
- 레포 안 단일 후보 경로로 채웠다. 원본은 `config/react_agent.yaml.bak`.
- `LiveTicVlaAdapter`는 `TICVLA.predict()`만 호출한다. `goto`/`rotate`도 추론만 하고 `cmd_vel`을 내지 않는다(코드 주석 명시).

### 프레임

ROS·브리지·컨트롤러 노드는 띄우지 않고 `scripts/color_target.py capture`로 CSI 카메라에서 직접 1장을 찍었다.

| 프레임 | 출처 | 상태 |
|---|---|---|
| dark | `logs/think_dump/capture_raw/frame_0001.jpg` | 연구실 소등 상태(KST 01:05). 평균 2.9/255, 최대 106, 99.67% 픽셀 ≤30 |
| scene | 기존 낮 촬영 프레임 중 1장 → `logs/think_dump/scene_frame/` | 쿨러 3개가 보임. 원본 폴더명은 미확인 |

어댑터 입력 규칙(`num_delayed=4`)에 맞춰 같은 이미지를 5장 복제했다(md5 동일).

---

## 3. 결과 ① — 고정 프레임 `<think>` 덤프

**로그:** `logs/think_dump/think_static_20261026_014200.jsonl` (41줄 = 메타 1 + 프레임 2종 × 지시문 4개 × 5회)
**degraded:** scene 0/20, dark 0/20 — `</think>` 닫힘은 매번 정상.

### 부재 물체에 대한 존재 단정 (P3 미리보기)

| 프레임 | "Find the gray bin." | "Find the cup." |
|---|---|---|
| scene (gray bin·cup 없음) | **4/5** 존재 단정 ("on my left a gray bin") | 0/5 (단, 1회 "red sofa" 환각) |
| dark (완전 암흑) | 1/5 ("A dark bin is ahead") | 0/5 (단, "cafe", "pedestrians" 환각) |

### 장면 자체의 환각

dark 프레임 **20/20**이 "paved walkway", "quiet plaza", "lamp post", "trees" 같은 구체적 야외 환경을 서술했다.
TIC-VLA의 원 학습 데이터(SCAND, 야외 보행로)와 같은 종류의 장면이다.

### 호칭 일관성 (P2)

"red cooler" / "blue cooler" 지시문은 5/5 회차 모두 같은 이름을 썼다.
그러나 지시문 문구를 되읊은 것에 가깝다. **본 것을 일관되게 부른다는 증거로 쓸 수 없다.**

### gray bin 편향

scene에서 cup은 0/5인데 gray bin만 4/5다. Claude Code는 `color_target.py` 주석에
"학습 데이터가 거의 회색 통"이라는 기록이 있다고 보고했다(직접 확인 안 함 — 미확인).
지시문 되읊기 현상과 구분되지 않는다.

---

## 4. 결과 ② — ReAct live 실행

설정: `config/react_agent_static.yaml` (`limits.max_goto_calls` 1, `limits.max_total_steps` 3). 다른 키는 원본과 동일.

| run | 프레임 | 목표 | 행동 순서 | 종료 코드 |
|---|---|---|---|---|
| 1 | dark | Find the cup. | think → rotate[right] → done[fail] | **2** (Gemini 503, 재시도 상한) |
| 2 | scene | Go to the red cooler. | think → rotate[right] → think → done[fail] | **1** (스텝 상한) |

로그: `logs/react/react_20261026_010156.jsonl`, `logs/react/react_20261026_015336.jsonl`
두 실행 모두 `valid: false` 0건, **goto 0회**.

### run 2 init 관측 원문 (실제 `<think>`)

```
"I am in an open office by a wall with tiled floor and a clear view ahead.
A low white block is ahead-left, likely stationary. ..."
```

쿨러 3개가 보이는 프레임인데 "cooler"가 한 번도 나오지 않는다.

### goto가 나올 수 없는 구조

| 위치 | 동작 |
|---|---|
| `tic_vla_adapter.py:211-212` (`initial_observation()`, `rotate()` 동일) | 항상 중립 지시문 `'Move forward safely and efficiently.'`으로 `predict()` 호출 |
| `react_loop.py:270` | goto 대상이 직전 관측 문자열에 **리터럴로** 있어야 통과 |

목표는 TIC-VLA에 전달되지 않는다. 중립 서술에 목표 이름이 우연히 나오지 않으면 goto를 시도할 수 없다.
반대로 지시문에 목표를 넣으면 3장에서 본 것처럼 없는 것도 있다고 한다. **어느 쪽으로도 올바른 관측이 나오지 않는다.**

run 1에서는 환각된 야외 장면을 근거로 Gemini가 "tables, benches, trash cans를 찾자"고 추론했다.
상위 판단층은 관측을 검증하지 않고 그대로 따라간다.

---

## 5. 해석 (가설, 미확인)

1001에서 학습한 것은 Action Expert뿐이다. `<think>`를 만드는 VLM은 SCAND로 학습된 상태 그대로이고
연구실 실내 장면 서술을 배운 적이 없다. 3·4장 결과와 일치하지만 검증하지 않았다.

이 문제는 ReAct만의 것이 아니다. A~D 네 조건 모두 `<think>`를 관측으로 쓴다.
261008에서 경고한 "관측 품질이 결과를 지배한다"가 실제로 확인된 것이다.

---

## 6. 산출물

| 파일 | 상태 |
|---|---|
| `config/react_agent.yaml` | `ticvla.*` 경로 채움, `.bak` 백업 |
| `config/react_agent_static.yaml` | 고정 위치 테스트용 상한 축소 |
| `agent/dump_think_static.py` | 신규. 어댑터 내부 메서드 재사용(로직 복제 없음) |
| `logs/think_dump/think_static_20261026_014200.jsonl` | 41줄 |
| `logs/think_dump/scene_frame/`, `static_frame/` | 고정 프레임 5장씩 보존 |

### 별건 (수정 안 함)

- `~/.gemini_env`가 자동 로드되지 않는다. 매 실행 전 수동 `source` 필요.
- 로그 첫 줄에 Gemini 모델명이 찍히지 않는다. `react_loop.py`/`llm_gemini.py`에 해당 코드가 없다. **261008 문서 4장의 "로그 첫 줄에 모델명과 실행 시각을 기록한다"는 사실과 다르다.**
- `agent/`가 git 커밋에 보이지 않는다(최근 커밋 `d4f969c`는 학습 설정). untracked 여부 미확인.

---

## 7. 과잉해석 경계

**검증된 것:** 실제 `<think>` → 추출 → Gemini → 파싱 → JSONL 경로가 끊김 없이 동작한다. 현재 체크포인트 + 중립 지시문 조합에서 scene 프레임의 쿨러가 서술되지 않았다.
**검증되지 않은 것:** `<think>`가 일반적으로 관측 채널로 쓸 수 없는가. 원인이 VLM 미학습인가.

- scene 프레임 **1장**, 지시문당 반복 **5회**
- dark 프레임은 극단적 분포 밖 입력이다. 20/20 환각은 가장 약한 근거다
- ReAct live는 **2회**, 로봇 고정이라 goto 이후 관측이 변하지 않는 조건
- P1(공간 구조 서술)은 지점이 1개라 측정하지 않았다

**이 결과로 "TIC-VLA `<think>`는 관측으로 못 쓴다"고 말할 수 없다.** 말할 수 있는 것은 "현재 체크포인트와 중립 지시문으로는 장면 속 목표를 서술하지 않았고, 지시문에 넣으면 부재 물체도 있다고 했다"까지다.

---

## 8. 다음

### 블로커 — 관측 품질

ReAct·ToT·Reflexion 비교 이전에 관측이 장면을 반영해야 한다. 후속 결정은 261027 문서에 기록한다.

### 사전 점검 (관측 방식 비교 시)

| # | 점검 | 통과 기준 |
|---|---|---|
| P1 | 공간 구조(복도/문/열린 방향) 서술 | 50% 이상 |
| P2 | 같은 대상 호칭 일관성 — **지시문에 대상 이름이 없는 조건에서** | 70% 이상 |
| P3 | 부재 물체 존재 단정률 | 20% 이하 |
| P4 | dark 프레임에서 장면 서술 비율 (대조군) | 낮을수록 좋음, 기준 미정 |

**P3이 가장 위험하다.** 현재 4/5 = 80%. 실패하면 D(Reflexion)가 가장 큰 손해를 본다 — 틀린 반성문이 메모리에 남는다.

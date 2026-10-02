# 어노테이션 묶음: subtask 하나마다 영상 하나와 속도 시계열

> 팀원이 "성공 뒤에 무엇을 하는가" 를 눈으로 라벨링하려고 요청한 묶음이다 (2026-10-02). 묶음을 만드는
> 도구는 `calvin/annotation_clips.py`, 수치의 뜻은 `calvin/RESULTS.ko.md` §3.2 와 Notion 의 CALVIN 페이지에 있다.

## 무엇이 들어 있나

CALVIN 새 규약 평가 (`--hold-random 15,300 --score persist`, 공식 시퀀스 앞 100 개) 의 덤프에서, 오라클이
성공을 선언한 subtask 하나를 영상 하나로 잘랐다. 두 모델만 넣는다 (PALM 은 넣지 않는다).

| 모델 | 덤프 | 클립 수 |
|---|---|---|
| π0.5 SFT (RLinf 공개 CALVIN ABC) | `out/calvin/persist_sft` | 313 |
| π0.5 RL Flow-SDE (RLinf 공개) | `out/calvin/persist_rl` | 366 |

폴더 구성은 모델마다 `clips/` (mp4), `speed/` (프레임마다 손 속도 csv), 그리고 묶음 뿌리의 `index.csv`,
`annotation.csv` 다.

## 영상 보는 법

- 성공 판정 2 초 전부터 시작해 대기가 끝날 때까지 간다. 대기는 subtask 마다 0.5 ~ 10 초 사이의 무작위다.
- **빨간 테두리가 켜지는 순간이 오라클의 성공 판정 시점이다.** 기존 CALVIN 평가는 바로 여기서 끊는다.
- 글자는 모델 이름, 에피소드와 subtask, 성공 판정을 0 으로 둔 시간, 그 프레임의 손 속도다. 마지막 프레임에
  `success kept` 또는 `success LOST` 가 뜬다. 대기가 끝나는 시점에 성공 조건이 아직 참인지다.
- 아래 띠가 손 속도 시계열이다. 흰 선이 속도, 붉은 세로선이 성공 판정, 밝은 세로선이 지금 프레임이다.
  가로선 둘은 판정 기준이다. 아래가 정지 기준 1 mm/frame, 위가 "계속 움직인다" 기준 7 mm/frame 이다.
- 영상은 15 fps 로 실시간이다 (30 Hz 에서 한 프레임씩 건너뛴다). 속도 csv 는 건너뛴 프레임을 뺀 값이고
  단위는 mm/frame 이다. 30 Hz 라 1 mm/frame 이 초속 3 cm 다.

## 라벨링

`annotation.csv` 의 `label` 칸에 아래 중 하나를 적는다. `note` 는 자유, `annotator` 는 이름이다.

| label | 뜻 |
|---|---|
| `same` | 끝낸 동작을 계속한다 (슬라이더를 더 밀고, 서랍을 더 당긴다) |
| `next` | 다른 곳으로 가서 다른 일을 시작한다 |
| `hesitate` | 오가거나 멈칫하며 머뭇거린다 |
| `drift` | 느리게 표류한다. 뜻이 있는 동작으로 보이지 않는다 |
| `stop` | 멈춰 선다 |
| `undo` | 해 놓은 것을 되돌린다 (든 블록을 놓고, 켠 LED 를 끈다) |

`index.csv` 의 `verdict_auto` 는 속도 규칙이 자동으로 낸 판정 (`stopped`, `slow`, `moving`) 이라 사람 라벨과
비교할 대상이다. 같은 열에 있는 `held_end` 는 대기 끝에 성공이 남았는지, `wait_s` 는 대기 길이다.

## 속도 데이터로 그림 그리기

    import pandas as pd, matplotlib.pyplot as plt
    d = pd.read_csv("sft/speed/sft_ep0003_s1.csv")
    plt.plot(d.t_rel_success_s, d.speed_mm_per_frame)
    plt.axvline(0, color="r"); plt.axhline(1.0, color="g"); plt.axhline(7.0, color="orange")
    plt.xlabel("성공 판정 뒤 [s]"); plt.ylabel("손 이동 [mm/frame]")

`index.csv` 의 `work_speed_median` 은 성공 전 속도, `after_speed_median` 은 성공 뒤 (유예를 뺀) 속도,
`ratio_after_over_work` 는 둘의 비다. 클립마다 이 비의 중앙값은 SFT 0.89, RL 0.79 다. 끝낸 뒤에도 작업 속도에
가깝게 움직인다는 뜻이다.

자동 판정을 모아 보면 이렇다. 묶음의 수치는 Notion 의 CALVIN 페이지 5 절과 같다.

| 모델 | 클립 | 멈춤 | 느림 | 계속 | 대기 끝 성공 유지 |
|---|---|---|---|---|---|
| π0.5 SFT | 313 | 0 | 150 (48%) | 163 (52%) | 224 (72%) |
| π0.5 RL Flow-SDE | 366 | 0 | 79 (22%) | 287 (78%) | 256 (70%) |

## 다시 만들기

    PY=/data2/hyeongjinkim/miniconda3/envs/liberovla/bin/python
    for i in 0 1 2 3 4 5; do $PY calvin/annotation_clips.py --dump out/calvin/persist_sft --model sft \
        --label "pi0.5 SFT" --out out/calvin/annotation/sft --shard $i/6 --stride 2 --cpu-render & done
    $PY calvin/annotation_clips.py --out out/calvin/annotation/sft --merge

CPU 렌더는 GPU 를 쓰지 않는다 (pybullet TinyRenderer). GPU 가 날 때는 `--cpu-render` 를 빼면 EGL 로 열 배
빠르다. 이미 만든 클립은 다시 그리지 않으므로 끊기면 같은 명령을 다시 내면 된다.

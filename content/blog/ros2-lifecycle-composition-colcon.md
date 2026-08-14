---
title: "TIL 2026-08-13 | ROS2 Lifecycle · Composition · Launch와 colcon 실행 경로 문제"
date: 2026-08-13 09:00:00 +0900
categories: [TIL, ROS2]
tags: [ros2, colcon, lifecycle, composition, launch, cmake, setup-bash, symlink-install]
description: "ROS2의 Lifecycle Node·Composition·Launch 개념을 정리하고, colcon 빌드 후 실행이 되지 않던 문제를 실행 시점 환경변수 관점에서 해결한 기록. source install/setup.bash와 colcon 빌드 옵션 팁까지."
prev: ""
next: ""
---

## 오늘의 문제 (Why · How · What)

### Why — 무엇이 문제였나

`colcon build` 중 `CMakeLists.txt`에서 executable을 링크하는 과정에서 오류가 발생했다.
빌드 자체는 통과하는 경우도 있었지만, 정작 **실행 시점에 필요한 라이브러리/패키지 경로를 찾지 못해 실행이 되지 않는** 문제였다.

즉 증상은 두 갈래였다.

- 링크 단계에서 곧바로 실패하는 경우
- 빌드는 성공했는데 실행하면 의존성을 찾지 못하는 경우

### How — 어떻게 해결했나

실행파일이 의존성을 찾아가는 **경로 설정**을 손봐서 해결했다.
구체적으로는 아래 세 환경변수가 실행 시점의 탐색 경로로 쓰인다는 것을 확인하고, 값이 올바르게 잡혀 있는지 점검·수정했다.

| 환경변수 | 역할 |
| --- | --- |
| `AMENT_PREFIX_PATH` | ament 패키지(리소스, 인터페이스 등) 탐색 경로 |
| `PYTHONPATH` | 파이썬 모듈/노드 탐색 경로 |
| `LD_LIBRARY_PATH` | 런타임 공유 라이브러리(.so) 탐색 경로 |

빌드 산출물이 있는 워크스페이스를 제대로 source 하지 않으면 이 값들이 비거나 이전 워크스페이스를 가리키게 되고, 그 상태에서는 빌드가 성공해도 실행이 깨진다.

### 그래서 `source install/setup.bash`

위 세 환경변수는 **직접 export 하는 게 아니라** `colcon build`가 만들어 주는 `install/setup.bash`를 source 하면 한 번에 잡힌다.

```bash
colcon build
source install/setup.bash
# 또는
. install/setup.bash
```

- `source X` 와 `. X` 는 **완전히 같은 동작**이다. `.` 이 POSIX 표준이고 `source` 는 bash/zsh 확장이라, `sh`(dash)로 돌리는 스크립트 안에서는 `.` 만 동작한다.
- 중요한 건 두 방식 모두 **현재 셸 안에서 실행된다**는 점이다. 실수로 `./install/setup.bash` 처럼 실행하면 자식 프로세스에서 돌고 끝나서, 환경변수가 부모 셸에 남지 않는다. → 빌드는 됐는데 실행이 안 되는 그 상태 그대로다.
- 새 터미널을 열 때마다, 그리고 새로 빌드할 때마다 다시 source 해야 한다.

| 파일 | 무엇을 불러오나 |
| --- | --- |
| `install/setup.bash` | underlay(`/opt/ros/<distro>`)까지 포함한 전체 환경 |
| `install/local_setup.bash` | 이 워크스페이스에서 빌드한 패키지만 |

### What — 무엇을 배웠나

- **빌드 성공 ≠ 실행 성공.** CMake 링크 설정이 맞아도 실행 시점의 환경변수가 어긋나면 노드는 뜨지 않는다.
- ROS2/colcon 결과물은 *컴파일 타임 링크*와 *런타임 경로 탐색*이라는 두 층으로 나눠서 봐야 한다.
- 문제를 "왜 안 되지"가 아니라 **"실행파일이 무엇을, 어느 경로에서 찾고 있나"**로 좁히면 원인 추적이 훨씬 빨라진다.

---

## 오늘 정리한 개념

### Lifecycle Node

- 노드의 생애(life)를 **명시적인 상태로 관리**하는 노드.
- 노드가 떠 있다 = 동작 중, 이 아니라 unconfigured → inactive → active 처럼 상태 전이를 통해 제어한다.
- 시스템 기동/정지 순서를 보장해야 할 때, 특정 노드가 준비되기 전에 데이터가 흐르는 것을 막고 싶을 때 유용하다.

### Composition

- 여러 노드를 **한 프로세스에 올려 메모리 상에서 통신**할 수 있게 하는 방식.
- 프로세스 간 직렬화/복사 없이 데이터를 주고받으므로, 대용량 데이터를 오가는 노드 묶음에서 이득이 크다.
- 판단 기준
  - **합성한다:** 카메라 → 인지처럼 큰 데이터가 오가고 성능이 중요한 노드들
  - **분리한다:** 안전성·독립성이 중요한 노드 (하나가 죽어도 나머지는 살아야 하는 경우)

### Launch

- 시스템 **기동 오케스트레이션** 담당.
- 어떤 노드를 어떤 순서로, 어떤 설정으로 띄울지 한 곳에서 기술한다.
- 파라미터 값을 수정해서 띄울 수 있고, 임시 파라미터를 선언해 사용하는 것도 가능하다.

> 같은 노드를 여러 개 띄울 때는 **네임스페이스로 구분**할 것.
> 이름이 겹치면 토픽·서비스가 충돌해서 의도치 않게 서로의 데이터를 먹는다.

---

## colcon 빌드 팁

```bash
# 특정 패키지만 빌드
colcon build --packages-select my_package

# 해당 패키지와 그 의존성까지 함께
colcon build --packages-up-to my_package

# 특정 패키지 제외
colcon build --packages-skip heavy_package

# 복사 대신 심볼릭 링크로 install
colcon build --symlink-install
```

### `--packages-select`

워크스페이스 전체를 다시 돌리지 않고 원하는 패키지만 빌드한다. 패키지가 늘어날수록 체감 차이가 커진다.

다만 선택한 패키지 **하나만** 빌드하므로, 의존 패키지 쪽을 건드렸다면 그 변경은 반영되지 않는다. 특히 `msg`/`srv` 같은 인터페이스를 수정했을 때 `--packages-select`로 소비자 패키지만 다시 빌드하면 옛 인터페이스를 그대로 물고 있게 된다. 이럴 땐 의존성까지 훑는 `--packages-up-to`가 안전하다.

### `--symlink-install`

install 디렉터리에 파일을 **복사하는 대신 소스로 심볼릭 링크**를 건다.

- 파이썬 노드, launch 파일, yaml 파라미터처럼 컴파일이 필요 없는 파일은 **수정하면 재빌드 없이 바로 반영**된다. 파라미터 하나 고칠 때마다 빌드를 돌리지 않아도 되니 개발 루프가 훨씬 짧아진다.
- C++ 소스는 당연히 재빌드가 필요하다. 심볼릭 링크는 설치 단계를 건너뛰게 해줄 뿐 컴파일을 없애주지 않는다.
- 중간에 이 옵션을 켰다 껐다 하면 install 디렉터리에 복사본과 링크가 섞여 꼬일 수 있다. 그럴 땐 `build/ install/ log/`를 지우고 처음부터 다시 빌드하는 게 빠르다.

---

## 정리

오늘은 ROS2를 "노드를 어떻게 띄우고 묶고 관리할 것인가"라는 운영 관점에서 본 하루였다.
Lifecycle은 *언제* 동작할지, Composition은 *어디에서(어느 프로세스에서)* 동작할지, Launch는 *어떤 조합으로* 동작할지를 각각 결정한다.
그리고 colcon 이슈를 통해, 그 모든 설계가 결국 **실행 시점의 경로가 맞아야 비로소 돌아간다**는 것을 확인했다.
그 경로를 맞춰주는 스위치가 `source install/setup.bash` 한 줄이고, 거기에 도달하는 시간을 줄여주는 게 `--packages-select`와 `--symlink-install`이다.

---

<!-- prev/next 내비게이션 -->

| ← Prev | Next → |
| --- | --- |
| _(이전 TIL 없음 — 첫 글)_ | _(작성 예정)_ |

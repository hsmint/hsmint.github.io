---
title: "[TIL] 스마트 포인터 leaks 실측 · ROS2 Executor/spin · TurtleBot3 Navigation"
date: 2026-08-11T21:00:00+09:00
description: "raw/smart 포인터의 실제 누수량을 macOS leaks로 실측하고, ROS2 콜백·executor·spin 실행 모델과 TurtleBot3 Gazebo SLAM→Navigation 흐름까지 하루치 학습을 정리"
image: "images/post/post-6.jpg"
categories: ["TIL"]
tags: ["cpp", "smart-pointer", "memory-leak", "ros2", "turtlebot3", "navigation"]
type: "post"
nextp: ""
prevp: ""
---

## 1. macOS leaks로 실측하는 스마트 포인터의 메모리 누수 방지

### 실험 코드 — 매크로 하나로 raw/smart 스위칭
```cpp
#ifndef USE_SMART_POINTER
#define USE_SMART_POINTER 0
#endif

class Motor {
  int id_;
  double speed_ = 0.0;
 public:
  explicit Motor(int id) : id_(id) {}
  void setSpeed(double v) { speed_ = v; }
};

int main() {
#if USE_SMART_POINTER
  for (int i = 0; i < 1000; ++i) {
    auto m = std::make_unique<Motor>(i);
    m->setSpeed(i);
  }
#else
  for (int i = 0; i < 1000; ++i) {
    Motor* m = new Motor(i);
    m->setSpeed(i);
  }
#endif
}
```
- `Motor`를 1000번 만들고 버리는 루프를 raw `new`와 `make_unique` 두 가지 방식으로 각각 실행
- `#ifndef USE_SMART_POINTER` 덕분에 컴파일 시 `-D` 플래그 하나로 두 버전을 같은 소스에서 뽑아낼 수 있음

### 컴파일 — 매크로 값 주입
```bash
g++ -Wall -Wextra -std=c++17 -DUSE_SMART_POINTER=0 -o leaks_raw   leaks_demo.cpp
g++ -Wall -Wextra -std=c++17 -DUSE_SMART_POINTER=1 -o leaks_smart leaks_demo.cpp
```
- `-D이름=값`은 소스 맨 위에 `#define 이름 값`을 넣은 것과 동일 — 소스를 건드리지 않고 빌드 시점에 분기를 바꿀 수 있다

### macOS `leaks`로 실측
```bash
leaks -atExit -- ./leaks_raw
leaks -atExit -- ./leaks_smart
```
- `leaks`는 실행 중인(또는 종료 시점의) 프로세스 힙을 스캔해 **아무 곳에서도 참조하지 않는 malloc 블록**을 찾아준다
- `-atExit`이 필요한 이유: 프로그램이 바로 종료해버리면 스캔할 시점이 없다 — 종료 직전에 스냅샷을 뜨도록 자식 프로세스로 감싸 실행해준다

| | malloc된 블록 | leaks | 누수 바이트 |
|---|---|---|---|
| raw (`new`만, `delete` 없음) | 1184 nodes / 61 KB | **999 leaks** | **31968 bytes** |
| smart (`make_unique`) | 184 nodes / 30 KB | **0 leaks** | **0 bytes** |

```text
# raw
Process 14238: 1184 nodes malloced for 61 KB
Process 14238: 999 leaks for 31968 total leaked bytes.

# smart
Process 14232: 184 nodes malloced for 30 KB
Process 14232: 0 leaks for 0 total leaked bytes.
```

### 왜 결과가 갈리는가
```cpp
// raw — m이 스코프를 벗어나도 힙 블록은 안 지워짐
Motor* m = new Motor(i);
m->setSpeed(i);
// } 루프 끝, m(포인터 변수)만 사라지고 가리키던 메모리는 그대로 남음 → 누수

// smart — m(unique_ptr)이 스코프를 벗어나는 순간 소멸자가 delete를 대신 호출
auto m = std::make_unique<Motor>(i);
m->setSpeed(i);
// } 루프 끝, unique_ptr 소멸자가 자동으로 delete → 누수 불가능
```
- raw 버전은 "잊지 않고 `delete` 호출"이라는 사람의 기억력에 의존한다. 이 코드처럼 애초에 `delete`를 안 썼다면 컴파일러도 잡아주지 않는다
- smart 버전은 `unique_ptr` 소멸자가 스코프 탈출 시점에 반드시 `delete`를 호출하는 RAII 덕분에, 실수로 안 지우는 경로 자체가 존재하지 않는다

---

## 2. ROS2 노드 실행 모델 — 콜백, Executor와 spin

### 계산 그래프 — 노드와 토픽
ROS2 시스템은 **노드(node)**들이 **토픽(topic)**으로 연결된 그래프다.
- 노드는 하나의 일을 하는 독립 프로세스(카메라 드라이버, 인지, 제어…), 토픽은 이름 붙은 단방향 메시지 흐름
- `camera_node`가 `/image`에 발행(publish)하면 `perception_node`가 구독(subscribe)해서 받는 식

| 용어 | 의미 | 비유 |
|---|---|---|
| Node | 하나의 일을 하는 실행 단위(프로세스) | 부서 |
| Topic | 이름 붙은 단방향 메시지 흐름 | 사내 공지 채널 |
| Message | 토픽으로 오가는 데이터의 형식 | 정해진 양식의 문서 |
| Pub/Sub | 토픽에 쓰기 / 토픽에서 읽기 | 게시 / 구독 |

이 구조의 힘은 **느슨한 결합**이다.
- 발행자는 구독자가 몇 명인지, 누군지 모른다 — 구독자가 늘어도 카메라 노드 코드는 그대로
- 노드는 독립 프로세스라서 인지 노드가 죽어도 제어·안전 노드는 계속 돈다
- 언어·위치 무관 — C++ 노드와 Python 노드가 같은 토픽으로 대화하고, 다른 컴퓨터의 노드와도 같은 방식으로 통신

```bash
ros2 node list                 # 실행 중인 노드 목록
ros2 topic list                # 토픽 목록
ros2 topic echo /scan          # 토픽에 흐르는 메시지 실시간 출력
ros2 topic hz /scan            # 발행 주파수 측정
rqt_graph                      # 그래프를 시각적으로 표시
```

### 모든 것은 콜백 — 이벤트 구동 모델
ROS2 노드는 위에서 아래로 순차 실행되는 스크립트가 아니라 **"어떤 일이 생기면 이 함수를 불러라"**를 등록해두는 이벤트 구동 방식이다. 등록할 수 있는 콜백은 세 가지.
- **구독 콜백**: 구독 중인 토픽에 메시지가 도착하면 실행
- **타이머 콜백**: 정해진 주기마다 실행 (예: 20ms마다 제어 명령 발행)
- **서비스 콜백**: 다른 노드가 요청을 보내면 실행

```python
class PerceptionNode(Node):
    def __init__(self):
        super().__init__('perception_node')
        self.sub = self.create_subscription(LaserScan, '/scan', self.on_scan, 10)
        self.pub = self.create_publisher(Obstacles, '/obstacles', 10)
        self.timer = self.create_timer(0.05, self.on_timer)  # 50ms마다 호출

    def on_scan(self, msg):        # 구독 콜백 — 받아서 저장만 (빠르게!)
        self.latest_scan = msg

    def on_timer(self):            # 타이머 콜백 — 무거운 처리는 여기서
        result = detect_obstacles(self.latest_scan)
        self.pub.publish(result)
```
- 구독 콜백은 받아서 저장만 하고 빠르게 끝내고, 무거운 처리는 타이머 콜백에서 자기 주기로 한다 — "빠른 루프는 느린 루프를 기다리지 않는다"는 원칙이 코드로 드러나는 지점
- 노드를 만든다는 건 결국 "어떤 이벤트에 어떤 콜백을 붙일지 등록하는 것" — 이 콜백들을 실제로 누가, 언제, 어떤 순서로 부르는지가 다음 주제

### Executor와 spin — 콜백을 실제로 돌리는 엔진
노드를 만들고 콜백을 등록하기만 하면 아무 일도 일어나지 않는다. 마지막에 반드시 필요한 한 줄.
```python
rclpy.spin(node)   # 이벤트를 기다리며 콜백을 계속 실행 (블로킹)
```
- `spin`은 executor에게 "이제 일을 시작하라"고 말하는 것
- 타이머 만료·토픽 도착·서비스 요청 같은 이벤트가 생기면 그에 붙은 콜백이 대기열에 들어가고, executor는 spin 루프를 돌며 준비된 콜백을 꺼내 실행
- 기본인 **단일 스레드 executor**는 한 번에 하나씩만 실행, **멀티스레드 executor**는 여러 콜백을 병렬로 실행

### 단일 스레드의 함정 — 콜백 블로킹
기본 executor는 단일 스레드라 콜백을 한 번에 하나만 실행한다. 대부분 충분하지만, 한 콜백이 오래 걸리면 다른 콜백이 전부 밀린다.

사고 실험: 인지 콜백이 무거운 이미지 처리로 80ms가 걸리면
- 20ms마다 돌아야 할 제어 타이머 콜백이 그동안 실행되지 못한다
- 제어 주기가 20ms → 80ms로 무너지고 제어가 불안정해진다

이게 **콜백 블로킹**이며, ROS2 초심자가 가장 흔히 겪는 성능 문제다. 증상은 "제어가 갑자기 버벅인다", 원인은 "같은 executor의 어떤 콜백이 오래 잡고 있다".
- 1차 예방책: 콜백은 짧게 유지. 콜백 안에서 `sleep`, 블로킹 I/O, 무한 대기를 하지 않는다. 무거운 일은 잘게 나누거나 별도 스레드/executor로 뺀다

### 멀티스레드 executor와 콜백 그룹
여러 콜백을 동시에 돌리려면 멀티스레드 executor를 쓴다. 무거운 인지 콜백이 도는 동안에도 제어 타이머 콜백이 다른 스레드에서 제때 실행된다.
```python
from rclpy.executors import MultiThreadedExecutor
executor = MultiThreadedExecutor(num_threads=4)
executor.add_node(node)
executor.spin()
```
하지만 병렬 실행에는 **콜백 그룹(callback group)**이라는 제어 장치가 필요하다.
- **MutuallyExclusive(상호 배타)**: 이 그룹의 콜백들은 서로 동시에 실행되지 않는다 — 같은 데이터를 만지는 콜백을 묶어 경쟁 상태(race condition)를 막는다
- **Reentrant(재진입)**: 자유롭게 병렬 실행 — 서로 독립적인 콜백에 쓴다

제어 콜백과 인지 콜백을 다른 그룹에 두면 병렬로 돌아 제어가 인지에 막히지 않는다. 반대로 같은 상태 변수를 쓰는 콜백 둘은 같은 상호 배타 그룹에 묶어 동시 접근을 막는다.

---

## 3. TurtleBot3 Gazebo 시뮬레이션으로 SLAM부터 Navigation까지

```
Gazebo 월드 실행 → (SLAM으로 지도 생성) → Nav2로 그 지도 위에서 자율주행
```
시뮬레이션은 Remote PC에서 실행하며, 사전에 `turtlebot3`/`turtlebot3_msgs`와 PC Setup으로 필수 패키지가 설치돼 있어야 함.

### 0. 시뮬레이션 패키지 설치 (ROS2 Humble/Jazzy)
```bash
cd ~/turtlebot3_ws/src/
git clone -b humble https://github.com/ROBOTIS-GIT/turtlebot3_simulations.git
cd ~/turtlebot3_ws
colcon build --symlink-install
```

### 1. Gazebo 월드 실행
로봇 모델(`burger`/`waffle`/`waffle_pi`)을 환경변수로 지정하고, 세 가지 월드 중 하나를 실행한다.
```bash
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_gazebo empty_world.launch.py       # 빈 공간

export TURTLEBOT3_MODEL=waffle
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py  # 로봇 월드 (지도 작성에 최적화)

export TURTLEBOT3_MODEL=waffle_pi
ros2 launch turtlebot3_gazebo turtlebot3_house.launch.py  # 집 환경
```
- 처음 실행할 때는 맵 리소스 다운로드로 네트워크 상태에 따라 몇 분 걸릴 수 있음
- 조작: 새 터미널에서 `ros2 run turtlebot3_teleop teleop_keyboard`(키보드 원격조종) 또는 `ros2 run turtlebot3_gazebo turtlebot3_drive`(자율 충돌 회피), 시각화는 `ros2 launch turtlebot3_bringup rviz2.launch.py`
- **Fake node 시뮬레이션**은 센서 없이 모델·움직임 테스트용, **Gazebo 시뮬레이션**은 IMU·LDS·카메라 등 센서를 지원해서 SLAM·Navigation에 필요한 건 이쪽

### 2. SLAM으로 지도 만들기
Gazebo 월드가 떠 있는 상태에서, 새 터미널에 SLAM 노드를 띄운다. 지도 작성에는 `turtlebot3_world`가 최적화되어 있음.
```bash
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_cartographer cartographer.launch.py use_sim_time:=True
```
- Cartographer가 기본 SLAM 방식
- 또 다른 터미널에서 `ros2 run turtlebot3_teleop teleop_keyboard`(w/x: 직진 속도, a/d: 회전 속도, 스페이스/s: 정지)로 로봇을 돌아다니게 하면서 지도를 채운다
- 다 돌았으면 새 터미널에서 지도 저장
```bash
ros2 run nav2_map_server map_saver_cli -f ~/map
```
- `~/map.pgm`(이미지)과 `~/map.yaml`(메타데이터)이 생성됨 — 다음 단계인 Navigation이 이 지도를 그대로 사용

### 3. Navigation(Nav2)으로 자율주행
지도가 준비됐다면 이전 프로세스를 정리하고 다시 Gazebo 월드부터 띄운다.
```bash
# 1) 월드 재실행
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py

# 2) 새 터미널 — 저장해둔 지도로 Nav2 실행
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_navigation2 navigation2.launch.py use_sim_time:=True map:=$HOME/map.yaml
```

**3-1. 초기 위치 추정 (Initial Pose Estimation)** — AMCL 파라미터를 초기화하는 필수 과정
1. RViz2에서 `2D Pose Estimate` 클릭
2. 로봇의 실제 위치를 지도에 클릭한 채로, 로봇이 향한 방향으로 녹색 화살표를 드래그
3. LDS 센서 데이터가 지도와 겹칠 때까지 반복
4. 정밀 보정이 필요하면 `teleop_keyboard`로 로봇을 앞뒤로 살짝 움직여 추정 위치를 수렴시킨 뒤 `Ctrl+C`로 종료

**3-2. 목표 지점 설정 (Navigation Goal)**
1. RViz2에서 `Navigation2 Goal` 클릭
2. 목표 지점을 지도에 클릭하고, 원하는 방향으로 녹색 화살표를 드래그 — 화살표 시작점이 목적지 (x, y), 화살표 방향이 목표 각도 θ
3. 설정하는 즉시 로봇이 이동을 시작

- 로봇은 글로벌 경로 계획기로 전체 경로를 세우고, 장애물을 만나면 로컬 경로 계획기로 우회한다. 목표에 도달할 수 없으면 현재 위치를 목표로 재설정해 정지시킬 수 있음

### 명령어 요약
| 단계 | 명령어 |
|---|---|
| 모델 지정 | `export TURTLEBOT3_MODEL=burger` |
| Gazebo 월드 | `ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py` |
| SLAM | `ros2 launch turtlebot3_cartographer cartographer.launch.py use_sim_time:=True` |
| 지도 저장 | `ros2 run nav2_map_server map_saver_cli -f ~/map` |
| Nav2 실행 | `ros2 launch turtlebot3_navigation2 navigation2.launch.py use_sim_time:=True map:=$HOME/map.yaml` |
| 원격조종 | `ros2 run turtlebot3_teleop teleop_keyboard` |

---

## 정리
- **스마트 포인터**: `leaks -atExit`으로 raw 999건 누수 vs `make_unique` 0건 누수를 직접 실측 — RAII는 "잊지 않고 지우기"가 아니라 "잊을 수 있는 경로 자체를 없애는 것"
- **ROS2 실행 모델**: 노드는 콜백을 등록만 하고, executor가 spin 루프로 그 콜백을 실제로 돌린다. 단일 스레드에서는 긴 콜백 하나가 다른 콜백을 다 막는 콜백 블로킹이 흔한 함정 — 멀티스레드 executor + 콜백 그룹으로 해결
- **TurtleBot3**: Gazebo(센서 지원) → Cartographer로 SLAM 지도 생성·저장 → Nav2가 그 지도 위에서 AMCL로 초기 위치를 잡고 글로벌/로컬 플래너로 자율주행하는 흐름

> 세 주제의 공통점은 결국 "수동으로 챙겨야 할 일을 프레임워크/도구가 대신 해주는 지점을 아는 것" — 스마트 포인터는 해제를, executor는 콜백 스케줄링을, Nav2는 경로 계획을 대신 맡아준다.

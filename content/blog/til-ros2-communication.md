---
title: "[TIL] ROS2 통신 총정리 — Executor·Topic/Service/Action·커스텀 인터페이스·QoS·TF2"
date: 2026-08-12T21:00:00+09:00
description: "ROS2 콜백/executor/spin부터 Topic pub-sub, Service·Action·Parameter, 커스텀 인터페이스(.msg/.srv/.action), QoS·DDS, TF2 좌표 변환까지 — 직접 작성한 pub/sub 노드와 URDF 예제 포함"
image: "images/post/post-1.jpg"
categories: ["TIL"]
tags: ["ros2", "topic", "service", "action", "qos", "dds", "tf2", "pubsub"]
type: "post"
nextp: ""
prevp: ""
---

## 1. 노드 실행 모델 — 콜백, Executor와 spin

ROS2 시스템은 **노드(node)**들이 **토픽(topic)**으로 연결된 계산 그래프다. 노드는 하나의 일을 하는 독립 프로세스, 토픽은 이름 붙은 단방향 메시지 흐름이다. 발행자는 구독자가 누군지 모르고(느슨한 결합), 노드는 독립 프로세스라 하나가 죽어도 나머지는 계속 돈다.

노드의 동작은 전부 **콜백**(구독 콜백·타이머 콜백·서비스 콜백)으로 등록된다. 순차 실행 스크립트가 아니라 "이 이벤트가 오면 이 함수를 불러라"를 등록해두는 이벤트 구동 모델이다.

```python
class PerceptionNode(Node):
    def __init__(self):
        super().__init__('perception_node')
        self.sub = self.create_subscription(LaserScan, '/scan', self.on_scan, 10)
        self.timer = self.create_timer(0.05, self.on_timer)  # 50ms마다

    def on_scan(self, msg):        # 구독 콜백 — 받아서 저장만 (빠르게!)
        self.latest_scan = msg

    def on_timer(self):            # 타이머 콜백 — 무거운 처리는 여기서
        result = detect_obstacles(self.latest_scan)
```

콜백을 등록하는 것만으로는 아무 일도 안 일어난다. 마지막에 `rclpy.spin(node)`가 필요한데, 이건 **executor**에게 "이제 콜백을 실행하라"고 말하는 것이다. 기본 **단일 스레드 executor**는 콜백을 한 번에 하나씩만 실행하기 때문에, 콜백 하나가 오래 걸리면(예: 인지 콜백 80ms) 20ms마다 돌아야 할 제어 타이머 콜백이 밀린다 — 이게 **콜백 블로킹**, ROS2 초심자가 가장 흔히 겪는 성능 문제다. 해법은 콜백을 짧게 유지하거나(`sleep`/블로킹 I/O 금지), **멀티스레드 executor + 콜백 그룹**(MutuallyExclusive로 경쟁 상태 방지, Reentrant로 독립 콜백 병렬화)을 쓰는 것.

## 2. Topic 통신과 노드 작성 — pub/sub, rclpy·rclcpp

Topic은 **단방향·비동기·다대다** 통신이다. 발행자가 흘려보내면 구독한 모든 노드가 각자 받는다. 토픽에는 메시지 큐가 있고, **큐 깊이**만큼 최근 메시지를 버퍼링한다 — 센서 스트림처럼 빠른 데이터는 깊이를 작게(늦은 데이터보다 최신값이 낫다), 놓치면 안 되는 명령은 충분히 크게(신뢰성 자체는 13강 QoS가 더 본격적으로 다룸).

| 패키지 | 대표 메시지 | 용도 |
|---|---|---|
| std_msgs | String, Float64, Bool | 단순 값 |
| geometry_msgs | Twist, Pose, Point | 속도·자세·위치 |
| sensor_msgs | LaserScan, Image, Imu | 센서 데이터 |
| nav_msgs | Odometry, Path | 주행 정보 |

`geometry_msgs/Twist`를 `/cmd_vel`에 발행하는 게 로봇을 움직이는 표준 방식이다 (`linear.x` 전진, `angular.z` 회전).

### 직접 작성한 실습 코드
Node 상속 → 발행자/구독자 생성 → 콜백 정의 → spin, 이 4단계 구조를 그대로 따라 20Hz로 속도 명령을 발행/구독하는 노드를 작성했다.

```python
# pub_sub/publisher.py
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist

class VelocityPublisher(Node):
    def __init__(self):
        super().__init__('velocity_publisher')      # 노드 이름
        # 발행자 생성: (메시지타입, 토픽명, 큐깊이)
        self.pub = self.create_publisher(Twist, '/cmd_vel', 10)
        # 0.05초(20Hz)마다 tick 호출
        self.timer = self.create_timer(0.05, self.tick)
        self.get_logger().info('발행 시작')

    def tick(self):
        msg = Twist()
        msg.linear.x = 0.2      # 0.2 m/s 전진
        msg.angular.z = 0.1     # 약간 회전
        self.pub.publish(msg)

def main():
    rclpy.init()
    node = VelocityPublisher()
    rclpy.spin(node)            # 콜백이 돌기 시작
    rclpy.shutdown()
```

```python
# pub_sub/subscriber.py
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import Twist

class VelocitySubscriber(Node):
    def __init__(self):
        super().__init__('velocity_subscriber')
        # 구독자 생성: (메시지타입, 토픽명, 콜백, 큐깊이)
        self.sub = self.create_subscription(
            Twist, '/cmd_vel', self.on_cmd, 10)

    def on_cmd(self, msg):       # 메시지가 올 때마다 호출
        self.get_logger().info(
            f'받음: 전진 {msg.linear.x:.2f} m/s, 회전 {msg.angular.z:.2f} rad/s')

def main():
    rclpy.init()
    rclpy.spin(VelocitySubscriber())
    rclpy.shutdown()
```
- `create_publisher(Twist, '/cmd_vel', 10)`처럼 (메시지 타입, 토픽명, 큐 깊이) 세 인자가 한 세트라는 걸 코드로 확인
- 구독자는 콜백(`on_cmd`)만 등록하고 실제 호출은 `spin` 중 executor가 대신 해준다 — 9강에서 배운 executor/콜백 구조가 그대로 적용됨
- 발행자를 껐다 켜도 구독자가 죽지 않는 것, 구독자를 여러 개 띄워도 각자 메시지를 받는 것이 토픽의 느슨한 결합·다대다 특성

rclcpp(C++)도 구조는 동일하다 — ① `rclcpp::Node` 상속 → ② `create_publisher<Twist>`/`create_subscription` → ③ 콜백 정의 → ④ `rclcpp::spin`. 8강의 클래스 상속·템플릿·`shared_ptr`(`make_shared<MyNode>()`)이 그대로 쓰인다. 인지·프로토타이핑은 rclpy, 고빈도 제어·성능 노드는 rclcpp — 언어가 달라도 같은 토픽으로 자유롭게 통신한다.

```bash
ros2 run <pkg> velocity_publisher
ros2 topic echo /cmd_vel      # 메시지 실시간 확인
ros2 topic hz /cmd_vel        # 20Hz로 발행되는지 확인
ros2 topic pub /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}}"  # 노드 없이 CLI로 직접 발행
```

## 3. Service·Action·Parameter 통신

Topic만으로 로봇의 모든 상호작용을 표현할 순 없다. 상황별로 맞는 패턴이 따로 있다.

| 질문 | 답 → 패턴 |
|---|---|
| 결과를 돌려받아야 하나? | 아니오(그냥 보냄) → Topic |
| 결과를 돌려받되, 즉시 끝나나? | 예 → Service |
| 오래 걸리고, 진행상황·취소가 필요한가? | 예 → Action |
| 노드의 설정값을 다루나? | 예 → Parameter |

### Service — 요청과 응답
짧고 확실히 끝나는 일(지도 저장, 순기구학 계산 등)에 쓴다. `.srv`는 `---`로 요청/응답을 나눈다.
```
# AddTwoInts.srv
int64 a
int64 b
---
int64 sum
```
```python
class AddServer(Node):
    def __init__(self):
        super().__init__('add_server')
        self.srv = self.create_service(AddTwoInts, 'add_two_ints', self.on_request)

    def on_request(self, request, response):
        response.sum = request.a + request.b
        return response
```
**함정**: 클라이언트가 응답을 콜백 안에서 동기로 기다리면 안 된다. 단일 스레드 executor에서 콜백이 응답을 기다리면, 그 응답을 처리할 spin 자체가 이미 그 콜백에 붙잡혀 있어 영원히 서로를 기다리는 데드락이 된다(9강 "콜백은 짧게, 콜백 안에서 대기 금지"의 구체적 사례). 그래서 `call_async` + `spin_until_future_complete`를 쓴다.


```python
future = client.call_async(request)
rclpy.spin_until_future_complete(node, future)
result = future.result()
```
```bash
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 3, b: 4}"
```

### Action — 장기 작업 + 피드백 + 취소
Service로 오래 걸리는 작업(주행, 조작)을 처리하면 그동안 정보도 없이 하염없이 기다려야 하고 중간에 멈출 수도 없다. Action은 **Service(목표/결과) + Topic(피드백 스트림)**을 합친 구조다.
```
# NavigateToPose.action (개념 예시)
geometry_msgs/Pose target      # Goal
---
bool success                   # Result
---
float32 distance_remaining     # Feedback (반복 전송)
```
① 클라이언트가 Goal 전송 → ② 서버가 수락/거부 → ③ 수행 중 Feedback 반복 전송 → ④ 완료 시 Result 반환, 언제든 취소 가능. Nav2 자율주행이 대표 사례 — "현관으로 가"를 보내면 남은 거리를 피드백하고, 취소 요청이 오면 안전하게 정지한다. 로봇은 물리적으로 움직이는 중이므로 취소는 Topic·Service엔 없는 Action만의 안전 설계 핵심 기능이다.

### Parameter — 런타임 설정값
최대 속도, PID 게인처럼 조정 가능한 값을 코드에 하드코딩하지 않고 파라미터로 다루면 재빌드 없이 값을 바꿀 수 있다.
```python
class ControlNode(Node):
    def __init__(self):
        super().__init__('control_node')
        self.declare_parameter('max_speed', 0.8)
        self.declare_parameter('kp', 1.2)

    def tick(self):
        max_speed = self.get_parameter('max_speed').value
```
```bash
ros2 param list
ros2 param get /control_node max_speed
ros2 param set /control_node max_speed 0.5   # 실행 중 변경!
```

## 4. 커스텀 인터페이스 정의 — .msg·.srv·.action

표준 메시지로 부족할 때(예: "장애물 목록"처럼 위치·크기·신뢰도를 갖고 개수가 매번 다른 데이터) 커스텀 인터페이스를 정의한다.

| 분류 | 타입 | 예 |
|---|---|---|
| 정수 | int8/16/32/64, uint8/… | `int32 count` |
| 실수 | float32, float64 | `float32 distance` |
| 논리·문자 | bool, string | `bool is_valid` |
| 배열 | 타입\[](가변), 타입\[N](고정) | `float32[] ranges` |
| 중첩 | 다른 메시지 타입 | `geometry_msgs/Pose pose` |

```
# Obstacle.msg
geometry_msgs/Point position     # 위치 (표준 메시지 중첩)
float32 radius
float32 confidence

# ObstacleArray.msg
std_msgs/Header header           # 타임스탬프·좌표계 — 시간 동기화와 TF2 변환에 필요
Obstacle[] obstacles
int32 count
```
좋은 설계 원칙: **의미 단위로 묶기**(x/y/z 대신 `Point`), **표준을 재사용**(위치엔 `geometry_msgs/Point`), **단위를 명시**(주석으로 `# meters`), **확장 여지**(필드는 지우기보다 더하기).

`.msg`/`.srv`/`.action`은 `colcon build` 시 **rosidl** 생성기가 읽어 Python 클래스와 C++ 헤더를 동시에 만든다 — 그래서 rclpy·rclcpp 노드가 같은 타입으로 통신할 수 있다(파스칼케이스 `ObstacleArray` → C++에서는 스네이크케이스 `obstacle_array.hpp`로 자동 변환).

인터페이스는 노드 코드와 분리된 **전용 패키지**로 만든다 — 여러 노드 패키지가 하나의 인터페이스 패키지에만 의존하게 해서 불필요한 결합을 없앤다("많은 것이 의존하는 것일수록 가볍고 안정적이어야 한다").
```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/Obstacle.msg"
  "msg/ObstacleArray.msg"
  "srv/SetGain.srv"
  DEPENDENCIES geometry_msgs std_msgs
)
```
```bash
colcon build --packages-select my_interfaces
ros2 interface show my_interfaces/msg/ObstacleArray
```

## 5. QoS 정책과 DDS 미들웨어

ROS2의 모든 통신은 **DDS(Data Distribution Service)** 미들웨어 위에서 일어난다. 가장 중요한 특징은 **분산 디스커버리** — 중앙 마스터가 없어서(ROS1의 `roscore`와 달리) 한 노드가 죽어도 나머지는 계속 통신한다. 단일 장애점이 없다는 게 로봇 안전성에 중요하다.

QoS(Quality of Service)는 이 통신 품질을 데이터마다 맞추는 설정이다. 핵심 세 가지:
- **Reliability**: `Reliable`(유실 시 재전송 — 명령·설정처럼 놓치면 안 되는 데이터) vs `Best-Effort`(유실 허용, 최신 우선 — 카메라·LiDAR처럼 늦은 옛 프레임보다 최신이 중요한 데이터)
- **Durability**: `Volatile`(접속 후 오는 메시지만) vs `Transient Local`(발행자가 마지막 메시지를 보관해서 늦게 접속한 구독자에게도 전달 — `/map`, `/robot_description` 같이 한 번 발행되고 잘 안 바뀌지만 나중에 접속한 노드도 알아야 하는 데이터)
- **History**: `Keep Last(depth N)`(최근 N개, 10강의 큐 깊이) vs `Keep All`

| 프로파일 | 구성 | 용도 |
|---|---|---|
| Default | Reliable, Volatile, KeepLast(10) | 일반 통신(명령 등) |
| Sensor Data | Best-Effort, Volatile, KeepLast(5) | 카메라·LiDAR 스트림 |
| Services | Reliable | 서비스 통신 |
| Parameters | Reliable | 파라미터 |

```python
from rclpy.qos import qos_profile_sensor_data
self.sub = self.create_subscription(
    LaserScan, '/scan', self.on_scan, qos_profile_sensor_data)
```

**호환성 규칙**: 구독자의 요구가 발행자보다 엄격하면 연결되지 않는다 — 발행자 Best-Effort + 구독자 Reliable은 연결 안 됨, 반대는 연결됨. "토픽 이름·메시지 타입은 맞는데 데이터가 안 온다"의 절반 이상이 QoS 불일치다.
```bash
ros2 topic info /scan --verbose   # 발행자·구독자의 QoS를 상세 표시해서 진단
```

## 6. TF2 좌표 변환

LiDAR가 "전방 2.3m에 장애물"이라 해도 그 "전방"은 LiDAR 기준이다. 로봇 몸체 기준으로는? 전역 지도 기준으로는? 센서·부품마다 좌표계(frame)가 다르고, 로봇이 움직이면 그 관계도 매 순간 바뀐다. 이 관계를 관리·변환해주는 게 **TF2**다.

### 표준 좌표계 트리
```
map → odom → base_link → 센서 링크들(lidar_link, camera_link, imu_link …)
```
- `map`: 방/건물에 고정된 전역 기준(SLAM/AMCL로 보정, 불연속이나 정확)
- `odom`: 출발점 기준 엔코더 추정 위치(부드럽지만 시간이 지나며 오차 누적)
- `base_link`: 로봇 몸체 기준점, 모든 센서가 이 기준으로 고정 장착
- 제어는 부드러운 `odom`을, 경로 계획은 정확한 `map`을 쓴다 — 2강의 "빠른 것과 정확한 것의 분리"가 좌표계 설계에도 나타난다

변환은 **정적**(고정 장착, 예: `base_link → lidar_link` — 한 번만 발행)과 **동적**(계속 바뀜, 예: `odom → base_link` — 로봇이 움직이니 매 순간 갱신)으로 나뉜다.

### 직접 작성한 URDF로 확인한 프레임 트리
```xml
<!-- tf2/robot.urdf -->
<link name="base_footprint"/>

<link name="base_link">
  <visual><geometry><box size="1 1 0.5"/></geometry></visual>
</link>

<link name="lidar">
  <visual><geometry><cylinder radius="0.05" length="0.1"/></geometry></visual>
</link>

<link name="left_wheel">
  <visual><geometry><cylinder radius="0.2" length="0.05"/></geometry></visual>
</link>

<link name="right_wheel">
  <visual><geometry><cylinder radius="0.2" length="0.05"/></geometry></visual>
</link>

<joint name="base_footprint_joint" type="fixed">
  <parent link="base_footprint"/><child link="base_link"/>
  <origin xyz="0 0 0.4" rpy="0 0 0"/>
</joint>

<joint name="lidar_joint" type="fixed">
  <parent link="base_link"/><child link="lidar"/>
  <origin xyz="0 0 0.3" rpy="0 0 0"/>
</joint>

<joint name="left_wheel_joint" type="continuous">
  <parent link="base_link"/><child link="left_wheel"/>
  <origin xyz="-0.525 0.3 -0.2" rpy="0 0 0"/><axis xyz="1 0 0"/>
</joint>

<joint name="right_wheel_joint" type="continuous">
  <parent link="base_link"/><child link="right_wheel"/>
  <origin xyz="0.525 0.3 -0.2" rpy="0 0 0"/><axis xyz="1 0 0"/>
</joint>
```
이 URDF가 곧 TF2 트리의 뼈대다.
- `base_footprint → base_link → lidar`는 전부 `type="fixed"` — 센서가 몸체에 나사로 박힌 것과 같아서, TF2에서는 `StaticTransformBroadcaster`로 한 번만 발행하면 되는 **정적 변환**에 대응된다
- `left_wheel_joint`/`right_wheel_joint`는 `type="continuous"`(회전 관절) — 바퀴는 계속 회전하므로 실제 로봇에서는 조인트 상태(엔코더)에 따라 매 순간 갱신되는 **동적 변환**에 대응된다
- 링크(`<link>`)의 `<origin>`은 대부분 `0 0 0`으로 두고, 실제 위치·자세 조정은 조인트(`<joint>`)의 `<origin>`에서 하는 걸 확인했다 — 부모-자식 관계(트리 구조)로 위치를 표현하는 TF2/URDF 공통 설계와 맞아떨어지는 부분

### broadcast와 lookup
각 노드가 자기가 아는 변환을 `/tf`(동적)·`/tf_static`(정적)에 **broadcast**하고, 필요한 노드가 "A에서 B로 가는 변환이 뭐야?"를 **lookup**하면 TF2가 트리를 따라 중간 변환들을 연쇄해서 계산해준다.
```python
from tf2_ros import TransformBroadcaster
from geometry_msgs.msg import TransformStamped

t = TransformStamped()
t.header.stamp = self.get_clock().now().to_msg()
t.header.frame_id = 'odom'          # 부모 프레임
t.child_frame_id = 'base_link'      # 자식 프레임
t.transform.translation.x = msg.x
t.transform.rotation.z = math.sin(msg.theta / 2.0)   # 회전은 쿼터니언으로
t.transform.rotation.w = math.cos(msg.theta / 2.0)
self.br.sendTransform(t)
```
```python
transform = tf_buffer.lookup_transform('map', 'lidar_link', rclpy.time.Time())
point_in_map = do_transform_point(point_in_lidar, transform)
```
- LiDAR가 본 장애물을 지도 좌표로 옮기려면 `lidar_link → base_link → odom → map`을 차례로 연쇄해야 하는데, `lookup_transform('map', 'lidar_link')` 한 줄이 이 중간 단계를 자동으로 처리해준다 — 프레임이 몇 개든 "어디서 어디로"만 말하면 됨

```bash
ros2 run tf2_tools view_frames             # 현재 TF 트리를 PDF로 그려줌
ros2 run tf2_ros tf2_echo map base_link    # 두 프레임 사이 변환을 실시간 출력
```

## 정리
- executor가 spin 루프로 콜백을 실행. 단일 스레드에선 긴 콜백 하나가 다른 콜백을 막는 콜백 블로킹이 흔한 함정 — 멀티스레드 executor + 콜백 그룹으로 해결
- Topic은 단방향·비동기·다대다. `create_publisher`/`create_subscription`으로 실제 VelocityPublisher/Subscriber를 만들어 20Hz 통신을 확인했고, rclpy·rclcpp는 언어만 다를 뿐 구조는 동일
- Topic(스트림)·Service(요청/응답)·Action(장기작업+피드백+취소)·Parameter(설정)를 상황에 맞게 고르는 게 설계의 첫 단추. Service 응답을 콜백 안에서 동기 대기하면 데드락
- 표준 메시지로 부족하면 `.msg`/`.srv`/`.action`으로 커스텀 인터페이스를 정의하고, rosidl이 Python·C++ 코드로 자동 생성. 인터페이스는 별도 패키지로 분리
- DDS의 분산 디스커버리 덕에 중앙 마스터가 없고, QoS(Reliability·Durability·History)로 통신 품질을 데이터에 맞춘다. "토픽은 맞는데 안 온다"는 QoS 불일치가 주범
- TF2가 `map → odom → base_link → 센서` 트리로 좌표계 관계를 관리. 내가 만든 URDF의 `fixed` 조인트는 정적 변환, `continuous` 조인트는 동적 변환에 대응된다는 걸 직접 매핑해봄

> 결국 "ROS2에서 데이터가 어떤 모양(Topic/Service/Action/Parameter)으로, 어떤 품질(QoS)로, 어떤 기준(TF2 좌표계)으로 흘러야 하는가"를 정하는 방법을 순서대로 쌓아온 것

---
title: "[TIL] rclcpp 구독 콜백 — 람다를 쓰는 진짜 이유, 그리고 const가 붙는 자리"
date: 2026-09-04
description: "구독 콜백은 std::bind보다 람다가 낫고, 메시지는 SharedPtr이 아니라 ConstSharedPtr로 받아야 한다. const가 붙는 자리가 두 군데라서 헷갈리는데, const를 어디에 붙였느냐가 라이브러리가 내 대신 최적화를 할 수 있느냐를 정한다."
image: "images/post/til-2026-09-04-banner.png"
categories: ["TIL"]
tags: ["ros2", "cpp"]
type: "post"
nextp: ""
prevp: "til-axis-angle"
---

```cpp
DistanceListener() : Node("turtle_distance_listener") {
    distance_sub_ = this->create_subscription<std_msgs::msg::Float32>(
        "/turtle_distance", 10,
        [this](std_msgs::msg::Float32::ConstSharedPtr msg) {
            RCLCPP_INFO(this->get_logger(), "거리 %.3f m", msg->data);
        }
    );
    RCLCPP_INFO(this->get_logger(), "구독 시작 (C++) — /turtle_distance");
}
```

## 1. 콜백을 넘기는 세 가지 방법

`create_subscription`의 세 번째 인자는 **호출 가능한 무엇이든** 받는다. 실제로 쓰는 건 세 가지다.

| 방식 | 생김새 | 노드 멤버 접근 |
| --- | --- | --- |
| 자유 함수 | `void cb(Msg::ConstSharedPtr)` | 불가 (전역이나 인자로 넘겨야 함) |
| `std::bind` | `std::bind(&Node::cb, this, _1)` | 가능 |
| 람다 | `[this](Msg::ConstSharedPtr msg) { ... }` | 가능 (캡처) |

```cpp
// bind — placeholders 개수를 콜백 인자 수에 맞춰야 한다
using std::placeholders::_1;
sub_ = create_subscription<Float32>(
    "/turtle_distance", 10,
    std::bind(&DistanceListener::on_msg, this, _1));

// 람다 — 시그니처가 호출부에 그대로 보인다
sub_ = create_subscription<Float32>(
    "/turtle_distance", 10,
    [this](Float32::ConstSharedPtr msg) { ... });
```

## 2. `SharedPtr`은 deprecated다

튜토리얼에서 흔히 보는 `std_msgs::msg::Float32::SharedPtr`은 더 이상 권장되지 않는다. `rclcpp` 이슈 [#1619](https://github.com/ros2/rclcpp/issues/1619)에서 `void (std::shared_ptr<MessageT>)` 계열 시그니처를 deprecate하기로 정했고, 대체 시그니처가 `void (std::shared_ptr<const MessageT>)` — 즉 `ConstSharedPtr`이다. Galactic API freeze 직전이라 미뤘다가 그 뒤에 실제로 적용됐다.

| 별칭 | 실제 타입 | 상태 |
| --- | --- | --- |
| `Msg::SharedPtr` | `std::shared_ptr<Msg>` | deprecated |
| `Msg::ConstSharedPtr` | `std::shared_ptr<const Msg>` | 권장 |

> **Problem** — `SharedPtr`로 받아도 코드는 잘 돌아간다. 그런데 왜 굳이 바꾸라는 건지, 그냥 스타일 규칙인 줄 알았다.
>
> **Why** — **복사 횟수 문제**다. 콜백이 non-const 포인터로 받으면 구독자가 메시지를 고칠 수 있다는 뜻이고, 그러면 같은 프로세스 안에서 구독자가 `N`개일 때 서로 간섭하지 않도록 **각자에게 복사본을 줘야** 한다. `const`로 받으면 아무도 못 고치니까 버퍼 하나를 `N`개가 같이 봐도 된다. 인트라프로세스 통신에서 복사가 0번이냐 `N`번이냐가 여기서 갈린다.
>
> **How** — 콜백 인자를 `Msg::ConstSharedPtr`로 바꾼다. 그것뿐이다.
>
> **What** — `const`가 **의도 표현이 아니라 최적화 조건**이었다. 라이브러리가 복사를 생략해도 되는지를 판단하는 근거가 타입에 붙어 있는 것이고, 내가 `const`를 안 붙이면 rclcpp는 안전한 쪽 — 복사하는 쪽 — 을 고를 수밖에 없다.

## 3. `const`가 붙는 자리가 두 군데다

여기서 제일 헷갈렸다. `const`를 포인터에 붙이는 것과 가리키는 대상에 붙이는 게 완전히 다른 얘기다.

| 선언 | 못 바꾸는 것 | 바꿀 수 있는 것 |
| --- | --- | --- |
| `const std::shared_ptr<Msg> &` | 포인터 (재대입 불가) | **메시지 내용** |
| `std::shared_ptr<const Msg>` | **메시지 내용** | 포인터 |

`ConstSharedPtr`은 **아래쪽**이다. 나는 위쪽이 const 버전인 줄 알았다. 위쪽은 참조자를 const로 받은 것뿐이라 `msg->data = 0;`이 그대로 컴파일된다 — 겉보기엔 `const`가 붙어 있는데 정작 메시지는 무방비다.

```cpp
void a(const Float32::SharedPtr & msg) { msg->data = 0; }  // 컴파일된다
void b(Float32::ConstSharedPtr msg)    { msg->data = 0; }  // 에러
```

한 가지 더. `ConstSharedPtr`은 값으로 받아도 된다. `shared_ptr` 복사는 참조 카운트 증가 한 번이고, 어차피 rclcpp가 콜백에 넘길 때 이미 하나 만들어서 준다.

## 정리

1. `Msg::SharedPtr` 콜백 시그니처는 **deprecated**다. 대체는 `Msg::ConstSharedPtr`이고, rclcpp 이슈 #1619에서 정해졌다.
2. `ConstSharedPtr`을 쓰라는 건 스타일이 아니라 **복사 횟수** 얘기다. 구독자가 메시지를 못 고친다는 게 타입으로 보장돼야 인트라프로세스에서 버퍼 하나를 여럿이 공유할 수 있다.
3. `const std::shared_ptr<Msg> &`와 `std::shared_ptr<const Msg>`는 **다르다.** 앞은 포인터가 const라 메시지 내용은 그대로 고쳐진다. `ConstSharedPtr`은 뒤쪽이다.
4. 결국 오늘 배운 건 하나다 — **`const`를 어디에 붙였느냐가 라이브러리가 내 대신 최적화를 할 수 있느냐를 정한다.**

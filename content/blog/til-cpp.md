---
title: "[TIL] C++ 스마트 포인터(unique_ptr)와 STL"
date: 2026-08-10T21:00:00+09:00
image: "images/post/post-5.jpg"
categories: ["TIL"]
tags: ["cpp", "stl", "smart-pointer", "cmake"]
type: "post"
nextp: ""
prevp: ""
---

## 컴파일 vs 인터프리터
- C++은 실행 전에 소스 전체를 기계어로 번역(컴파일), Python은 실행하며 한 줄씩 해석(인터프리트)
- 속도 차이의 근원은 세 가지: **타입 확정 시점**(컴파일 타임 vs 매 연산마다 확인), **메모리 배치**(원시값 vs 참조·타입 정보가 붙은 객체), **최적화 범위**(전역 최적화 가능 vs 한 줄 단위)

## clang++ 빌드 파이프라인
```bash
clang++ -Wall -Wextra -O2 -std=c++17 -o robot main.cpp
#            │        │       └ 언어 표준 지정
#            │        └ 최적화 레벨 2 (배포용)
#            └ 모든 경고 켜기
```
- 코드는 **헤더(.hpp, 선언)**와 **소스(.cpp, 정의)**로 분리 — 구현이 바뀌어도 헤더만 참조하는 파일은 재컴파일 불필요
- 파이프라인: 각 `.cpp` → 목적파일(`.o`)로 개별 컴파일 → 링커가 `.o`들을 묶어 실행파일 생성

```bash
clang++ -c motor.cpp -o motor.o   # -c: 컴파일만, 링크 안 함
clang++ motor.o main.o -o robot   # 링크
```

## CMake — 빌드의 조리법
```cmake
cmake_minimum_required(VERSION 3.16)
project(robot_controller CXX)
set(CMAKE_CXX_STANDARD 17)

add_executable(robot
    src/main.cpp
    src/motor.cpp
)
target_include_directories(robot PRIVATE include)
```

```bash
cmake -B build .
cd build
make       # 바뀐 파일만 다시 컴파일 (증분 빌드)

```

- 핵심 가치: **증분 빌드**(수정된 파일만 재컴파일)와 **의존성 선언**(헤더 경로·링크 설정 자동 처리)
- ROS2의 `colcon build`는 여러 패키지의 CMake를 의존성 순서대로 호출하는 상위 도구 

## 스택 vs 힙
```cpp
void control_loop() {
    double target = 0.5;            // 스택: 함수 끝나면 자동 소멸
    std::vector<double> scan(360);  // 내부 데이터는 힙, vector 객체는 스택
}   // 여기서 모두 자동 정리됨
```
- 기준: 크기가 컴파일 타임에 정해지고 함수 안에서만 쓰면 **스택**(기본 선택), 실행 중 크기가 정해지거나 함수 밖까지 살아야 하면 **힙**
- 원칙: 가능하면 스택. 힙이 필요할 때는 반드시 스마트 포인터로

## raw 포인터의 위험
```cpp
Motor* p = new Motor();
// delete p;  ← 잊으면 메모리 누수

Motor* q = new Motor();
delete q;
q->setSpeed(0.5);  // 이미 해제된 걸 사용 → 댕글링 포인터, 크래시
```
컴파일러가 이런 실수를 잡아주지 않아서, 현대 C++은 `new`/`delete`를 손으로 쓰지 않는 방향으로 진화했다.

## RAII와 스마트 포인터
**RAII**(Resource Acquisition Is Initialization): 자원의 수명을 객체의 수명에 묶는 원칙. 스택 객체는 범위를 벗어나면 반드시 소멸되므로, 그 소멸에 정리를 걸어두면 잊을 수가 없다.

```cpp
#include <memory>

// unique_ptr — 단독 소유 (기본 선택)
auto motor = std::make_unique<Motor>();
motor->setSpeed(0.5);
// 함수 끝나면 자동 delete — 잊을 수가 없음

// shared_ptr — 공동 소유 (여러 곳에서 참조해야 할 때)
auto lidar = std::make_shared<Lidar>();
auto lidar2 = lidar;   // 참조 카운트 2로 증가
// 둘 다 사라지면 그때 delete
```

| | unique_ptr | shared_ptr |
|---|---|---|
| 소유권 | 단독 소유, 복사 불가(이동만 가능) | 공동 소유, 참조 카운트로 관리 |
| 삭제 시점 | 소유자가 사라질 때 | 마지막 참조가 사라질 때 |
| 기본 선택 여부 | 기본값 — 소유자가 하나면 충분 | 여러 주체가 공유해야 할 때만 |

- 선택 기준: **기본은 unique_ptr**, 여러 곳에서 공유해야만 shared_ptr. "그냥 다 shared_ptr"는 참조 카운트 비용과 소유권 모호함을 부르므로 지양
- ROS2에서 노드·발행자·구독자·메시지는 거의 전부 `shared_ptr`(`rclcpp::Node::SharedPtr`, `std::make_shared<...>()`)로 다뤄짐 — 여러 콜백이 같은 노드를 공유하는 구조라서

### unique_ptr는 왜 복사가 안 될까 — 이동 의미론
unique_ptr는 "단독 소유"라 복사하면 소유자가 둘이 되는 모순이 생긴다. 그래서 복사는 금지되고 **이동(move)**만 허용된다 — 소유권을 넘기고 원래 포인터는 비운다.

```cpp
auto a = std::make_unique<Motor>();
// auto b = a;              // 컴파일 에러! 복사 불가
auto b = std::move(a);      // OK: 소유권이 a→b로 이동, 이제 a는 비었음
```

이동 의미론은 큰 데이터(`vector` 등)를 복사 없이 넘기는 성능 최적화의 핵심이기도 하다. 최신 컴파일러는 반환값도 자동으로 이동 처리한다.

## 클래스와 다형성
```cpp
class Sensor {
public:
    virtual ~Sensor() = default;             // 가상 소멸자 (상속 시 필수!)
    virtual std::vector<double> read() = 0;  // 순수 가상 (자식이 반드시 구현)
};

class Lidar : public Sensor {
public:
    std::vector<double> read() override { return {0.8, 0.82, 0.85}; }
};

std::vector<std::unique_ptr<Sensor>> sensors;
sensors.push_back(std::make_unique<Lidar>());
for (auto& s : sensors) {
    auto data = s->read();   // 다형성: 실제 타입의 read() 호출
}
```
- `virtual` 없이는 부모 포인터로 자식 함수를 불러도 다형성이 작동하지 않는다 — C++은 기본적으로 함수 호출을 정적으로 결정하기 때문
- `override`를 항상 붙이면 시그니처 오타 같은 실수를 컴파일러가 잡아준다
- 부모 포인터로 자식을 삭제할 때 자식 소멸자가 제대로 불리려면 **가상 소멸자** 필수 — 빠뜨리면 부분 소멸 누수

## STL — 표준 컨테이너

| 용도 | Python | C++ STL |
|---|---|---|
| 순서 있는 배열 | `list` | `std::vector<T>` |
| 키→값 조회 | `dict` | `std::map` / `std::unordered_map` |
| 중복 없는 모음 | `set` | `std::set` / `std::unordered_set` |
| 고정 크기 묶음 | `tuple` | `std::array`, `std::tuple` |
| 문자열 | `str` | `std::string` |

```cpp
std::vector<double> scan = {0.3, 8.2, 0.5, 12.0};
scan.push_back(1.1);
double closest = *std::min_element(scan.begin(), scan.end());

std::unordered_map<std::string, double> sensors;  // 조회 O(1)
sensors["battery"] = 11.9;

int near_count = std::count_if(scan.begin(), scan.end(),
                                [](double d){ return d < 1.0; });
```
- `vector`는 연속 메모리라 캐시 효율이 좋아 "특별한 이유가 없으면 vector"가 기본값
- 조회가 잦으면 `unordered_map`(해시, O(1)), 정렬 순서가 필요하면 `map`(트리, O(log n))
- 실시간 루프에서는 `reserve()`로 미리 용량을 확보해 루프 중 힙 재할당(지터)을 없앤다

```cpp
std::vector<double> buffer;
buffer.reserve(360);   // 루프 밖에서 1회, 이후 재할당 없음
```

## 정리
- 컴파일 언어(C++)는 실행 전 번역으로 속도와 조기 오류 검출을 얻고, CMake는 그 빌드를 자동화하는 조리법
- 힙은 기본적으로 피하고, 필요할 땐 `new`/`delete` 대신 스마트 포인터로 소유권을 타입으로 표현
- 기본은 `unique_ptr`(단독 소유, 복사 불가·이동만 가능), 공유가 필요할 때만 `shared_ptr`
- 다형성엔 `virtual` + `override` + 가상 소멸자, 컨테이너는 특별한 이유 없으면 `vector`

> new/delete를 직접 쓰지 말고, 스마트 포인터로 소유권을 표현하라 — 현대 C++의 핵심 원칙.

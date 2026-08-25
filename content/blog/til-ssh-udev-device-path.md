---
title: "[TIL] 원격접속(SSH)과 센서 장치 경로 고정 — /dev 장치 파일부터 udev 규칙까지"
date: 2026-08-25T21:00:00+09:00
description: "헤드리스 온보드 컴퓨터를 다루기 위한 SSH 원격접속 전 과정과, /dev 장치 파일·udev 속성으로 센서 장치 경로를 고정하는 방법 — 공개키 인증은 암호화가 아니라 서명이라는 것과, udevadm attribute-walk가 경로형 속성을 숨긴다는 함정까지"
image: "images/post/til-2026-08-25-banner.png"
categories: ["TIL"]
tags: ["ssh", "sshd", "ssh-keygen", "udev", "udevadm", "sysfs", "losetup", "linux", "robotics"]
type: "post"
nextp: ""
prevp: "ros2-lifecycle-composition-colcon"
---

## 왜 SSH부터인가

로봇의 온보드 컴퓨터에는 모니터도 키보드도 안 붙어 있다. 그래서 **다른 컴퓨터에서 셸만 빌려 쓰는 방법**이 개발의 출발점이 된다. 온보드 컴퓨터가 아직 없으니 내 노트북 자신을 접속 대상으로 세워서 같은 상황을 만들었다.

```bash
sudo apt install openssh-server
sudo systemctl start sshd
ssh pa17@localhost
```

- `ssh`(클라이언트)는 대부분 깔려 있지만 `sshd`(서버)는 따로 설치해야 한다. **접속을 받는 쪽에 필요한 건 서버**다
- `localhost`로 접속해도 실제 원격 접속과 경로가 완전히 같다 — TCP 22번 포트를 타고, 키 인증을 거치고, 새 의사 터미널(pty)을 받는다

## 서버가 22번 포트를 듣고 있는지 확인

원격 접속이 안 될 때 원인은 대개 셋 중 하나다. 서버가 안 떴거나, 포트를 안 듣거나, 방화벽이 막거나. 앞의 둘은 명령 두 개로 갈린다.

| 명령 | 확인하는 것 | 봐야 할 곳 |
|---|---|---|
| `systemctl status ssh` | 서비스가 살아 있나 | `Active: active (running)` |
| `ss -tlnp \| grep :22` | 실제로 소켓을 듣고 있나 | `LISTEN 0.0.0.0:22` |

```text
pa17@pa17 ~> systemctl status ssh
● ssh.service - OpenBSD Secure Shell server
     Active: active (running) since Tue 2026-08-25 09:26:15 JST; 1min 23s ago
Aug 25 09:26:15 pa17 sshd[7663]: Server listening on 0.0.0.0 port 22.
Aug 25 09:26:15 pa17 sshd[7663]: Server listening on :: port 22.

pa17@pa17 ~> ss -tlnp | grep :22
LISTEN 0      128          0.0.0.0:22        0.0.0.0:*
LISTEN 0      128             [::]:22           [::]:*
```

- `ss` 옵션은 **t**cp / **l**istening / **n**umeric(이름 해석 안 함) / **p**rocess다. `-n`이 없으면 22를 `ssh`라는 이름으로 바꿔 보여줘서 `grep :22`가 안 걸린다
- `0.0.0.0`과 `[::]` 두 줄이 뜨는 건 IPv4·IPv6 양쪽을 각각 듣고 있다는 뜻이다

## 키 쌍을 만들고 공개키를 등록

로봇은 부팅될 때마다 사람이 비밀번호를 쳐 줄 수 없다. 스크립트·CI·`scp`가 전부 무인으로 돌아야 하니 **키 기반 인증이 편의가 아니라 전제**다.

```bash
ssh-keygen -t ed25519 -C "pa17@robot"
ssh-copy-id pa17@localhost
```

- `-t`로 키 종류를 고른다. `dsa`·`ecdsa`·`ed25519`·`rsa` 중 요즘 기본값은 `ed25519`고, `rsa`를 쓸 거면 서명 방식이 `ssh-rsa`(SHA-1)가 아니라 `rsa-sha2-256`/`rsa-sha2-512`인지 봐야 한다
- `-C`는 주석이라 암호학적 의미가 없다. 그런데 서버의 `authorized_keys`에 그대로 남아서, **키가 여러 대 쌓였을 때 어느 기계 것인지 구분하는 유일한 단서**가 된다
- `ssh-copy-id`가 하는 일은 결국 공개키 한 줄을 서버의 `~/.ssh/authorized_keys`에 붙이고 권한을 맞춰주는 것뿐이다

> ### 🔑 공개키가 서버에 올라가는데 왜 안전한가 (Why · How · What)
>
> **Why — 무엇이 문제였나**
> "서버에 올라가는 건 어느 쪽이고 왜 안전한가"에 나는 **"공개키로 암호화한 건 개인키로만 복호화되니까 안전하다"**고 적었다. 결론(공개키가 올라가도 안전하다)은 맞는데 **근거가 틀렸다**. 이 설명대로면 인증이 "서버가 뭔가를 암호화해 보내고 클라이언트가 풀어서 되돌려주는" 과정이어야 하는데, 실제 SSH 공개키 인증은 그렇게 동작하지 않는다.
>
> **How — 어떻게 확인했나**
> RFC 4252의 `publickey` 인증을 보면, 클라이언트가 **세션 식별자를 포함한 데이터에 개인키로 서명**해서 보내고 서버는 `authorized_keys`의 공개키로 그 **서명을 검증**한다. 암호문을 주고받는 단계가 아예 없다. 결정적인 증거는 오늘 만든 `ed25519` 키 자체다 — Ed25519는 [RFC 8709](https://datatracker.ietf.org/doc/html/rfc8709)에서 SSH용 **서명 알고리즘**으로 정의된 것이고, 애초에 암호화 기능이 없다. 암호화/복호화로 설명하면 오늘 만든 키로는 로그인이 성립할 수 없다는 모순이 생긴다.
>
> **What — 무엇을 배웠나**
> 공개키가 서버에 올라가도 되는 이유는 **공개키로는 검증만 되고 서명은 못 만들기 때문**이다. 서버가 털려서 `authorized_keys`가 통째로 유출돼도 공격자는 내 개인키 없이 서명을 위조할 수 없다. 그리고 서명 대상에 세션 식별자가 들어가므로, 한 세션에서 탈취한 서명을 다른 세션에 재사용하는 것도 막힌다. **"공개키 = 자물쇠"보다 "공개키 = 인감 대조표"에 가깝다.**

## 지금 이 세션이 진짜 원격인지 증명

`ssh localhost`는 로컬 터미널과 화면이 똑같아서, 정말 SSH를 타고 들어온 건지 그냥 원래 셸인지 헷갈린다. 세 가지로 교차 확인했다.

| 명령 | 출력 | 읽는 법 |
|---|---|---|
| `who` | `pa17 tty2 (tty2)` / `pa17 pts/1 (127.0.0.1)` | 물리 콘솔은 `tty2`, 네트워크 세션은 `pts/*`에 접속 출처가 붙는다 |
| `tty` | `/dev/pts/1` | `/dev/tty*`면 물리 콘솔, `/dev/pts/*`면 의사 터미널 |
| `echo $SSH_CONNECTION` | `127.0.0.1 42618 127.0.0.1 22` | 출발 IP·포트, 도착 IP·포트. **sshd가 직접 넣어주는 변수라 이게 비어 있으면 SSH 세션이 아니다** |

- 셋 중 가장 확실한 건 `$SSH_CONNECTION`이다. `who`나 `tty`는 `tmux`·`screen`·터미널 에뮬레이터에서도 `pts`가 나오지만, 이 변수는 sshd만 설정한다
- 그래서 `.bashrc`에서 `[ -n "$SSH_CONNECTION" ]`로 원격일 때만 프롬프트에 호스트명을 띄우는 식으로 쓸 수 있다

## 헤드리스 운용에서 자주 쓰는 두 가지

로그인해서 손으로 치는 게 아니라, **접속 자체를 명령 한 줄에 끼워 넣는** 방식이다.

```bash
# 1) 붙지 않고 명령 하나만 실행하고 빠져나오기
pa17@pa17 ~> ssh pa17@localhost 'uname -a'
Linux pa17 6.8.0-138-generic #138~22.04.1-Ubuntu SMP PREEMPT_DYNAMIC x86_64 GNU/Linux

# 2) 파일 옮기기 — 같은 SSH 통로를 그대로 탄다
pa17@pa17 ~> scp hello pa17@localhost:hello2
hello                                    100%    0     0.0KB/s   00:00
```

- `ssh host '명령'`은 로그인 셸을 띄우지 않고 명령만 돌리고 종료한다. 로봇 상태를 긁어오거나 노드를 재시작하는 스크립트가 전부 이 형태다
- 대화형이 아니라서 `.bashrc`가 안 읽힐 수 있다 → **원격 명령에서 `ros2`를 못 찾는 전형적인 원인**이 여기다. `ssh host 'source ~/ros2_ws/install/setup.bash && ros2 node list'`처럼 명시적으로 source 해야 한다
- `scp`의 `:` 뒤는 원격 경로다. `hello2`처럼 상대경로면 원격 홈 기준

## `/dev/tty*` — 파일 종류와 소유 그룹

센서는 결국 `/dev` 밑의 **장치 파일**로 보인다. 그래서 접속에 성공한 다음 할 일은 장치 파일을 읽는 법을 아는 것이다.

```text
pa17@pa17 ~> ls -l /dev/tty*
crw-rw-rw- 1 root tty      5,  0 Aug 25 09:43 /dev/tty
crw--w---- 1 pa17 tty      4,  2 Aug 25 00:06 /dev/tty2
crw-rw---- 1 root dialout  4, 64 Aug 25 00:06 /dev/ttyS0
crw-rw---- 1 root dialout  4, 65 Aug 25 00:06 /dev/ttyS1
```

> 💡 **파일 종류 문자** (`ls -l` 맨 앞 한 글자)
>
> - `-` : 일반 파일
> - `d` : 디렉토리
> - `l` : 심볼릭 링크
> - `c` : 문자 장치 — 바이트를 순서대로 흘려보낸다. 터미널·시리얼 포트
> - `b` : 블록 장치 — 정해진 크기 단위로 임의 위치를 읽고 쓴다. 디스크

- 시리얼 포트(`ttyS*`)는 전부 **`c`(문자 장치)**이고 소유 그룹이 **`dialout`**이다. 라이다·IMU가 USB-시리얼로 붙으면 `/dev/ttyUSB*`로 잡힐 텐데 그룹은 똑같이 `dialout`이다
- 그래서 센서를 열려다 나는 `Permission denied`는 대개 코드 문제가 아니라 **내 계정이 `dialout`에 없는 것**이다 — `sudo usermod -aG dialout $USER` 후 재로그인
- `4, 64` 두 숫자는 major·minor 번호다. major가 커널의 어느 드라이버인지, minor가 그 드라이버 안의 몇 번째 장치인지를 가리킨다

## loop 장치로 센서 두 대가 꽂힌 상황 만들기

라이다와 IMU가 아직 없으니 **장치 파일이 두 개 생기는 상황**만 똑같이 만들면 된다. loop 장치는 파일을 블록 장치처럼 보이게 해주는 커널 기능이라 이 실습에 딱 맞다.

```bash
pa17@pa17 ~/S/fake_sensors> truncate -s 10M lidar.img imu.img
pa17@pa17 ~/S/fake_sensors> sudo losetup -f --show lidar.img
/dev/loop15
pa17@pa17 ~/S/fake_sensors> sudo losetup -f --show imu.img
/dev/loop16
```

- `truncate -s 10M`은 실제로 10MB를 쓰지 않고 **크기만 10MB인 희소 파일(sparse file)**을 만든다. 그래서 즉시 끝난다
- `losetup -f`는 "비어 있는 첫 loop 장치를 알아서 골라라", `--show`는 "고른 이름을 출력해라"다. **골라주는 번호가 그때그때 다르다**는 것 자체가 오늘의 핵심 문제로 이어진다
- 실제 센서도 똑같다. 부팅 순서나 USB 인식 순서에 따라 라이다가 `/dev/ttyUSB0`이 되기도, `/dev/ttyUSB1`이 되기도 한다. **번호를 코드에 박으면 언젠가 라이다 자리에서 IMU를 읽는다**

## 두 장치를 구분할 속성 찾기

번호가 흔들려도 변하지 않는 속성이 있어야 이름을 고정할 수 있다. `udevadm`으로 커널이 들고 있는 속성을 전부 훑었다.

```bash
udevadm info --attribute-walk /dev/loop15
udevadm info --attribute-walk /dev/loop16
```

두 출력을 나란히 놓고 다른 줄을 찾았는데, **거의 전부 똑같았다**.

| 속성 | loop15 | loop16 | 식별에 쓸 수 있나 |
|---|---|---|---|
| `KERNEL` | `loop15` | `loop16` | ✗ 이 번호가 흔들리는 게 문제다 |
| `ATTR{size}` | `20480` | `20480` | ✗ 같은 크기라 구분 불가 |
| `ATTR{diskseq}` | `35` | `37` | ✗ 붙일 때마다 증가하는 일련번호라 재부팅하면 달라진다 |
| `ATTR{loop/offset}`·`sizelimit`·`autoclear`·`dio` | `0` | `0` | ✗ 전부 기본값 |
| **`loop/backing_file`** | `…/lidar.img` | `…/imu.img` | **✓ 유일하게 다르고, 내가 정한 값** |

> ### 🔍 답인 `backing_file`이 목록에 안 보인다 (Why · How · What)
>
> **Why — 무엇이 문제였나**
> 답은 `loop/backing_file`인데 **`udevadm info --attribute-walk` 출력 어디에도 `ATTR{loop/backing_file}` 줄이 없다.** `loop/`로 시작하는 속성은 `autoclear`·`dio`·`offset`·`partscan`·`sizelimit` 다섯 개만 나오고, 정작 필요한 하나가 빠져 있다. "속성이 없는 건가, 내가 잘못 본 건가"에서 막혔다.
>
> **How — 어떻게 해결했나**
> sysfs를 직접 읽어보니 파일은 멀쩡히 있었다.
>
> ```bash
> cat /sys/block/loop15/loop/backing_file
> # /home/pa17/S/fake_sensors/lidar.img
> ```
>
> 원인은 udevadm 쪽이었다. systemd의 [`src/udev/udevadm-info.c`](https://github.com/systemd/systemd/blob/main/src/udev/udevadm-info.c)에서 속성을 출력하는 부분에 `if (value[0] == '/') continue;`가 있다 — **값이 `/`로 시작하면(= 경로처럼 보이면) 목록에서 빼버린다.** `backing_file` 값은 절대경로라 항상 이 필터에 걸린다. 즉 속성이 없는 게 아니라 **보여주지 않을 뿐**이었다.
>
> **What — 무엇을 배웠나**
> 중요한 건 이 필터가 **출력에만 적용되고 매칭에는 적용되지 않는다**는 점이다. 규칙에 `ATTR{loop/backing_file}`을 그대로 쓰면 udev는 sysfs를 직접 읽어서 정상적으로 매칭한다. 같은 상황을 loop 장치 두 개로 재현해 규칙을 넣고 `udevadm test /sys/block/loop0`을 돌리니 `Added SYMLINK 'lidar'`가 그대로 찍혔다.
>
> 그래서 교훈은 하나다 — **`attribute-walk`는 쓸 수 있는 속성의 전부가 아니다.** 찾는 속성이 안 보이면 포기하지 말고 `/sys/…` 경로를 직접 `ls`하고 `cat`해 봐야 한다. 특히 경로처럼 `/`로 시작하는 값이 그렇다.

## udev 규칙으로 이름 고정하기

구분할 속성을 찾았으니 **커널이 준 번호 대신 내가 정한 이름**을 붙인다. udev 규칙은 "이 조건에 맞는 장치가 나타나면 이렇게 해라"를 적어두는 파일이다.

```bash
# /etc/udev/rules.d/99-fake-sensors.rules
SUBSYSTEM=="block", KERNEL=="loop*", ATTR{loop/backing_file}=="/home/pa17/S/fake_sensors/lidar.img", SYMLINK+="lidar"
SUBSYSTEM=="block", KERNEL=="loop*", ATTR{loop/backing_file}=="/home/pa17/S/fake_sensors/imu.img", SYMLINK+="imu"
```

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
ls -l /dev/lidar /dev/imu
```

- `==`는 **조건**, `+=`는 **동작**이다. 이 둘을 섞어 쓰는 게 udev 규칙 문법의 기본이다
- `SYMLINK+=`는 원래 이름을 바꾸는 게 아니라 **별명을 하나 더 붙인다**. `/dev/loop15`는 그대로 있고 `/dev/lidar`가 그걸 가리킨다 — 그래서 기존 도구가 깨지지 않는다
- 파일명 앞의 `99-`는 적용 순서다. 숫자가 클수록 나중에 실행돼서, 시스템 기본 규칙(`60-persistent-storage.rules` 등)이 다 돈 뒤에 내 규칙이 올라간다
- 실제 USB 센서라면 조건이 `SUBSYSTEM=="tty"`에 `ATTRS{idVendor}`·`ATTRS{idProduct}`·`ATTRS{serial}`로 바뀐다. **같은 모델을 두 개 꽂으면 vendor/product가 같으니 `serial`이 유일한 구분자**가 된다
- 규칙을 고쳤으면 반드시 `udevadm test /sys/…`로 먼저 시뮬레이션한다. 장치를 다시 꽂지 않고도 어느 규칙 몇 번째 줄이 걸렸는지 그대로 찍힌다

## 정리

오늘 한 일은 **"모니터 없는 컴퓨터에 들어가서, 흔들리는 장치 이름을 붙박이로 만드는 것"** 한 줄로 꿰인다.

1. 헤드리스 개발의 전제는 SSH다. 안 될 때는 `systemctl status ssh`(서비스가 살았나)와 `ss -tlnp | grep :22`(소켓을 듣나)로 원인을 두 갈래로 좁힌다.
2. 공개키 인증은 암호화가 아니라 **서명**이다. 클라이언트가 세션 식별자에 개인키로 서명하고 서버가 공개키로 검증한다. 오늘 만든 ed25519 키는 서명 전용이라 애초에 암호화할 수 없다는 게 그 증거다.
3. `$SSH_CONNECTION`은 sshd만 설정하는 변수라, 지금 세션이 진짜 원격인지 판별하는 가장 확실한 근거다. `tty`나 `who`의 `pts`는 tmux에서도 나온다.
4. `ssh host '명령'`은 로그인 셸을 안 띄운다. 원격 스크립트에서 `ros2`를 못 찾는 이유가 여기고, `source install/setup.bash`를 명령 안에 같이 넣어야 한다.
5. 센서는 `/dev`의 문자 장치(`c`)로 보이고 소유 그룹은 `dialout`이다. `Permission denied`의 대부분은 코드가 아니라 그룹 문제다.
6. `losetup -f`가 골라주는 번호가 매번 다르듯, 실제 센서의 `/dev/ttyUSB*` 번호도 인식 순서에 따라 바뀐다. **번호를 코드에 박으면 언젠가 라이다 자리에서 IMU를 읽는다.**
7. 그래서 udev 규칙으로 **변하지 않는 속성**(loop은 `backing_file`, USB 센서는 `idVendor`/`idProduct`/`serial`)을 조건 삼아 `SYMLINK+=`로 별명을 붙인다. 원래 이름은 남으니 부작용이 없다.
8. `udevadm info --attribute-walk`는 값이 `/`로 시작하는 속성을 출력에서 걸러낸다. **안 보인다고 없는 게 아니다** — 매칭에는 멀쩡히 쓸 수 있고, 확인은 `/sys`를 직접 읽으면 된다.

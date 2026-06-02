# ASP PX4/Gazebo Simulation Environment

## 목적

이 문서는 `jsb` 브랜치 기준으로 ASP PX4/Gazebo 시뮬레이션 환경의 변경 내용을 정리한다. 원본 코드 대비 UGV Gazebo model name과 DiffDrive topic 이름이 불일치하던 문제를 정리하고, PX4 SITL + Gazebo Sim 실행 및 검증 방법을 설명한다.

## 프로젝트 개요

이 저장소는 ASP 자율주행 시스템 플랫폼 실험을 위한 PX4-Autopilot 기반 시뮬레이션 환경이다. PX4 SITL + Gazebo Sim을 사용한다.

UAV `x500_gimbal_0`와 UGV `X1_asp`가 같은 Gazebo world에서 동작한다. 실험 목적은 UAV/UGV 협동 탐색, 센서 연동, ROS2 제어 인터페이스 검증이다.

## 원본 저장소

```text
upstream: https://github.com/chan-1207/PX4-Autopilot_ASP.git
origin:   https://github.com/subin11111/asp.git
working branch: jsb
```

## 주요 시뮬레이션 구성

* World: `default`
* Park/world model: `simple_baylands`
* UAV model: `x500_gimbal`
* Runtime UAV name: `x500_gimbal_0`
* UGV model name: `X1_asp`
* ArUco markers
* tunnel debris
* collapsed buildings
* hatchback obstacles
* camera / LiDAR / IMU / GNSS sensors

## 변경 사항

| File | Type | Description |
| --- | --- | --- |
| `Tools/simulation/gz/worlds/default.sdf` | Modified | UGV DiffDrive odometry/cmd_vel topic을 `X1_asp` 기준으로 수정 |

## default.sdf 수정 내용

기존 DiffDrive topic은 다음과 같았다.

```xml
<odom_topic>/model/X1/odometry</odom_topic>
<cmd_vel_topic>/model/X1/cmd_vel</cmd_vel_topic>
```

수정 후 topic은 다음과 같다.

```xml
<odom_topic>/model/X1_asp/odometry</odom_topic>
<cmd_vel_topic>/model/X1_asp/cmd_vel</cmd_vel_topic>
```

수정 이유는 다음과 같다.

* Gazebo model name이 `X1_asp`이므로 DiffDrive topic도 `/model/X1_asp/...`로 통일해야 한다.
* 기존 `/model/X1/...` topic은 실제 모델 이름과 불일치하여 ROS2 bridge 및 제어 연결에서 혼동을 유발했다.
* 수정 후 Gazebo에서 `/model/X1_asp/cmd_vel`, `/model/X1_asp/odometry`가 정상 생성된다.

## 확인된 Gazebo Topic

```text
/model/X1_asp/cmd_vel
/model/X1_asp/odometry
/model/X1_asp/pose
/model/X1_asp/pose_static
/model/X1_asp/tf
/world/default/model/X1_asp/link/base_link/sensor/camera_front/camera_info
/world/default/model/X1_asp/link/base_link/sensor/camera_front/image
/world/default/model/X1_asp/link/base_link/sensor/gpu_lidar/scan
/world/default/model/X1_asp/link/base_link/sensor/gpu_lidar/scan/points
```

## 실행 방법

PX4/Gazebo를 실행한다.

```bash
cd ~/PX4-Autopilot_ASP
make px4_sitl gz_x500_gimbal
```

## 검증 방법

Gazebo topic을 확인한다.

```bash
gz topic -l | grep -Ei "X1_asp|x500_gimbal_0|cmd_vel|odometry|camera|lidar|imu|pose"
```

UGV DiffDrive plugin을 직접 제어하여 확인할 수 있다.

```bash
gz topic -t /model/X1_asp/enable \
  -m gz.msgs.Boolean \
  -p "data: true"
```

```bash
gz topic -t /model/X1_asp/cmd_vel \
  -m gz.msgs.Twist \
  -p "linear: {x: 1.0}, angular: {z: 0.0}"
```

이 명령으로 UGV가 움직이면 Gazebo DiffDrive plugin은 정상이다. ROS2 keyboard control은 별도 저장소 `asp_ws`의 UGV bridge와 keyboard node를 통해 연결한다.

## ROS2 Workspace 연동

연동 저장소는 다음과 같다.

```text
https://github.com/subin11111/asp_ws
```

ROS2 쪽 최종 구조는 다음과 같다.

```text
/command/ugv_cmd_vel
  -> ros_gz_bridge
  -> /model/X1_asp/cmd_vel
```

UAV와 UGV 명령은 다음처럼 분리한다.

```text
/command/twist       = UAV 전용
/command/ugv_cmd_vel = UGV 전용
```

## 주의 사항

* PX4 저장소는 submodule이 많으므로 `--recursive` clone이 필요하다.
* VS Code에서 submodule들이 여러 Git repository로 보일 수 있으나 정상이다.
* submodule 내부 `.git`은 삭제하지 말 것.
* ROS2 workspace의 `utilities_pkg`와 달리, PX4 내부 submodule은 유지해야 한다.

clone 명령은 다음과 같다.

```bash
git clone --recursive https://github.com/subin11111/asp.git
```

submodule 복구는 다음 명령으로 수행한다.

```bash
git submodule update --init --recursive
```

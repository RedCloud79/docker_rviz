# SSH → Docker RViz 실행

## 상황 1. 서버 PC GUI 상태 확인

서버 PC 모니터에서 직접 터미널을 열고 실행:

```bash
echo $DISPLAY
```

정상:

```text
:0
```

또는:

```text
:1
```

값이 없으면 서버 PC의 GUI 터미널에서 다시 실행해야 함.

---

## 상황 2. 서버 PC에서 로봇 접속

```bash
ssh -Y user@10.21.31.104
```

접속 후 확인:

```bash
echo $DISPLAY
```

정상:

```text
localhost:10.0
```

또는:

```text
localhost:11.0
```

값이 나오면 RViz 실행 단계로 진행.

---

## 상황 3. SSH 접속 후 DISPLAY가 비어 있음

일반 SSH로 접속했으면 종료:

```bash
exit
```

`-Y` 옵션으로 다시 접속:

```bash
ssh -Y user@10.21.31.104
```

그래도 값이 없으면 로봇에서 확인:

```bash
which xauth
sudo sshd -T | grep -i x11
```

정상 설정:

```text
x11forwarding yes
x11displayoffset 10
x11uselocalhost yes
```

`xauth`가 없으면 설치(서버 pc에서 패키지를 scp를 통해서 전달 할 수도 있음):
인터넷이 안되는경우 - `witch xauth` 를 통해서 경로 확인

```bash
sudo apt install -y xauth
```
or

```bash
#서버 pc에서 진행
mkdir -p ~/xauth_offline

sudo apt install --download-only xauth
# 아키텍처가 다른경우
apt download xauth:arm64
cp /var/cache/apt/archives/*.deb ~/xauth_offline/

scp ~/xauth_offline/*.deb user@10.21.31.104:/tmp/xauth_offline/


# 로봇에 접속해서 진행
cd /tmp/xauth_offline
sudo dpkg -i *.deb
```

SSH 설정 수정:

```bash
sudo nano /etc/ssh/sshd_config
```

추가 또는 수정:

```text
X11Forwarding yes
X11DisplayOffset 10
X11UseLocalhost yes
```

SSH 재시작:

```bash
sudo systemctl restart ssh
```

다시 접속:

```bash
exit
ssh -Y user@10.21.31.104
```

---

## 상황 4. DISPLAY가 정상일 때 Xauthority 생성

```bash
XAUTH_FILE=$(mktemp /tmp/docker-xauth.XXXXXX)

xauth nlist "$DISPLAY" \
  | sed 's/^..../ffff/' \
  | xauth -f "$XAUTH_FILE" nmerge -
```

생성 확인:

```bash
xauth -f "$XAUTH_FILE" list
```

---

## 상황 5. Xauthority를 Docker에 복사

```bash
sudo docker cp "$XAUTH_FILE" noetic_robot:/tmp/.docker.xauth
```

복사 확인:

```bash
sudo docker exec noetic_robot ls -l /tmp/.docker.xauth
```

---

## 상황 6. Docker에서 RViz 실행

```bash
sudo docker exec -it \
  -e DISPLAY="$DISPLAY" \
  -e XAUTHORITY=/tmp/.docker.xauth \
  -e QT_X11_NO_MITSHM=1 \
  noetic_robot bash -lc '
    source /opt/ros/noetic/setup.bash
    source /root/catkin_ws/devel/setup.bash 2>/dev/null || true

    export ROS_MASTER_URI=http://10.21.31.104:11311
    export ROS_IP=10.21.31.104
    unset ROS_HOSTNAME
    unset LIBGL_ALWAYS_SOFTWARE

    rviz -d /root/catkin_ws/src/navigation_3d/hdl_localization/rviz/hdl_localization.rviz
  '
```

---

## 상황 7. X11 인증 오류

오류 예:

```text
Authorization required
X11 connection rejected because of wrong authentication
```

인증 파일 다시 생성:

```bash
rm -f /tmp/docker-xauth.*

XAUTH_FILE=$(mktemp /tmp/docker-xauth.XXXXXX)

xauth nlist "$DISPLAY" \
  | sed 's/^..../ffff/' \
  | xauth -f "$XAUTH_FILE" nmerge -

sudo docker cp "$XAUTH_FILE" noetic_robot:/tmp/.docker.xauth
```

이후 RViz 실행 명령 재실행.

---

## 상황 8. X 서버 연결 오류

오류 예:

```text
qt.qpa.xcb: could not connect to display
```

확인:

```bash
echo $DISPLAY
xauth list
sudo docker exec noetic_robot ls -l /tmp/.docker.xauth
```

Docker 네트워크 확인:

```bash
sudo docker inspect noetic_robot \
  --format '{{.HostConfig.NetworkMode}}'
```

정상:

```text
host
```

---

## 상황 9. OpenGL 또는 GLX 오류

오류 예:

```text
libGL error
failed to load driver
Could not initialize GLX
```

소프트웨어 렌더링으로 실행:

```bash
sudo docker exec -it \
  -e DISPLAY="$DISPLAY" \
  -e XAUTHORITY=/tmp/.docker.xauth \
  -e QT_X11_NO_MITSHM=1 \
  -e LIBGL_ALWAYS_SOFTWARE=1 \
  noetic_robot bash -lc '
    source /opt/ros/noetic/setup.bash
    source /root/catkin_ws/devel/setup.bash 2>/dev/null || true

    export ROS_MASTER_URI=http://10.21.31.104:11311
    export ROS_IP=10.21.31.104
    unset ROS_HOSTNAME

    rviz -d /root/catkin_ws/src/navigation_3d/hdl_localization/rviz/hdl_localization.rviz
  '
```

---

## 상황 10. RViz는 열리지만 ROS 데이터가 안 보임

컨테이너 접속:

```bash
sudo docker exec -it noetic_robot bash
```

컨테이너 내부 확인:

```bash
source /opt/ros/noetic/setup.bash
source /root/catkin_ws/devel/setup.bash

export ROS_MASTER_URI=http://10.21.31.104:11311
export ROS_IP=10.21.31.104
unset ROS_HOSTNAME

rostopic list
rostopic echo -n 1 /tf
```

RViz에서 Fixed Frame 확인:

```text
map
```

필요 시:

```text
odom
base_link
```

---

## 평소 실행용

서버 PC:

```bash
ssh -Y user@10.21.31.104
```

로봇 접속 후:

```bash
echo $DISPLAY
```

Xauthority 생성 및 복사:

```bash
XAUTH_FILE=$(mktemp /tmp/docker-xauth.XXXXXX)

xauth nlist "$DISPLAY" \
  | sed 's/^..../ffff/' \
  | xauth -f "$XAUTH_FILE" nmerge -

sudo docker cp "$XAUTH_FILE" noetic_robot:/tmp/.docker.xauth
```

RViz 실행:

```bash
sudo docker exec -it \
  -e DISPLAY="$DISPLAY" \
  -e XAUTHORITY=/tmp/.docker.xauth \
  -e QT_X11_NO_MITSHM=1 \
  noetic_robot bash -lc '
    source /opt/ros/noetic/setup.bash
    source /root/catkin_ws/devel/setup.bash 2>/dev/null || true

    export ROS_MASTER_URI=http://10.21.31.104:11311
    export ROS_IP=10.21.31.104
    unset ROS_HOSTNAME
    unset LIBGL_ALWAYS_SOFTWARE

    rviz -d /root/catkin_ws/src/navigation_3d/hdl_localization/rviz/hdl_localization.rviz
  '
```

---
.Xauthority 파일 그대로 복사해서 RViz 실행
# 서버 PC에서 로봇 접속
```bash
ssh -Y user@10.21.31.104
```
접속 후 확인:
```
echo $DISPLAY
XAUTH_SRC="${XAUTHORITY:-$HOME/.Xauthority}"
ls -l "$XAUTH_SRC"
```
정상이면 컨테이너로 복사:
```
sudo docker cp "$XAUTH_SRC" noetic_robot:/tmp/.docker.xauth
```
RViz 실행:
```
sudo docker exec -it \
  -e DISPLAY="$DISPLAY" \
  -e XAUTHORITY=/tmp/.docker.xauth \
  -e QT_X11_NO_MITSHM=1 \
  noetic_robot bash -lc '
    source /opt/ros/noetic/setup.bash
    source /root/catkin_ws/devel/setup.bash 2>/dev/null || true

    export ROS_MASTER_URI=http://10.21.31.104:11311
    export ROS_IP=10.21.31.104
    unset ROS_HOSTNAME
    unset LIBGL_ALWAYS_SOFTWARE

    rviz -d /root/catkin_ws/src/navigation_3d/hdl_localization/rviz/hdl_localization.rviz
  '
```
조건:

echo $DISPLAY → localhost:10.0 등 출력
.Xauthority 파일 → 실제로 존재

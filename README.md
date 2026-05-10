# Jetson_nano_devkit

* 링크
   * https://docs.nvidia.com/
   * https://docs.nvidia.com/omniverse/index.html
   * https://docs.isaacsim.omniverse.nvidia.com/4.5.0/ros2_tutorials/tutorial_ros2_navigation.html

* Jetson AI 인증
   * Jetson 개발자 키트 및 NVIDIA의 무료 온라인 교육을 통해 직접 AI를 학습해보세요. 이러한 과정을 완료하면 Jetson 및 AI 개발 역량을 입증하는 인증서를 받으시게 됩니다.
   * https://www.nvidia.com/en-us/training/

* https://developer.nvidia.com/embedded/community/support-resources

* Jetson Nano
   * Jetson Nano System-on-Module Data Sheet : https://developer.nvidia.com/embedded/dlc/jetson-nano-system-module-datasheet
   * Jetson Developer Kit user guides : https://developer.nvidia.com/embedded/learn/jetson-nano-2gb-devkit-user-guide
   * Jetson Linux Developer Guide  : https://docs.nvidia.com/jetson/archives/r34.1/DeveloperGuide/index.html
   * Pin and function names guides : https://developer.nvidia.com/embedded/learn/jetson-nano-2gb-devkit-user-guide
   * Jetson Nano Developer Kit Carrier Board Specification : https://developer.nvidia.com/jetson-nano-developer-kit-carrier-board-p3449-b01-specification
   * Jetson PCN Center

* Get Started with NVIDIA Jetson
   * https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-devkit


* 참고
   * https://github.com/dusty-nv/jetbot_ros#jetbot-model-for-gazebo-robotics-simulator%5B/url%5D
 
* Nvidia Solution
   * CUDA Toolkit
       * Toolkit for GPU-accelerated apps: libraries, debugging/optimization tools, a C/C++ compiler, and a runtime.
   * NVIDIA HPC SDK
       * A comprehensive suite of C, C++, and Fortran compilers, libraries, and tools for GPU-accelerating HPC applications.
   * CUDA-X Libraries
       * GPU-accelerated libraries delivering improved performance across a wide variety of application domains.
   * Jetson
       * Embedded solutions for automomous machines and edge computing.
   * Isaac
       * Robotic AI development and simulation platform.
   * Clara
       * Frameworks for AI-powered imaging, genomics, and smart sensors.
   * DRIVE
       * Platform for autonomous vehicles, data center-hosted simulation, and neural network training.
   * Metropolis
       * Solutions for smart cities, intelligent video analytics, and more.
   * Gameworks
       * Tools, samples and libraries for real-time graphics and physics.
   * Riva
       * Framework for multimodal conversational AI.
   * Developer Tools
       * IDE plugins, debugging, performance optimization, and other tools.
   * Graphics Research Tools
       * Tools, libraries, and samples from NVIDIA Research.
   * Omniverse
       * A powerful multi-GPU real-time simulation and collaboration platform for 3D production pipelines.

## Jetson Nano에 MobaXterm과 TigerVNC를 조합하여 리모트 접속

### 1단계: Jetson Nano 설정 (TigerVNC 서버)
   * 먼저 Jetson Nano에 접속하여 TigerVNC를 설치하고 설정을 완료해야 합니다.

   * 1. TigerVNC 및 데스크탑 환경 설치 (가벼운 XFCE 추천)

```Bash
sudo apt update
sudo apt install tigervnc-standalone-server tigervnc-xorg-extension xfce4 xfce4-goodies -y
```

   * 2. VNC 비밀번호 설정

```Bash
vncpasswd
```

   * 3. VNC 서버 실행 (최초 실행 시 설정 파일이 생성됩니다)
```Bash
vncserver :1 -geometry 1920x1080 -depth 24
```
   * :1은 디스플레이 번호를 뜻하며, 포트 번호 5901에 대응됩니다.

### 2단계: MobaXterm에서 접속 (VNC Session)
   * 이제 윈도우 PC에서 MobaXterm을 실행합니다.

   * 방법 A: 일반 VNC 세션 연결
      * 동일한 네트워크(LAN)에 있다면 바로 연결할 수 있습니다.
      * 1. 상단의 Session 버튼 클릭 -> VNC 선택.
      * 2. Remote hostname: Jetson Nano의 IP 주소를 입력.
      * 3. Port: 5901 입력 (디스플레이 번호 :1 기준).
      * 4. OK를 누르고 아까 설정한 VNC 비밀번호를 입력하면 화면이 뜹니다.

   * 방법 B: SSH 터널링을 이용한 보안 연결 (강력 추천)
      * 네트워크 보안이 중요하거나 외부망에서 접속할 때 유용합니다.
      * 1. MobaXterm의 Session -> VNC 설정 창에서 Network settings 탭으로 이동합니다.
      * 2. SSH Gateway (jump host) 설정을 활성화하여 Jetson Nano의 SSH 정보(IP, ID)를 입력합니다.
      * 3. 이렇게 하면 VNC 데이터가 SSH 암호화 통로를 통해 전달되므로 훨씬 안전합니다.

---

# Jetson 보드 모델을 확인하는 방법

1. /proc/device-tree/model 읽기 (가장 확실)
```
cat /proc/device-tree/model
```

```
gotree94@gotree94-desktop2:~$ cat /proc/device-tree/model
NVIDIA Jetson Nano Developer Kitgotree94@gotree94-desktop2:~$ 
```

출력 예시:
```
NVIDIA Jetson Nano Developer Kit → Jetson Nano
NVIDIA Jetson Orin Nano Developer Kit → Jetson Orin Nano
NVIDIA Jetson AGX Orin Developer Kit → Jetson AGX Orin (Orin Nano Super 아님)
```

2. Orin Nano Super 특별 확인
Orin Nano Super는 JetPack 6.2+ 에서 지원되며, Orin Nano와 동일한 하드웨어에서 소프트웨어 업그레이드로 성능이 향상된 모델입니다. 다음 명령어로 추가 확인:
```
# NVIDIA SoC 정보
cat /proc/device-tree/compatible

# CPU 정보
lscpu | grep "Model name"

# GPU 정보
cat /proc/device-tree/gpu-name
# 또는
cat /sys/devices/gpu.0/device

# Max GPU frequency (Orin Nano vs Super 구분 포인트)
sudo cat /sys/kernel/debug/bpmp/debug/clk/gpu/max_rate
# Orin Nano: 625 MHz
# Orin Nano Super: 800 MHz (또는 750MHz)
```

3. jetson_release.py (가장 user-friendly)
```
# 있으면 실행
cat /etc/nv_tegra_release
# 또는
python3 -c "import subprocess; print(subprocess.check_output(['head', '-n', '1', '/etc/nv_tegra_release']).decode())"
```

4. tegrastats 로 GPU 주파수 확인
```
sudo tegrastats
```

요약: 구분 기준
| 모델	| /proc/device-tree/model	| GPU 최대 클럭	| Tensor Core | 
|:-------------:|:-------------:|:-------------:|:-------------:|
| Jetson Nano	| Jetson Nano	| 921 MHz (Maxwell)	| 없음 | 
| Jetson Orin Nano	| Orin Nano	| 625 MHz	(Ampere) |  있음  | 
| Jetson Orin Nano Super | Orin Nano (동일)	| 800 MHz	| 있음 | 

* Orin Nano와 Orin Nano Super는 하드웨어가 같고, 전력/클럭 설정과 JetPack 버전으로 구분됩니다. 가장 정확한 방법은 GPU 최대 주파수를 확인하는 것입니다.

---

* Jetson Nano 환경에서 Firefox를 설치하는 방법입니다. Snap 버전은 Jetson에서 CPU 100% 먹는 문제가 있으니 반드시 Mozilla PPA로 설치해야 합니다.

* JetPack 버전 확인 (선행)
```
cat /etc/nv_tegra_release
# 또는
cat /etc/os-release  # Ubuntu 버전 확인
```

* 방법 1: Mozilla PPA (권장 — JetPack 5/6, Ubuntu 20.04/22.04)
```
# Snap으로 깔린 Firefox가 있다면 먼저 제거
sudo snap remove firefox

# Mozilla PPA 추가
sudo add-apt-repository ppa:mozillateam/ppa
sudo apt update

# Snap보다 PPA 우선순위 높게 설정
echo '
Package: *
Pin: release o=LP-PPA-mozillateam
Pin-Priority: 1001
' | sudo tee /etc/apt/preferences.d/mozilla-firefox

# 설치
sudo apt install firefox
```

* 방법 2: 직접 .deb 다운로드 (JetPack 4.x, Ubuntu 18.04)
* PPA가 18.04를 지원하지 않으므로 ARM64 .deb를 직접 받아야 합니다.
```
# Snap 제거
sudo snap remove firefox

# 최신 ARM64 Firefox .deb 찾기 (런치패드에서 확인)
# https://launchpad.net/~ubuntu-mozilla-security/+archive/ubuntu/ppa/+builds?build_text=&build_state=all
# arm64, ubuntu focal (20.04) 용 최신 빌드 링크를 복사

# 예시 (최신 버전에 맞게 URL 변경 필요)
wget -O firefox.deb "https://launchpad.net/~ubuntu-mozilla-security/+archive/ubuntu/ppa/+build/ID/build_id/+files/firefox_VERSION_arm64.deb"

# 설치
sudo apt install ./firefox.deb -y

# PPA 등록해서 업데이트 받을 수 있도록
sudo add-apt-repository ppa:mozillateam/ppa
echo '
Package: *
Pin: release o=LP-PPA-mozillateam
Pin-Priority: 1001
' | sudo tee /etc/apt/preferences.d/mozilla-firefox

# 이후 업데이트
sudo apt update
sudo apt upgrade firefox -y
```

* 방법 3: Flatpak (대안)
```
sudo apt install flatpak
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install flathub org.mozilla.firefox
```

* ⚠️ 중요: Snap 버전 쓰지 마세요
* Jetson Nano의 Ubuntu 22.04+에서 snap Firefox는 하드웨어 가속이 안 되어 CPU만 100% 먹습니다. 반드시 위 방법(PPA 또는 Flatpak)을 사용하세요.
* JetPack 6 (Ubuntu 22.04) 기준 가장 간단한 명령어 한 줄 요약:
```
sudo snap remove firefox 2>/dev/null; sudo add-apt-repository ppa:mozillateam/ppa -y; sudo apt update; sudo apt install firefox -y
```


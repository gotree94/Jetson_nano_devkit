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
   * CUDA Toolkit : Toolkit for GPU-accelerated apps: libraries, debugging/optimization tools, a C/C++ compiler, and a runtime.
   * NVIDIA HPC SDK : A comprehensive suite of C, C++, and Fortran compilers, libraries, and tools for GPU-accelerating HPC applications.
   * CUDA-X Libraries : GPU-accelerated libraries delivering improved performance across a wide variety of application domains.
   * Jetson : Embedded solutions for automomous machines and edge computing.
   * Isaac : Robotic AI development and simulation platform.
   * Clara : Frameworks for AI-powered imaging, genomics, and smart sensors.
   * DRIVE : Platform for autonomous vehicles, data center-hosted simulation, and neural network training.
   * Metropolis : Solutions for smart cities, intelligent video analytics, and more.
   * Gameworks : Tools, samples and libraries for real-time graphics and physics.
   * Riva : Framework for multimodal conversational AI.
   * Developer Tools : IDE plugins, debugging, performance optimization, and other tools.
   * Graphics Research Tools : Tools, libraries, and samples from NVIDIA Research.
   * Omniverse : A powerful multi-GPU real-time simulation and collaboration platform for 3D production pipelines.

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

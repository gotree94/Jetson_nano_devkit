# Module 00 — 개발 환경 구축 및 시스템 초기화

> **소요 시간**: 2~3시간  
> **난이도**: ★☆☆☆☆  
> **사전 준비**: microSD 32GB 이상 (UHS-I U3), HDMI 모니터, USB 키보드/마우스, 5V⎓4A 전원 어댑터

---

## 1. 개요

Jetson Nano Developer Kit에서 학습을 시작하기 위한 첫 단계입니다.  
운영체제 설치부터 원격 접속 설정, 시스템 모니터링 도구 사용법, Python 개발 환경 구성까지 다룹니다.

### 학습 목표
- JetPack 4.6.x SD 카드 이미지를 올바르게 플래싱할 수 있다
- SSH / VNC를 통해 헤드리스(headless) 환경에서 Jetson Nano에 접속할 수 있다
- `tegrastats`, `jetson_clocks`, `nvidia-smi`로 시스템 상태를 확인할 수 있다
- Python3 가상 환경과 GPIO/I2C/SPI 라이브러리를 설치할 수 있다
- 호스트 PC에서 aarch64 크로스 컴파일 환경을 이해한다

### 사전 지식
- Linux 기본 명령어 (`ls`, `cd`, `cp`, `sudo`, `apt`)
- SD 카드 포맷 및 이미지写入 경험

---

## 2. 이론

### 2.1 JetPack SDK

NVIDIA JetPack은 Jetson 보드용 통합 소프트웨어 패키지입니다.  
여기에는 다음이 포함됩니다:

| 구성 요소 | 설명 |
|-----------|------|
| **L4T (Linux for Tegra)** | 커스텀 Ubuntu 18.04/20.04 기반 Linux OS |
| **CUDA Toolkit** | GPU 병렬 컴퓨팅을 위한 라이브러리 및 컴파일러 |
| **cuDNN** | 딥러닝 연산 가속 라이브러리 |
| **TensorRT** | 추론 최적화 엔진 |
| **OpenCV** | CUDA 가속이 적용된 컴퓨터 비전 라이브러리 |
| **Multimedia API** | 하드웨어 비디오 인코더/디코더 API |

> Jetson Nano Developer Kit은 SD 카드 이미지 방식을 사용합니다.  
> (Jetson Xavier NX 이상은 NVMe SSD 방식도 지원)

### 2.2 Jetson Nano 파워 모드

Jetson Nano는 두 가지 파워 모드를 지원합니다:

| 모드 | 전력 | CPU 클럭 | GPU 클럭 | 사용 시나리오 |
|------|------|----------|----------|-------------|
| **MAXN** (10W) | 10W | 1.479 GHz | 921 MHz | 최대 성능, AI 추론 |
| **5W** | 5W | 0.918 GHz | 0 MHz (off) | 저전력, GPIO 제어 |

모드 전환 명령어:
```bash
# MAXN 모드
sudo nvpmodel -m 0

# 5W 모드  
sudo nvpmodel -m 1

# 현재 모드 확인
sudo nvpmodel --query
```

### 2.3 JetPack vs Raspberry Pi OS 비교

| 항목 | Jetson Nano (JetPack) | Raspberry Pi (Raspberry Pi OS) |
|------|----------------------|-------------------------------|
| 베이스 OS | Ubuntu 18.04 (aarch64) | Debian 11/12 (armhf/arm64) |
| 패키지 매니저 | `apt` | `apt` |
| GPIO 라이브러리 | `Jetson.GPIO` | `RPi.GPIO` / `gpiozero` |
| GPU | 128-core Maxwell | VideoCore VI |
| AI 가속 | CUDA + TensorRT | (없음 / TPU 별도) |

---

## 3. 실습 코드

### 실습 0-1: SD 카드 플래싱 (호스트 PC)

**Windows** — BalenaEtcher 사용:
```bash
# 1. https://www.balena.io/etcher/ 에서 Etcher 다운로드 및 설치
# 2. JetPack 4.6.1 SD 카드 이미지 다운로드
#    https://developer.nvidia.com/jetson-nano-sd-card-image
# 3. SD 카드를 PC에 삽입
# 4. Etcher 실행 → Flash from file → 대상 SD 카드 선택 → Flash
# 5. 완료 후 SD 카드를 Jetson Nano에 삽입
```

**Linux (dd)**:
```bash
# SD 카드 장치 확인
lsblk
# 예: /dev/sdb

# 이미지 플래싱 (주의: bs=1M, of= 장치 경로 정확히 확인!)
sudo dd if=jetson-nano-jp461-sd-card-image.img of=/dev/sdb bs=1M status=progress conv=fsync
```

### 실습 0-2: 첫 부팅 및 초기 설정

Jetson Nano에 HDMI 모니터, USB 키보드/마우스, 전원을 연결한 후 부팅합니다.

부팅 후 초기 설정:
```bash
# 1. 시스템 업데이트
sudo apt update
sudo apt upgrade -y

# 2. Swap 메모리 확장 (선택, 4GB RAM 부족 시)
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
# /etc/fstab 에 추가하여 영구 적용
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab

# 3. 호스트네임 설정
sudo hostnamectl set-hostname jetson-nano
```

### 실습 0-3: SSH 서버 활성화

```bash
# SSH 서버 설치 및 시작
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh

# 방화벽 확인 (ufw 사용 시)
sudo ufw allow 22

# IP 주소 확인
ip addr show | grep inet
# 또는
hostname -I
```

**PC에서 접속**:
```bash
ssh username@192.168.x.x
```

### 실습 0-4: VNC 원격 데스크탑

```bash
# TigerVNC 설치
sudo apt install -y tigervnc-standalone-server tigervnc-common

# VNC 비밀번호 설정 (최초 1회)
vncpasswd

# 화면 해상도 설정용 xstartup 파일 생성
mkdir -p ~/.vnc
cat > ~/.vnc/xstartup << 'EOF'
#!/bin/bash
export DISPLAY=:1
export XDG_SESSION_TYPE=x11
gnome-session &
EOF
chmod +x ~/.vnc/xstartup

# VNC 서버 시작 (해상도 1280x720)
vncserver -geometry 1280x720 -depth 24 :1

# systemd 서비스 등록 (부팅 시 자동 시작)
sudo bash -c 'cat > /etc/systemd/system/vncserver@.service << "EOF"
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=simple
User=your-username
PIDFile=/home/your-username/.vnc/%H-%i.pid
ExecStartPre=-/usr/bin/vncserver -kill :%i > /dev/null 2>&1
ExecStart=/usr/bin/vncserver -geometry 1280x720 -depth 24 :%i
ExecStop=/usr/bin/vncserver -kill :%i

[Install]
WantedBy=multi-user.target
EOF'

# your-username을 실제 사용자명으로 변경 후
sudo systemctl daemon-reload
sudo systemctl enable vncserver@1
sudo systemctl start vncserver@1
```

### 실습 0-5: 시스템 모니터링 도구

```bash
# tegrastats — 실시간 GPU/CPU/온도/전력 모니터링
sudo tegrastats

# 출력 예시:
# RAM 1776/3956MB (lfb 1147x4MB) IRAM 0/0kB 
# CPU [6%@1190,8%@1190,8%@1190,1%@1190] 
# GPU 0%@110 
# Tboard@43.5C Tdiode@50.5C 
# VDD_IN 3153/3161 VDD_CPU 128/153 VDD_GPU 57/86

# jetson_clocks — 모든 클럭 최대 성능 고정
sudo jetson_clocks
sudo jetson_clocks --show  # 현재 클럭 상태 출력

# nvidia-smi — GPU 프로세스 확인
nvidia-smi
# (Jetson Nano의 경우 nvidia-smi 대신这部分 명령어 사용)
```

### 실습 0-6: Python 개발 환경

```bash
# Python3 확인
python3 --version

# pip 설치
sudo apt install -y python3-pip python3-dev

# 가상 환경 도구 설치
sudo apt install -y python3-venv

# 가상 환경 생성 및 활성화
python3 -m venv jetson_env
source jetson_env/bin/activate

# 필수 패키지 설치
pip install --upgrade pip
pip install numpy matplotlib jupyter

# Jetson.GPIO 설치
sudo apt install -y python3-jetson-gpio
# 또는 pip로 설치
pip install Jetson.GPIO

# GPIO 접근 권한 설정
sudo groupadd -f gpio
sudo usermod -aG gpio $USER
# 99-gpio.rules 파일 생성
sudo bash -c 'cat > /etc/udev/rules.d/99-gpio.rules << "EOF"
SUBSYSTEM=="gpio", KERNEL=="gpiochip*", ACTION=="add", PROGRAM="/bin/sh -c '\''chown root:gpio /dev/gpiochip* && chmod 660 /dev/gpiochip*'\''"
SUBSYSTEM=="gpio", KERNEL=="gpio*", ACTION=="add", PROGRAM="/bin/sh -c '\''chown root:gpio /sys/class/gpio/export || true'\''"
EOF'

# I2C 도구 설치
sudo apt install -y i2c-tools libi2c-dev

# SPI 도구 설치
sudo apt install -y spidev-tools

# 재부팅 (그룹 권한 적용)
sudo reboot
```

### 실습 0-7: 크로스 컴파일 환경 (호스트 PC)

호스트 PC에서 Jetson Nano용 바이너리를 컴파일하는 방법입니다.

```bash
# 호스트 PC (x86_64 Ubuntu)에서:
# 1. L4T 크로스 컴파일 도구체인 다운로드
# https://developer.nvidia.com/embedded/downloads

# 2. aarch64 크로스 컴파일러 설치
sudo apt install -y gcc-aarch64-linux-gnu g++-aarch64-linux-gnu

# 3. 테스트 코드 (hello.c)
cat > hello.c << 'EOF'
#include <stdio.h>

int main() {
    printf("Hello from Jetson Nano!\n");
    return 0;
}
EOF

# 4. 크로스 컴파일
aarch64-linux-gnu-gcc -o hello hello.c

# 5. 파일 확인
file hello
# 출력: hello: ELF 64-bit LSB executable, ARM aarch64, ...

# 6. Jetson Nano로 복사 후 실행
scp hello username@jetson-ip:~/
ssh username@jetson-ip './hello'
```

---

## 4. 최종 실습 — 시스템 헬스 체크 스크립트

**파일명**: `system_health_check.py`

```python
#!/usr/bin/env python3
"""
Jetson Nano 시스템 헬스 체크 스크립트
CPU/GPU/온도/메모리/디스크 정보를 수집하여 리포트 출력

실행: python3 system_health_check.py
"""

import subprocess
import re
import platform
import time
from datetime import datetime


def run_cmd(cmd):
    """쉘 명령어 실행 후 stdout 반환"""
    try:
        result = subprocess.run(cmd, shell=True, capture_output=True,
                                text=True, timeout=5)
        return result.stdout.strip()
    except Exception as e:
        return f"Error: {e}"


def get_cpu_info():
    """CPU 정보 수집"""
    info = {}
    # CPU 코어 수
    cpu_count = run_cmd("nproc")
    info["cores"] = cpu_count

    # CPU 아키텍처
    arch = run_cmd("uname -m")
    info["architecture"] = arch

    # CPU 사용률 (1초 평균)
    cpu_usage = run_cmd("top -bn1 | grep 'Cpu(s)' | awk '{print $2}'")
    info["usage_percent"] = cpu_usage

    # CPU 온도
    try:
        with open("/sys/devices/virtual/thermal/thermal_zone1/temp", "r") as f:
            temp_raw = int(f.read().strip())
            info["temperature_c"] = temp_raw / 1000.0
    except:
        info["temperature_c"] = "N/A"

    return info


def get_gpu_info():
    """GPU 정보 수집"""
    info = {}
    # GPU 사용률 (tegrastats 파싱)
    tegrastats_out = run_cmd(
        "timeout 2 tegrastats | tail -1"
    )
    gpu_match = re.search(r'GPU\s*(\d+)%', tegrastats_out)
    if gpu_match:
        info["usage_percent"] = gpu_match.group(1)
    else:
        info["usage_percent"] = "N/A"

    return info


def get_memory_info():
    """메모리 정보 수집"""
    info = {}
    mem_total = run_cmd("free -m | grep Mem | awk '{print $2}'")
    mem_used = run_cmd("free -m | grep Mem | awk '{print $3}'")
    mem_free = run_cmd("free -m | grep Mem | awk '{print $4}'")

    info["total_mb"] = mem_total
    info["used_mb"] = mem_used
    info["free_mb"] = mem_free

    # Swap 정보 (Module 0 실습에서 추가한 swap 확인)
    swap_total = run_cmd("free -m | grep Swap | awk '{print $2}'")
    swap_used = run_cmd("free -m | grep Swap | awk '{print $3}'")
    info["swap_total_mb"] = swap_total
    info["swap_used_mb"] = swap_used

    return info


def get_disk_info():
    """디스크 정보 수집"""
    info = {}
    df_out = run_cmd("df -h / | tail -1")
    parts = df_out.split()
    if len(parts) >= 4:
        info["size"] = parts[1]
        info["used"] = parts[2]
        info["available"] = parts[3]
        info["usage_percent"] = parts[4]
    return info


def get_power_mode():
    """현재 파워 모드 확인"""
    mode = run_cmd("nvpmodel --query 2>/dev/null | grep -o 'MAXN\\|5W\\|10W'")
    return mode if mode else "N/A"


def print_report(cpu, gpu, mem, disk, power_mode):
    """보고서 출력"""
    separator = "=" * 56
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    print(f"""
{separator}
  NVIDIA Jetson Nano — System Health Report
  {timestamp}
{separator}

[CPU]
  Architecture  : {cpu['architecture']}
  Cores         : {cpu['cores']}
  Usage         : {cpu['usage_percent']}%
  Temperature   : {cpu['temperature_c']}°C

[GPU]
  Usage         : {gpu['usage_percent']}%

[Memory]
  RAM           : {mem['used_mb']} / {mem['total_mb']} MB
  Free          : {mem['free_mb']} MB
  Swap          : {mem['swap_used_mb']} / {mem['swap_total_mb']} MB

[Disk]
  Rootfs        : {disk['used']} / {disk['size']} ({disk['usage_percent']})
  Available     : {disk['available']}

[Power]
  Mode          : {power_mode}

{separator}
""")

    # 경고 조건
    warnings = []
    if isinstance(cpu['temperature_c'], (int, float)) and cpu['temperature_c'] > 75:
        warnings.append(f"⚠ CPU 온도가 높습니다: {cpu['temperature_c']}°C (권장: 75°C 이하)")
    if isinstance(mem['free_mb'], str) and mem['free_mb'].isdigit():
        if int(mem['free_mb']) < 200:
            warnings.append(f"⚠ RAM 여유 공간 부족: {mem['free_mb']}MB")
    if isinstance(disk['usage_percent'], str) and disk['usage_percent'].replace('%', '').isdigit():
        if int(disk['usage_percent'].replace('%', '')) > 85:
            warnings.append(f"⚠ 디스크 사용률 높음: {disk['usage_percent']}")

    if warnings:
        print("=" * 56)
        print("  WARNINGS")
        print("=" * 56)
        for w in warnings:
            print(f"  {w}")
        print()


def main():
    print("Collecting system information...")

    cpu = get_cpu_info()
    gpu = get_gpu_info()
    mem = get_memory_info()
    disk = get_disk_info()
    power_mode = get_power_mode()

    print_report(cpu, gpu, mem, disk, power_mode)


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 system_health_check.py
```

**예상 출력**:
```
========================================================
  NVIDIA Jetson Nano — System Health Report
  2026-05-10 14:30:22
========================================================

[CPU]
  Architecture  : aarch64
  Cores         : 4
  Usage         : 12%
  Temperature   : 48.5°C

[GPU]
  Usage         : 0%

[Memory]
  RAM           : 892 / 3956 MB
  Free          : 2532 MB
  Swap          : 0 / 4096 MB

[Disk]
  Rootfs        : 8.2G / 29G (32%)
  Available     : 19G

[Power]
  Mode          : MAXN

========================================================
```

---

## 5. 확인 및 테스트

### 모듈 00 완료 체크리스트

| 항목 | 확인 명령어 | 기대 결과 |
|------|-----------|----------|
| SSH 접속 | `ssh user@jetson-ip` | 로그인 성공 |
| VNC 접속 | VNC Viewer로 `jetson-ip:5901` | 데스크탑 화면 출력 |
| CUDA 확인 | `nvcc --version` | 버전 정보 출력 |
| Python3 | `python3 --version` | 3.6 이상 출력 |
| GPIO 라이브러리 | `python3 -c "import Jetson.GPIO; print('OK')"` | `OK` 출력 |
| I2C 도구 | `i2cdetect -y -r 1` | 오류 없이 실행 |
| 시스템 헬스 체크 | `python3 system_health_check.py` | 온도, 메모리, 디스크 정보 출력 |

### 문제 해결

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| 부팅 안 됨 | SD 카드 불량/이미지 손상 | SD 카드 다시 플래싱, 검증된 SD 카드 사용 |
| SSH 접속 실패 | SSH 서비스 미실행 | `sudo systemctl start ssh` |
| VNC 회색 화면 | xstartup 설정 오류 | `~/.vnc/xstartup` 스크립트 확인 |
| `Jetson.GPIO` Permission denied | GPIO 그룹 권한 없음 | 사용자를 gpio 그룹에 추가 후 재부팅 |
| `tegrastats` command not found | JetPack 버전 문제 | `sudo apt install -y tegrastats` 재설치 |
| 전원 부족 | 5V⎓2A 이하 어댑터 사용 | 5V⎓4A 어댑터로 교체 (DC 배럴 잭 권장) |

---

> **다음 모듈**: Module 01 — GPIO 디지털 입출력 (LED, 버튼, 인터럽트)

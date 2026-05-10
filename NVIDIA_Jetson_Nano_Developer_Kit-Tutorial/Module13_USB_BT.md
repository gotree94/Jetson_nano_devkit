# Module 13 — USB & 블루투스

> **소요 시간**: 3~4시간  
> **난이도**: ★★★☆☆  
> **준비물**: USB 조이스틱, BLE 지원 스마트폰, HC-06 블루투스 모듈

---

## 1. 개요

Jetson Nano의 USB 호스트/디바이스 모드와 블루투스 통신을 다룹니다.

### 학습 목표
- USB Gadget 모드로 Jetson을 키보드/시리얼 장치로 에뮬레이션할 수 있다
- USB HID 장치(조이스틱) 입력을 처리할 수 있다
- BLE로 스마트폰과 데이터를 주고받을 수 있다

---

## 2. 실습 코드

### 실습 13-1: USB Serial Gadget

```bash
# USB Gadget 설정 (Jetson을 USB 시리얼 장치로)
sudo modprobe libcomposite
cd /sys/kernel/config/usb_gadget
mkdir -p g1 && cd g1
echo 0x1d6b > idVendor   # Linux Foundation
echo 0x0104 > idProduct  # Multifunction Composite Gadget
echo 0x0100 > bcdDevice
echo 0x0200 > bcdUSB

mkdir -p strings/0x409
echo "deadbeef12345678" > strings/0x409/serialnumber
echo "NVIDIA" > strings/0x409/manufacturer
echo "Jetson Nano Serial" > strings/0x409/product

mkdir -p functions/acm.usb0
mkdir -p configs/c.1
ln -s functions/acm.usb0 configs/c.1

# UDC 드라이버 확인 후 연결
ls /sys/class/udc/
echo "700d0000.usb" > UDC  # UDC 이름은 환경에 따라 다름
```

연결 후 PC에서 `/dev/ttyACM0`으로 접속 가능:
```bash
screen /dev/ttyACM0 115200
```

### 실습 13-2: 조이스틱 입력

**코드**: `joystick_input.py`
```python
#!/usr/bin/env python3
"""
USB 조이스틱 입력 처리 (pygame)
"""

import pygame
import sys

pygame.init()
pygame.joystick.init()

if pygame.joystick.get_count() == 0:
    print("조이스틱이 연결되지 않았습니다")
    sys.exit(1)

joy = pygame.joystick.Joystick(0)
joy.init()

print(f"조이스틱: {joy.get_name()}")
print(f"축: {joy.get_numaxes()}, 버튼: {joy.get_numbuttons()}")

try:
    while True:
        pygame.event.pump()
        axes = [joy.get_axis(i) for i in range(joy.get_numaxes())]
        buttons = [joy.get_button(i) for i in range(joy.get_numbuttons())]
        print(f"\r축: {[f'{a:5.2f}' for a in axes]}  "
              f"버튼: {[f'{b}' for b in buttons]}", end='', flush=True)
        pygame.time.wait(50)
except KeyboardInterrupt:
    print("\n종료")
    pygame.quit()
```

### 실습 13-3: BLE 환경 센서 광고

**코드**: `ble_sensor_advertise.py`
```python
#!/usr/bin/env python3
"""
BLE Advertisement로 센서 데이터 브로드캐스트
스마트폰 nRF Connect 앱으로 수신 가능
"""

import socket
import struct
import time
from smbus2 import SMBus

# HCI 소켓으로 BLE Advertisement 전송
def ble_advertise(temp, hum):
    """BLE Advertisement 패킷 전송"""
    try:
        sock = socket.socket(socket.AF_BLUETOOTH, socket.SOCK_RAW,
                             socket.BTPROTO_HCI)
        sock.setsockopt(socket.SOL_HCI, socket.HCI_FILTER, struct.pack("I", 0))
        sock.settimeout(1)

        # Manufacturer Specific Data (NVIDIA Company ID: 0x0955)
        company_id = 0x0955
        payload = struct.pack('<H', company_id) + struct.pack('<hh', int(temp*100), int(hum*100))

        # HCI LE Advertising command
        hci_cmd = struct.pack('BBBBB', 0x01, 0x00, len(payload)+3, 0x02, 0x01) + \
                  struct.pack('B', 0xFF) + payload

        sock.send(hci_cmd)
        sock.close()
        return True
    except Exception as e:
        return False

# 메인 루프
sensor = None
try:
    sensor = SMBus(1)
    while True:
        # AHT20 읽기 (생략)
        ble_advertise(25.5, 60.0)  # 예시 데이터
        time.sleep(1)
except KeyboardInterrupt:
    pass
```

---

> **다음 모듈**: Module 14 — CAN 통신

# Module 14 — CAN 통신

> **소요 시간**: 3~4시간  
> **난이도**: ★★★★☆  
> **준비물**: MCP2515 CAN 모듈 + SN65HVD230 CAN 트랜시버(2세트, 2대의 Jetson 또는 PC 필요)

---

## 1. 개요

CAN(Controller Area Network)은 자동차 및 산업용 장비에서 널리 사용되는 견고한 통신 프로토콜입니다.  
Jetson Nano는 SoC에 CAN 컨트롤러가 있으나 40-pin 헤더로 노출되지 않아, MCP2515(SPI CAN)를 사용합니다.

### 학습 목표
- CAN 프로토콜 프레임 구조를 이해한다
- MCP2515 CAN 컨트롤러를 SPI로 제어할 수 있다
- Linux SocketCAN 인터페이스를 설정할 수 있다
- CAN 메시지를 송수신할 수 있다
- OBD-II로 차량 데이터를 읽을 수 있다

---

## 2. 실습 코드

### 실습 14-1: MCP2515 + SocketCAN 설정

**회로 구성**:
```
MCP2515 + SN65HVD230:
  VCC ──→ 3.3V
  GND ──→ GND
  SCK ──→ Pin 23 (SPI_0_SCLK)
  SI  ──→ Pin 19 (SPI_0_MOSI)
  SO  ──→ Pin 21 (SPI_0_MISO)
  CS  ──→ Pin 24 (SPI_0_CS1)
  INT ──→ Pin 26 (GPIO)

  CANH ──→ 다른 노드의 CANH
  CANL ──→ 다른 노드의 CANL
  (120Ω 종단 저항 필수!)
```

```bash
# Device Tree Overlay로 MCP2515 활성화
# 또는 직접 SPI + GPIO 제어

# SocketCAN 인터페이스 설정
sudo modprobe can
sudo modprobe can_raw
sudo ip link set can0 up type can bitrate 250000
sudo ifconfig can0 up

# 확인
ip -details link show can0
```

### 실습 14-2: CAN 메시지 송수신

**코드**: `can_communicate.py`
```python
#!/usr/bin/env python3
"""
CAN 통신 — SocketCAN send/receive
"""

import socket
import struct
import time
import threading

# CAN 소켓 생성
class CANInterface:
    def __init__(self, interface='can0'):
        self.sock = socket.socket(socket.AF_CAN, socket.SOCK_RAW,
                                  socket.CAN_RAW)
        self.interface = interface

    def open(self):
        try:
            self.sock.bind((self.interface,))
            return True
        except OSError as e:
            print(f"CAN bind 오류: {e}")
            return False

    def send(self, can_id, data):
        """CAN 메시지 전송"""
        flags = 0  # 표준 프레임 (11비트 ID)
        can_frame = struct.pack('IB3x8s', can_id, len(data), bytes(data))
        self.sock.send(can_frame)

    def recv(self, timeout=1.0):
        """CAN 메시지 수신"""
        self.sock.settimeout(timeout)
        try:
            frame = self.sock.recv(16)
            can_id, length, data = struct.unpack('IB3x8s', frame)
            return can_id, list(data[:length])
        except socket.timeout:
            return None, None

    def close(self):
        self.sock.close()

# 송신 스레드
def send_thread(can):
    count = 0
    while True:
        data = [count & 0xFF, (count >> 8) & 0xFF, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00]
        can.send(0x123, data)
        print(f"[송신] ID=0x123 data={data[:4]}")
        count += 1
        time.sleep(1)

# 수신 스레드
def recv_thread(can):
    while True:
        can_id, data = can.recv(2.0)
        if can_id is not None:
            print(f"[수신] ID=0x{can_id:x} data={data}")

def main():
    can = CANInterface('can0')
    if not can.open():
        print("CAN 인터페이스를 열 수 없습니다.")
        print("sudo ip link set can0 up type can bitrate 250000")
        return

    t1 = threading.Thread(target=send_thread, args=(can,), daemon=True)
    t2 = threading.Thread(target=recv_thread, args=(can,), daemon=True)
    t1.start()
    t2.start()

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("\n종료")
        can.close()

if __name__ == "__main__":
    main()
```

---

> **다음 모듈**: Module 15 — CUDA 기초

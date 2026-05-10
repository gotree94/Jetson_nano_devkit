# Module 04 — SPI (Serial Peripheral Interface)

> **소요 시간**: 3~4시간  
> **난이도**: ★★☆☆☆  
> **준비물**: MCP3008 ADC(아날로그 입력), CDS 조도 센서, 10kΩ 저항, MAX7219 8x8 LED Matrix 모듈, RC522 RFID 리더(선택)

---

## 1. 개요

SPI(Serial Peripheral Interface)는 I2C보다 빠른 전이중(Full-Duplex) 동기식 직렬 통신 프로토콜입니다.  
ADC 변환, LED 매트릭스, RFID 리더 등 고속 데이터 전송이 필요한 장치에 사용됩니다.

### 학습 목표
- SPI 프로토콜의 4선(MOSI/MISO/SCLK/CS)과 4가지 모드를 이해한다
- Jetson Nano의 SPI 버스를 활성화하고 `spidev`로 제어할 수 있다
- MCP3008 ADC로 아날로그 신호를 디지털로 변환할 수 있다
- MAX7219로 LED 매트릭스를 제어할 수 있다
- RC522 RFID 리더로 카드 UID를 읽을 수 있다

### 사전 지식
- Module 01의 GPIO 개념
- 아날로그 vs 디지털 신호 차이 이해

---

## 2. 이론

### 2.1 SPI 프로토콜

SPI는 4개의 신호선을 사용합니다:

| 신호 | 이름 | 설명 |
|------|------|------|
| **MOSI** | Master Out Slave In | Master → Slave 데이터 |
| **MISO** | Master In Slave Out | Slave → Master 데이터 |
| **SCLK** | Serial Clock | Master가 생성하는 클럭 |
| **CS/SS** | Chip Select / Slave Select | Slave 선택 (Active LOW) |

**SPI 버스 구성**:
```
Jetson Nano (Master)         Slave 1           Slave 2
                          ┌──────────┐    ┌──────────┐
  MOSI ───────────────────┤ MOSI     │    │ MOSI     │
  MISO ───────────────────┤ MISO     │    │ MISO     │
  SCLK ───────────────────┤ SCLK     │    │ SCLK     │
  CS1  ───────────────────┤ CS       │    │          │
  CS2  ───────────────────┤          │    │ CS       │
                          └──────────┘    └──────────┘
```

**SPI Mode (CPOL / CPHA)**:

| Mode | CPOL | CPHA | 클럭 극성 | 데이터 샘플링 |
|------|------|------|----------|-------------|
| 0 | 0 | 0 | Idle LOW | 첫 번째 엣지(상승) |
| 1 | 0 | 1 | Idle LOW | 두 번째 엣지(하강) |
| 2 | 1 | 0 | Idle HIGH | 첫 번째 엣지(하강) |
| 3 | 1 | 1 | Idle HIGH | 두 번째 엣지(상승) |

> 대부분의 장치는 **Mode 0**을 사용합니다 (MCP3008, MAX7219).

**SPI vs I2C 비교**:

| 항목 | SPI | I2C |
|------|-----|-----|
| 신호선 | 4개 (MOSI/MISO/SCLK/CS) | 2개 (SDA/SCL) |
| 속도 | 수십 Mbps | 최대 3.4Mbps |
| 통신 방식 | 전이중 (Full-Duplex) | 반이중 (Half-Duplex) |
| Slave 선택 | 별도 CS 핀 | 주소 지정 |
| 장치 주소 | 불필요 | 7비트 주소 필요 |
| 거리 | 짧음 (보드 내) | 중간 (1m 내외) |

### 2.2 Jetson Nano SPI 버스

Jetson Nano는 **2개의 SPI 버스**를 제공합니다:

| 버스 | 장치 파일 | 40-pin 핀 | 비고 |
|------|----------|-----------|------|
| **SPI_0** | `/dev/spidev0.0` | CS0: Pin 26, CS1: Pin 24 | MOSI: Pin 19, MISO: Pin 21, SCLK: Pin 23 |
| **SPI_1** | `/dev/spidev1.0` | CS0: Pin 18 | MOSI: Pin 37, MISO: Pin 22, SCLK: Pin 13 |

**SPI 활성화** (JetPack 기본 활성화됨, 확인):
```bash
# SPI 장치 파일 확인
ls -l /dev/spidev*

# 출력 예시:
# crw-rw-rw- 1 root root 153, 0 May 10  2026 /dev/spidev0.0
# crw-rw-rw- 1 root root 153, 1 May 10  2026 /dev/spidev0.1
# crw-rw-rw- 1 root root 153, 2 May 10  2026 /dev/spidev1.0
```

만약 `/dev/spidev*`가 없으면 Jetson-IO로 활성화:
```bash
sudo /opt/nvidia/jetson-io/jetson-io.py
# → "Configure 40-pin expansion header" → SPI 활성화
```

### 2.3 spidev 라이브러리

```python
import spidev

# SPI 장치 열기
spi = spidev.SpiDev()
spi.open(0, 0)  # bus=0, device=0 → /dev/spidev0.0

# 설정
spi.max_speed_hz = 1000000   # 1MHz
spi.mode = 0b00              # Mode 0 (CPOL=0, CPHA=0)
spi.bits_per_word = 8        # 8비트 전송
spi.lsbfirst = False         # MSB 우선

# 데이터 송수신 (동시에 이루어짐)
data_out = [0x01, 0x02, 0x03]
data_in = spi.xfer2(data_out)  # 전이중 통신

# 단일 바이트 전송 후 응답 읽기
response = spi.xfer2([0x00])  # 1바이트 전송 → 1바이트 수신

# 장치 닫기
spi.close()
```

> `xfer()` vs `xfer2()`: `xfer2()`는 CS를 한 번만 토글 (연속 전송),  
> `xfer()`는 매 바이트마다 CS 토글.

### 2.4 MCP3008 ADC

MCP3008은 8채널 10비트 ADC(아날로그-디지털 변환기)입니다.

| 특성 | 값 |
|------|-----|
| 채널 | 8 (CH0~CH7) |
| 분해능 | 10비트 (0~1023) |
| 인터페이스 | SPI |
| 전압 범위 | 0~VDD (보통 3.3V 또는 5V) |
| 샘플링 속도 | 최대 200ksps |

**데이터시트 기준 SPI 명령어**:
```
Byte 1: Start bit (1) + Single/Diff(1) + 채널(3비트) + xxxx
        1         1           CHS2 CHS1 CHS0  0 0 0 0

Byte 2: 수신 데이터 상위 2비트(XXX) + Null bit + 변환 데이터 상위 5비트
Byte 3: 변환 데이터 하위 8비트

수신 데이터:
  10비트 = (Byte2 & 0x1F) << 8 | Byte3
```

**MCP3008 회로 구성**:
```
MCP3008 (DIP):
  CH0 ──→ CDS 조도 센서 분압 회로
  CH1 ──→ 가변 저항 (10kΩ)
  CH2~7 ──→ 다른 아날로그 센서

  VDD ──→ 3.3V
  VREF ──→ 3.3V (기준 전압)
  AGND ──→ GND
  DGND ──→ GND
  CLK  ──→ SCLK (Jetson Pin 23)
  DOUT ──→ MISO (Jetson Pin 21)
  DIN  ──→ MOSI (Jetson Pin 19)
  CS   ──→ CS0 (Jetson Pin 26)
```

### 2.5 MAX7219 LED Matrix

MAX7219는 8x8 LED 매트릭스(또는 7-Segment)를 구동하는 LED 드라이버 칩입니다.

| 특성 | 값 |
|------|-----|
| 인터페이스 | SPI (16비트 패킷) |
| 구동 방식 | 다이나믹 스캐닝 |
| 밝기 제어 | 16단계 (PWM) |
| 데이지 체인 | 여러 모듈 직렬 연결 가능 |

**명령어 포맷** (16비트):
```
[D15 D14 D13 D12 D11 D10 D9 D8] [D7 D6 D5 D4 D3 D2 D1 D0]
 ────────── 상위 바이트 ──────────  ────────── 하위 바이트 ──────────
 상위 8비트 = 명령어/주소
 하위 8비트 = 데이터
```

**주요 레지스터**:
| 주소 | 이름 | 설명 |
|------|------|------|
| 0x00 | No-Op | 데이지 체인 통과 |
| 0x09 | Decode Mode | BCD 디코딩 (LED Matrix: 0x00) |
| 0x0A | Intensity | 밝기 (0x00~0x0F, 16단계) |
| 0x0B | Scan Limit | 스캔 라인 수 (LED Matrix: 0x07) |
| 0x0C | Shutdown | 0=전원 OFF, 1=정상 동작 |
| 0x0F | Display Test | 0x01=모든 LED ON |
| 0x01~0x08 | Digit 0~7 | 각 행(Row)의 LED 패턴 |

**초기화 순서**:
1. Shutdown 해제 (0x0C → 0x01)
2. Decode Mode 설정 (0x09 → 0x00, No-decode)
3. Scan Limit 설정 (0x0B → 0x07, 8줄 전체)
4. Intensity 설정 (0x0A → 0x08, 중간 밝기)
5. Display Test OFF (0x0F → 0x00)

---

## 3. 실습 코드

### 실습 4-1: SPI 기본 통신 테스트

**코드**: `spi_test.py`
```python
#!/usr/bin/env python3
"""
SPI 기본 통신 테스트
spidev 열기/닫기, 설정 출력, 루프백 테스트
"""

import spidev


def list_spi_devices():
    """사용 가능한 SPI 장치 확인"""
    import os
    devices = []
    for entry in os.listdir('/dev/'):
        if entry.startswith('spidev'):
            devices.append(f"/dev/{entry}")
    return devices


def test_spi_loopback(bus, device):
    """
    SPI 루프백 테스트 (MOSI → MISO 연결 필요)
    송신한 데이터가 그대로 돌아오는지 확인
    """
    spi = spidev.SpiDev()
    spi.open(bus, device)

    print(f"\nSPI 테스트: /dev/spidev{bus}.{device}")
    print(f"  최대 속도: {spi.max_speed_hz} Hz")
    print(f"  모드: {bin(spi.mode)}")

    # 루프백 테스트
    test_data = [0x55, 0xAA, 0x00, 0xFF, 0x12, 0x34, 0xAB, 0xCD]
    received = spi.xfer2(test_data)

    print(f"  송신: {[hex(d) for d in test_data]}")
    print(f"  수신: {[hex(d) for d in received]}")

    if test_data == received:
        print("  ✅ 루프백 성공 (MOSI ←→ MISO 연결됨)")
    else:
        print("  ⚠️  루프백 실패 (MOSI-MISO 미연결 상태 = 정상)")

    spi.close()


def main():
    print("SPI 장치 확인")
    devices = list_spi_devices()
    if not devices:
        print("SPI 장치가 없습니다. Jetson-IO로 활성화하세요.")
        print("  sudo /opt/nvidia/jetson-io/jetson-io.py")
        return

    for dev in devices:
        print(f"  발견: {dev}")

    # SPI 0.0 테스트 (MCP3008가 연결된 버스)
    test_spi_loopback(0, 0)

    # SPI 0.1 테스트 (MAX7219가 연결된 버스)
    test_spi_loopback(0, 1)


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 spi_test.py
```

### 실습 4-2: MCP3008 ADC — 조도 센서 읽기

**회로 구성 (CDS 조도 센서 분압 회로)**:
```
 3.3V
   │
   ├── CDS 조도 센서
   │
   ┌┴┐
   │ │ 10kΩ 고정 저항
   └┬┘
   │
   ├── MCP3008 CH0
   │
  GND
```

**코드**: `mcp3008_light.py`
```python
#!/usr/bin/env python3
"""
MCP3008 8채널 ADC — 조도 센서 읽기
CDS 센서(또는 가변 저항)의 아날로그 값을 SPI로 읽어 출력
"""

import spidev
import time


class MCP3008:
    """MCP3008 10비트 ADC 드라이버"""

    def __init__(self, bus=0, device=0):
        self.spi = spidev.SpiDev()
        self.spi.open(bus, device)
        self.spi.max_speed_hz = 1000000  # 1MHz
        self.spi.mode = 0b00
        self.spi.bits_per_word = 8

    def read_channel(self, channel):
        """
        지정 채널의 아날로그 값 읽기 (10비트, 0~1023)

        채널: 0~7
        """
        if channel < 0 or channel > 7:
            raise ValueError("채널은 0~7 사이여야 합니다")

        # SPI 명령어 구성
        # Start=1, Single=1, 채널=3비트
        command = 0b00000001  # Start bit
        command = (command << 1) | 0b1  # Single mode
        command = (command << 3) | (channel & 0x07)

        # 3바이트 전송
        # Byte 1: 명령어
        # Byte 2: 0x00 (Dummy)
        # Byte 3: 0x00 (Dummy)
        data = [command, 0x00, 0x00]
        result = self.spi.xfer2(data)

        # 결과 추출: 10비트 결합
        # result[0] = 응답 (무시)
        # result[1] 하위 1비트 + result[2] 상위 1비트 (Null bit)
        # 실제 변환값: result[1] & 0x1F 상위 5비트 + result[2] 하위 8비트
        value = ((result[1] & 0x1F) << 8) | result[2]

        return value

    def read_voltage(self, channel, vref=3.3):
        """전압 값으로 읽기"""
        raw = self.read_channel(channel)
        voltage = raw * vref / 1023.0
        return raw, voltage

    def close(self):
        self.spi.close()


def main():
    print("MCP3008 ADC — 조도 센서 읽기")
    print("=" * 40)

    try:
        adc = MCP3008(bus=0, device=0)
    except FileNotFoundError:
        print("SPI 장치를 찾을 수 없습니다.")
        print("sudo /opt/nvidia/jetson-io/jetson-io.py 로 SPI 활성화 필요")
        return

    print(f"{'시간':>8} {'CH0(조도)':>10} {'전압(V)':>10} {'밝기(%)':>10}")
    print("-" * 40)

    try:
        while True:
            raw, voltage = adc.read_voltage(0, vref=3.3)
            # 밝기 백분율 (어두울수록 0%, 밝을수록 100%)
            brightness = raw / 1023.0 * 100.0

            # 시각화
            bar = '█' * int(brightness / 5) + '░' * (20 - int(brightness / 5))

            now = time.strftime("%H:%M:%S")
            print(f"{now}  {raw:4d} ({raw:04X})  {voltage:6.3f}V  "
                  f"{brightness:5.1f}%  {bar}")

            time.sleep(0.5)
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        adc.close()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 mcp3008_light.py
```

손으로 CDS 센서를 가리면 값이 낮아지고, 밝게 비추면 값이 올라가는 것을 확인합니다.

### 실습 4-3: MAX7219 8x8 LED Matrix

**회로 구성**:
```
MAX7219 LED Matrix 모듈:
  VCC ──→ 5V (Jetson Pin 2, 모듈은 5V 필요)
  GND ──→ GND
  DIN ──→ MOSI (Jetson Pin 19)
  CS  ──→ CS1 (Jetson Pin 24, SPI_0.1)
  CLK ──→ SCLK (Jetson Pin 23)

※ MAX7219 모듈 레벨 시프터 내장 (5V/3.3V 호환)
```

**코드**: `max7219_matrix.py`
```python
#!/usr/bin/env python3
"""
MAX7219 8x8 LED Matrix 제어 (SPI)
도트 매트릭스에 패턴 및 텍스트 표시
"""

import spidev
import time


class MAX7219:
    """MAX7219 LED 매트릭스 드라이버 (8x8)"""

    # 레지스터 주소
    REG_NOOP = 0x00
    REG_DECODE_MODE = 0x09
    REG_INTENSITY = 0x0A
    REG_SCAN_LIMIT = 0x0B
    REG_SHUTDOWN = 0x0C
    REG_DISPLAY_TEST = 0x0F
    # Digit 0~7: 0x01 ~ 0x08 (LED Matrix에서는 Row 0~7)

    def __init__(self, bus=0, device=1, count=1):
        """
        bus: SPI 버스
        device: SPI 장치 (CS)
        count: 데이지 체인 모듈 수
        """
        self.spi = spidev.SpiDev()
        self.spi.open(bus, device)
        self.spi.max_speed_hz = 10000000  # 10MHz
        self.spi.mode = 0b00
        self.spi.lsbfirst = False
        self.count = count
        self._init_display()

    def _spi_write(self, address, data):
        """
        MAX7219에 16비트 명령 쓰기
        address: 8비트 주소
        data: 8비트 데이터
        """
        self.spi.xfer2([address, data])

    def _spi_write_all(self, address, data):
        """
        모든 모듈에 동일 명령 쓰기 (데이지 체인)
        """
        for _ in range(self.count):
            self.spi.xfer2([address, data])

    def _init_display(self):
        """MAX7219 초기화"""
        self._spi_write_all(self.REG_SHUTDOWN, 0x01)      # 정상 동작
        self._spi_write_all(self.REG_DECODE_MODE, 0x00)   # No decode
        self._spi_write_all(self.REG_SCAN_LIMIT, 0x07)    # 8줄 전체
        self._spi_write_all(self.REG_INTENSITY, 0x04)     # 밝기 (0~15)
        self._spi_write_all(self.REG_DISPLAY_TEST, 0x00)  # 테스트 모드 OFF
        self.clear()

    def set_intensity(self, level):
        """밝기 설정 (0~15)"""
        self._spi_write_all(self.REG_INTENSITY, max(0, min(15, level)))

    def clear(self):
        """모든 LED OFF"""
        for row in range(8):
            self._spi_write_all(self.REG_NOOP if self.count > 1 else 0x01 + row, 0x00)

    def set_row(self, row, pattern, module=0):
        """
        특정 행(row)에 패턴 표시

        row: 0~7
        pattern: 8비트 패턴 (bit 7=LED7, ..., bit 0=LED0)
        module: 데이지 체인 모듈 인덱스
        """
        if row < 0 or row > 7:
            return
        reg = 0x01 + row  # Digit 레지스터 (0x01~0x08)

        if self.count == 1:
            self._spi_write(reg, pattern)
        else:
            # 데이지 체인 모드
            for m in range(self.count):
                if m == module:
                    self.spi.xfer2([reg, pattern])
                else:
                    self.spi.xfer2([self.REG_NOOP, 0x00])

    def set_pixel(self, x, y, on=True):
        """
        개별 픽셀 설정

        x: 0~7 (열, 좌→우)
        y: 0~7 (행, 상→하)
        on: True=ON, False=OFF
        """
        # 현재 행 패턴 읽기 (소프트웨어적으로 유지)
        # MAX7219는 읽기 기능이 없으므로 shadow buffer 사용 필요
        # 여기서는 간단히 단일 픽셀만 설정할 수 없음
        # → set_row로 행 전체를 다시 써야 함
        pass

    def set_pattern(self, pattern_matrix):
        """
        8x8 패턴 매트릭스 표시

        pattern_matrix: 8바이트 리스트 (각 행의 비트 패턴)
        """
        for row in range(8):
            self.set_row(row, pattern_matrix[row])

    def show_char(self, char_patterns, char):
        """ASCII 문자 표시 (미리 정의된 폰트 사용)"""
        # luma.oled의 폰트를 재사용할 수도 있지만,
        # 여기서는 하드코딩된 8x8 폰트 사용
        font_8x8 = FONT_8x8.get(char, FONT_8x8[' '])
        self.set_pattern(font_8x8)

    def scroll_text(self, text, font_dict, delay=0.1):
        """텍스트 스크롤 표시"""
        import copy

        # 전체 텍스트의 비트맵 생성
        total_width = len(text) * 8
        bitmap = [0x00] * 8  # 8개 행

        for char_idx, char in enumerate(text):
            char_bitmap = font_dict.get(char, font_dict[' '])
            for row in range(8):
                bitmap[row] = (bitmap[row] << 8) | char_bitmap[row]

        # 스크롤
        for offset in range(total_width):
            for row in range(8):
                # 8비트 윈도우로 슬라이스
                mask = (0xFF << (total_width - offset - 8)) if total_width - offset >= 8 else 0xFF
                shifted = (bitmap[row] >> (total_width - offset - 8)) & 0xFF
                self.set_row(row, shifted)
            time.sleep(delay)

    def close(self):
        self.clear()
        self.spi.close()


# ============================================================
# 8x8 폰트 (부분)
# ============================================================

FONT_8x8 = {
    ' ': [0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00],
    'A': [0x7E, 0x81, 0x81, 0x81, 0x7E, 0x00, 0x00, 0x00],
    'B': [0x7F, 0x49, 0x49, 0x49, 0x36, 0x00, 0x00, 0x00],
    'C': [0x3C, 0x42, 0x81, 0x81, 0x42, 0x00, 0x00, 0x00],
    'D': [0x7F, 0x81, 0x81, 0x81, 0x7E, 0x00, 0x00, 0x00],
    'E': [0x7F, 0x49, 0x49, 0x41, 0x41, 0x00, 0x00, 0x00],
    'F': [0x7F, 0x48, 0x48, 0x40, 0x40, 0x00, 0x00, 0x00],
    'G': [0x3C, 0x42, 0x81, 0x89, 0x4A, 0x00, 0x00, 0x00],
    'H': [0x7F, 0x08, 0x08, 0x08, 0x7F, 0x00, 0x00, 0x00],
    'I': [0x00, 0x81, 0x7F, 0x81, 0x00, 0x00, 0x00, 0x00],
    'J': [0x06, 0x01, 0x01, 0x01, 0x7E, 0x00, 0x00, 0x00],
    'K': [0x7F, 0x08, 0x14, 0x22, 0x41, 0x00, 0x00, 0x00],
    'L': [0x7F, 0x01, 0x01, 0x01, 0x01, 0x00, 0x00, 0x00],
    'M': [0x7F, 0x20, 0x10, 0x20, 0x7F, 0x00, 0x00, 0x00],
    'N': [0x7F, 0x10, 0x08, 0x04, 0x7F, 0x00, 0x00, 0x00],
    'O': [0x3C, 0x42, 0x81, 0x42, 0x3C, 0x00, 0x00, 0x00],
    'P': [0x7F, 0x48, 0x48, 0x48, 0x30, 0x00, 0x00, 0x00],
    'Q': [0x3C, 0x42, 0x85, 0x42, 0x3D, 0x00, 0x00, 0x00],
    'R': [0x7F, 0x48, 0x4C, 0x4A, 0x31, 0x00, 0x00, 0x00],
    'S': [0x32, 0x49, 0x49, 0x49, 0x26, 0x00, 0x00, 0x00],
    'T': [0x40, 0x40, 0x7F, 0x40, 0x40, 0x00, 0x00, 0x00],
    'U': [0x7E, 0x01, 0x01, 0x01, 0x7E, 0x00, 0x00, 0x00],
    'V': [0x7C, 0x02, 0x01, 0x02, 0x7C, 0x00, 0x00, 0x00],
    'W': [0x7F, 0x02, 0x0C, 0x02, 0x7F, 0x00, 0x00, 0x00],
    'X': [0x63, 0x14, 0x08, 0x14, 0x63, 0x00, 0x00, 0x00],
    'Y': [0x60, 0x10, 0x0F, 0x10, 0x60, 0x00, 0x00, 0x00],
    'Z': [0x43, 0x45, 0x49, 0x51, 0x61, 0x00, 0x00, 0x00],
    '0': [0x3E, 0x45, 0x49, 0x51, 0x3E, 0x00, 0x00, 0x00],
    '1': [0x00, 0x21, 0x7F, 0x01, 0x00, 0x00, 0x00, 0x00],
    '2': [0x23, 0x45, 0x49, 0x49, 0x31, 0x00, 0x00, 0x00],
    '3': [0x42, 0x81, 0x89, 0x89, 0x76, 0x00, 0x00, 0x00],
    '4': [0x0E, 0x12, 0x22, 0x7F, 0x02, 0x00, 0x00, 0x00],
    '5': [0x71, 0x89, 0x89, 0x89, 0x46, 0x00, 0x00, 0x00],
    '6': [0x3E, 0x49, 0x49, 0x49, 0x26, 0x00, 0x00, 0x00],
    '7': [0x40, 0x40, 0x47, 0x48, 0x70, 0x00, 0x00, 0x00],
    '8': [0x36, 0x49, 0x49, 0x49, 0x36, 0x00, 0x00, 0x00],
    '9': [0x30, 0x49, 0x49, 0x4A, 0x3C, 0x00, 0x00, 0x00],
    '!': [0x00, 0x00, 0x5F, 0x00, 0x00, 0x00, 0x00, 0x00],
    '.': [0x00, 0x00, 0x03, 0x00, 0x00, 0x00, 0x00, 0x00],
    ':': [0x00, 0x00, 0x24, 0x00, 0x00, 0x00, 0x00, 0x00],
}


def main():
    print("MAX7219 8x8 LED Matrix 테스트")
    print("=" * 35)

    try:
        matrix = MAX7219(bus=0, device=1)
    except FileNotFoundError:
        print("SPI 장치 /dev/spidev0.1을 찾을 수 없습니다.")
        return

    try:
        # 1. 전체 ON/OFF 테스트
        print("1. 전체 LED 점등")
        matrix.set_pattern([0xFF] * 8)
        time.sleep(1)

        print("2. 전체 LED 소등")
        matrix.clear()
        time.sleep(0.5)

        # 2. 체커보드 패턴
        print("3. 체커보드")
        checker = [0xAA, 0x55, 0xAA, 0x55, 0xAA, 0x55, 0xAA, 0x55]
        matrix.set_pattern(checker)
        time.sleep(1)

        # 3. 밝기 변화
        print("4. 밝기 변화")
        for i in range(16):
            matrix.set_intensity(i)
            matrix.set_pattern([0xFF] * 8)
            time.sleep(0.15)
        matrix.clear()

        # 4. 개별 문자 표시
        print("5. 문자 표시: 'JETSON'")
        for ch in "JETSON":
            matrix.set_pattern(FONT_8x8.get(ch, FONT_8x8[' ']))
            time.sleep(0.8)
        matrix.clear()

        # 5. 스크롤 텍스트
        print("6. 스크롤 텍스트: 'HELLO JETSON NANO'")
        matrix.scroll_text("HELLO JETSON NANO", FONT_8x8, delay=0.08)

        print("\n테스트 완료")

    except KeyboardInterrupt:
        print("\n종료")
    finally:
        matrix.close()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 max7219_matrix.py
```

---

## 4. 최종 실습 — 조도 센서 연동 LED 밝기 제어 + 매트릭스 스크롤

### 회로 구성

```
[CDS 조도 센서 분압 회로]     [MCP3008]             [Jetson Nano]
  3.3V ── CDS ───┬── 10kΩ ── GND     CH0 ── 분압점
                 │
                 ├──── MCP3008 CH0    VDD ── 3.3V
                                       VREF ── 3.3V
                                       AGND ── GND
                                       DGND ── GND
                                       CLK  ── Pin 23 (SPI_0_SCLK)
                                       DOUT ── Pin 21 (SPI_0_MISO)
                                       DIN  ── Pin 19 (SPI_0_MOSI)
                                       CS   ── Pin 26 (SPI_0_CS0)

[MAX7219 LED Matrix]              [Jetson Nano]
  VCC ── 5V (Pin 2)
  GND ── GND
  DIN ── Pin 19 (SPI_0_MOSI)
  CS  ── Pin 24 (SPI_0_CS1)
  CLK ── Pin 23 (SPI_0_SCLK)
```

**주의**: MCP3008은 SPI_0.0 (CS=Pin 26), MAX7219는 SPI_0.1 (CS=Pin 24)  
→ 같은 SPI 버스(SPI_0)에서 다른 CS 핀으로 분리

### 코드: `light_matrix_project.py`

```python
#!/usr/bin/env python3
"""
CDS 조도 센서 + MAX7219 LED Matrix 연동
- 조도 값에 따라 LED 매트릭스 밝기 자동 조절
- 조도 값을 매트릭스에 막대그래프로 표시
- 버튼 누르면 모드 전환 (밝기 자동 / 스크롤 텍스트)
"""

import spidev
import Jetson.GPIO as GPIO
import time
import threading

# ============================================================
# 핀 설정
# ============================================================
BUTTON_PIN = 29  # GPIO 149 (모드 전환)

# ============================================================
# MCP3008 ADC 드라이버
# ============================================================

class MCP3008:
    def __init__(self, bus=0, device=0):
        self.spi = spidev.SpiDev()
        self.spi.open(bus, device)
        self.spi.max_speed_hz = 1000000
        self.spi.mode = 0b00

    def read(self, channel=0):
        if channel < 0 or channel > 7:
            return 0
        cmd = 0x01 << 2 | 0x01 << 1 | (channel >> 2)
        data = [cmd, (channel & 0x03) << 6, 0x00]
        result = self.spi.xfer2(data)
        value = ((result[1] & 0x0F) << 8) | result[2]
        return value

    def close(self):
        self.spi.close()


# ============================================================
# MAX7219 LED Matrix 드라이버
# ============================================================

class MAX7219:
    REG_SHUTDOWN = 0x0C
    REG_DECODE_MODE = 0x09
    REG_SCAN_LIMIT = 0x0B
    REG_INTENSITY = 0x0A
    REG_DISPLAY_TEST = 0x0F
    REG_NOOP = 0x00

    def __init__(self, bus=0, device=1):
        self.spi = spidev.SpiDev()
        self.spi.open(bus, device)
        self.spi.max_speed_hz = 10000000
        self.spi.mode = 0b00
        self._init()

    def _write(self, addr, data):
        self.spi.xfer2([addr, data])

    def _init(self):
        self._write(self.REG_SHUTDOWN, 0x01)
        self._write(self.REG_DECODE_MODE, 0x00)
        self._write(self.REG_SCAN_LIMIT, 0x07)
        self._write(self.REG_INTENSITY, 0x08)
        self._write(self.REG_DISPLAY_TEST, 0x00)
        self.clear()

    def set_intensity(self, level):
        self._write(self.REG_INTENSITY, max(0, min(15, level & 0x0F)))

    def clear(self):
        for row in range(8):
            self._write(0x01 + row, 0x00)

    def set_pattern(self, pattern):
        for row in range(8):
            self._write(0x01 + row, pattern[row] & 0xFF)

    def set_bar_graph(self, value_0_1023):
        """
        조도 값을 8x8 매트릭스 막대 그래프로 표시
        value_0_1023: MCP3008 원시값 (0~1023)
        """
        # 8개 열 중 켜질 열 개수 계산
        num_columns = int((value_0_1023 / 1023.0) * 8)
        pattern = [0x00] * 8

        for col in range(num_columns):
            mask = (0xFF << col) & 0xFF
            self._write(0x01 + col, 0xFF)

        # 나머지 열 OFF
        for col in range(num_columns, 8):
            self._write(0x01 + col, 0x00)

    def scroll_text(self, text, font, delay=0.1):
        """텍스트 스크롤"""
        width = len(text) * 8
        bitmap = [0x00] * 8

        for ci, ch in enumerate(text):
            cb = font.get(ch, font[' '])
            for r in range(8):
                bitmap[r] = (bitmap[r] << 8) | cb[r]

        for offset in range(width + 8):
            for r in range(8):
                shift = width - offset
                if shift >= 0:
                    val = (bitmap[r] >> shift) & 0xFF
                else:
                    val = 0
                self._write(0x01 + r, val)
            time.sleep(delay)

    def close(self):
        self.clear()
        self._write(self.REG_SHUTDOWN, 0x00)
        self.spi.close()


# ============================================================
# 8x8 폰트 (실습 4-3의 FONT_8x8과 동일, 간략 버전)
# ============================================================

FONT_8x8 = {
    ' ': [0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00],
    'A': [0x7E, 0x81, 0x81, 0x81, 0x7E, 0x00, 0x00, 0x00],
    'B': [0x7F, 0x49, 0x49, 0x49, 0x36, 0x00, 0x00, 0x00],
    'C': [0x3C, 0x42, 0x81, 0x81, 0x42, 0x00, 0x00, 0x00],
    'D': [0x7F, 0x81, 0x81, 0x81, 0x7E, 0x00, 0x00, 0x00],
    'E': [0x7F, 0x49, 0x49, 0x41, 0x41, 0x00, 0x00, 0x00],
    'F': [0x7F, 0x48, 0x48, 0x40, 0x40, 0x00, 0x00, 0x00],
    'G': [0x3C, 0x42, 0x81, 0x89, 0x4A, 0x00, 0x00, 0x00],
    'H': [0x7F, 0x08, 0x08, 0x08, 0x7F, 0x00, 0x00, 0x00],
    'I': [0x00, 0x81, 0x7F, 0x81, 0x00, 0x00, 0x00, 0x00],
    'J': [0x06, 0x01, 0x01, 0x01, 0x7E, 0x00, 0x00, 0x00],
    'K': [0x7F, 0x08, 0x14, 0x22, 0x41, 0x00, 0x00, 0x00],
    'L': [0x7F, 0x01, 0x01, 0x01, 0x01, 0x00, 0x00, 0x00],
    'M': [0x7F, 0x20, 0x10, 0x20, 0x7F, 0x00, 0x00, 0x00],
    'N': [0x7F, 0x10, 0x08, 0x04, 0x7F, 0x00, 0x00, 0x00],
    'O': [0x3C, 0x42, 0x81, 0x42, 0x3C, 0x00, 0x00, 0x00],
    'P': [0x7F, 0x48, 0x48, 0x48, 0x30, 0x00, 0x00, 0x00],
    'Q': [0x3C, 0x42, 0x85, 0x42, 0x3D, 0x00, 0x00, 0x00],
    'R': [0x7F, 0x48, 0x4C, 0x4A, 0x31, 0x00, 0x00, 0x00],
    'S': [0x32, 0x49, 0x49, 0x49, 0x26, 0x00, 0x00, 0x00],
    'T': [0x40, 0x40, 0x7F, 0x40, 0x40, 0x00, 0x00, 0x00],
    'U': [0x7E, 0x01, 0x01, 0x01, 0x7E, 0x00, 0x00, 0x00],
    'V': [0x7C, 0x02, 0x01, 0x02, 0x7C, 0x00, 0x00, 0x00],
    'W': [0x7F, 0x02, 0x0C, 0x02, 0x7F, 0x00, 0x00, 0x00],
    'X': [0x63, 0x14, 0x08, 0x14, 0x63, 0x00, 0x00, 0x00],
    'Y': [0x60, 0x10, 0x0F, 0x10, 0x60, 0x00, 0x00, 0x00],
    'Z': [0x43, 0x45, 0x49, 0x51, 0x61, 0x00, 0x00, 0x00],
    '0': [0x3E, 0x45, 0x49, 0x51, 0x3E, 0x00, 0x00, 0x00],
    '1': [0x00, 0x21, 0x7F, 0x01, 0x00, 0x00, 0x00, 0x00],
    '2': [0x23, 0x45, 0x49, 0x49, 0x31, 0x00, 0x00, 0x00],
    '3': [0x42, 0x81, 0x89, 0x89, 0x76, 0x00, 0x00, 0x00],
    '4': [0x0E, 0x12, 0x22, 0x7F, 0x02, 0x00, 0x00, 0x00],
    '5': [0x71, 0x89, 0x89, 0x89, 0x46, 0x00, 0x00, 0x00],
    '6': [0x3E, 0x49, 0x49, 0x49, 0x26, 0x00, 0x00, 0x00],
    '7': [0x40, 0x40, 0x47, 0x48, 0x70, 0x00, 0x00, 0x00],
    '8': [0x36, 0x49, 0x49, 0x49, 0x36, 0x00, 0x00, 0x00],
    '9': [0x30, 0x49, 0x49, 0x4A, 0x3C, 0x00, 0x00, 0x00],
    '!': [0x00, 0x00, 0x5F, 0x00, 0x00, 0x00, 0x00, 0x00],
    '.': [0x00, 0x00, 0x03, 0x00, 0x00, 0x00, 0x00, 0x00],
    '?': [0x20, 0x40, 0x45, 0x48, 0x30, 0x00, 0x00, 0x00],
    '*': [0x24, 0x18, 0x7E, 0x18, 0x24, 0x00, 0x00, 0x00'],
}


# ============================================================
# 메인 프로그램
# ============================================================

MODE_BAR = 0    # 막대 그래프 모드
MODE_SCROLL = 1  # 스크롤 텍스트 모드
current_mode = MODE_BAR


def button_callback(channel):
    """버튼 인터럽트 — 모드 전환"""
    global current_mode
    current_mode = 1 - current_mode
    mode_name = "막대 그래프" if current_mode == MODE_BAR else "스크롤 텍스트"
    print(f"모드 전환: {mode_name}")


def main():
    global current_mode

    print("조도 센서 + LED 매트릭스 프로젝트")
    print("=" * 40)
    print("모드 0: 막대 그래프 (조도 값 표시)")
    print("모드 1: 스크롤 텍스트 ('HELLO JETSON')")
    print("버튼(핀 29)을 누르면 모드 전환")
    print("Ctrl+C로 종료")
    print("-" * 40)

    # SPI 장치 초기화
    try:
        adc = MCP3008(bus=0, device=0)
        matrix = MAX7219(bus=0, device=1)
    except FileNotFoundError as e:
        print(f"SPI 장치 오류: {e}")
        return

    # GPIO 버튼 설정
    GPIO.setmode(GPIO.BOARD)
    GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
    GPIO.add_event_detect(BUTTON_PIN, GPIO.FALLING,
                          callback=button_callback, bouncetime=500)

    last_value = -1
    try:
        while True:
            # 조도 값 읽기
            light_raw = adc.read(0)

            # 조도에 따라 매트릭스 밝기 조절 (어두우면 밝게)
            # 어두움(값 낮음) → 밝기 ↑, 밝음(값 높음) → 밝기 ↓
            brightness = max(1, 15 - int(light_raw / 68))
            matrix.set_intensity(brightness)

            if current_mode == MODE_BAR:
                # 막대 그래프 모드
                matrix.set_bar_graph(light_raw)

                if abs(light_raw - last_value) > 10:
                    pct = light_raw / 1023.0 * 100.0
                    bar = '█' * int(pct / 5) + '░' * (20 - int(pct / 5))
                    print(f"조도: {light_raw:4d} ({pct:5.1f}%)  밝기:{brightness:2d}  {bar}")
                    last_value = light_raw

            else:
                # 스크롤 텍스트 모드 (5초 간격)
                matrix.scroll_text("HELLO JETSON", FONT_8x8, delay=0.08)
                time.sleep(0.5)

            time.sleep(0.05)

    except KeyboardInterrupt:
        print("\n종료 중...")
    finally:
        GPIO.cleanup()
        matrix.close()
        adc.close()
        print("정리 완료")


if __name__ == "__main__":
    main()
```

**실행**:
```bash
sudo python3 light_matrix_project.py
```

---

## 5. 확인 및 테스트

### 검증 절차

```
1. SPI 장치 확인
   → ls -l /dev/spidev* → /dev/spidev0.0, /dev/spidev0.1 확인

2. MCP3008 조도 센서
   → mcp3008_light.py → CDS 가리면 값 ↓, 밝히면 값 ↑ 확인

3. MAX7219 LED Matrix
   → max7219_matrix.py → 전체 ON → 체커보드 → 문자 → 스크롤 순차 실행

4. 최종 실습
   → sudo python3 light_matrix_project.py
   → CDS 조도 변화에 따라 매트릭스 막대 그래프 변화 확인
   → 버튼 누르면 "HELLO JETSON" 스크롤 표시
   → 다시 버튼 누르면 막대 그래프로 복귀
```

### 문제 해결

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| SPI 장치 없음 | SPI 비활성화 | `sudo /opt/nvidia/jetson-io/jetson-io.py`로 활성화 |
| MAX7219 안 켜짐 | 5V 전원 필요 | 3.3V 대신 5V 핀에 연결 |
| MCP3008 값 고정 | CS 풀업 문제 | CS 핀에 10kΩ 풀업 저항 추가 |
| MCP3008 값 튐 | 기준 전압 노이즈 | VREF에 0.1µF 커패시터 추가 |
| MAX7219 패턴 깨짐 | SPI 속도 너무 빠름 | `max_speed_hz`를 1MHz로 낮춤 |
| 두 장치 충돌 | CS 핀 분리 필요 | MCP3008=CS0(Pin26), MAX7219=CS1(Pin24) |
| SPI Permission denied | 권한 문제 | `sudo`로 실행 |
| MAX7219 흐릿함 | 밝기 낮음 | `set_intensity(15)`로 최대 밝기 |

---

> **다음 모듈**: Module 05 — UART (GPS, 블루투스, 디버그 콘솔)

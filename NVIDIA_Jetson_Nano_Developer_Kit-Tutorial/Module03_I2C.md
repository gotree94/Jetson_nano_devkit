# Module 03 — I2C (Inter-Integrated Circuit)

> **소요 시간**: 3~4시간  
> **난이도**: ★★☆☆☆  
> **준비물**: AHT20(또는 SHT30) 온습도 센서, MPU6050 IMU 센서, SSD1306 OLED 128x64(I2C), TCA9548A I2C 멀티플렉서(선택)

---

## 1. 개요

I2C는 두 개의 선(SDA, SCL)만으로 여러 장치를 통신할 수 있는 동기식 직렬 통신 프로토콜입니다.  
Jetson Nano는 두 개의 I2C 버스를 제공하며, 다양한 센서와 디스플레이를 연결할 수 있습니다.

### 학습 목표
- I2C 프로토콜의 프레임 구조(Start/Stop/Ack)를 이해한다
- `i2cdetect`로 I2C 버스를 스캔하고 장치 주소를 확인할 수 있다
- `smbus2` 라이브러리로 I2C 장치의 레지스터를 읽고 쓸 수 있다
- 온습도 센서(AHT20)의 데이터시트를 따라 센서 값을 읽을 수 있다
- IMU 센서(MPU6050)로 가속도/자이로 데이터를 수집할 수 있다
- OLED 디스플레이(SSD1306)에 텍스트와 도형을 출력할 수 있다

### 사전 지식
- Module 00의 I2C 도구 설치 완료 (`i2c-tools`, `libi2c-dev`)
- 2진수/16진수 표현 이해

---

## 2. 이론

### 2.1 I2C 프로토콜

I2C는 **Master-Slave** 구조로 동작합니다. Jetson Nano가 Master 역할을 합니다.

**버스 구성**:
```
  3.3V (풀업 저항 2.2kΩ)
    │
    ├─── R ────┐      ┌───── R ──────┐
    │           │      │              │
  SDA ──────────┴──────┴───┬──────────┴─── ...
    │                      │
  SCL ─────────────────────┴───┬─────────── ...
    │                          │
  GND ─────────────────────────┴─────────── ...
              Slave 1         Slave 2
            (AHT20)         (MPU6050)
```

> Jetson Nano의 I2C 핀(SDA/SCL)에는 이미 **2.2kΩ 풀업 저항**이 내장되어 있습니다.  
> 별도 풀업 저항을 추가할 필요가 없습니다.

**프레임 구조**:
```
Start | 7-bit 주소 | R/W | Ack | 데이터(8bit) | Ack | ... | Stop

Start: SDA가 HIGH→LOW로 전이, SCL은 HIGH (유일한 조건)
Stop : SDA가 LOW→HIGH로 전이, SCL은 HIGH
Ack  : 수신자가 SDA를 LOW로 유지 (1클럭)
Nack : 수신자가 SDA를 HIGH로 유지 (1클럭)
```

**7비트 주소**:
- 각 I2C 장치는 고유의 7비트 주소를 가집니다
- 주소 범위: 0x08 ~ 0x77 (0x00~0x07, 0x78~0x7F는 예약)
- 일반적으로 장치 데이터시트에 명시됨 (예: MPU6050 = 0x68)

**R/W 비트**:
- 0 = Master가 Slave에 쓰기 (Write)
- 1 = Master가 Slave에서 읽기 (Read)

### 2.2 Jetson Nano I2C 버스

Jetson Nano에는 **2개의 I2C 버스**가 있습니다:

| 버스 | 장치 파일 | 40-pin 헤더 핀 | 용도 |
|------|----------|----------------|------|
| **I2C_0** (I2C2) | `/dev/i2c-0` | Pin 27(SDA), 28(SCL) | 카메라 CSI, 일반 센서 |
| **I2C_1** (I2C1) | `/dev/i2c-1` | Pin 3(SDA), 5(SCL) | **일반 센서 권장** |

> **실습에서는 I2C_1 (Pin 3, 5)을 사용합니다.**  
> `i2cdetect -y -r 1` 명령어로 스캔합니다.

### 2.3 smbus2 라이브러리

Python에서 I2C 통신을 위한 표준 라이브러리입니다.

```python
from smbus2 import SMBus

# I2C 버스 열기
bus = SMBus(1)  # /dev/i2c-1

# 단일 바이트 읽기
value = bus.read_byte(address)

# 레지스터에서 바이트 읽기
value = bus.read_byte_data(address, register)

# 레지스터에서 여러 바이트 읽기
data = bus.read_i2c_block_data(address, register, length)

# 단일 바이트 쓰기
bus.write_byte(address, value)

# 레지스터에 바이트 쓰기
bus.write_byte_data(address, register, value)

# 버스 닫기
bus.close()
```

### 2.4 AHT20 온습도 센서 데이터시트 읽기

AHT20은 I2C 온습도 센서입니다. 데이터시트 기준 통신 절차:

```
1. 초기화: 0xBE (초기화 명령)
2. 측정 트리거: 0xAC + 0x33 + 0x00
3. 80ms 대기 (측정 완료 대기)
4. 데이터 읽기: 6바이트
   Byte 0: 상태 비트
   Byte 1-2: 습도 데이터 (20비트 중 상위)
   Byte 3-4: 온도 데이터 (20비트 중 상위)
   Byte 5: CRC (선택)
   
5. 값 변환:
   습도 = (raw_humidity / 2^20) × 100%
   온도 = (raw_temperature / 2^20) × 200 - 50
```

| I2C 주소 | 쓰기 | 읽기 |
|----------|------|------|
| 0x38 | 0x38 | 0x39 |

### 2.5 MPU6050 IMU

MPU6050은 3축 가속도계 + 3축 자이로스코프를 포함하는 IMU 센서입니다.

| 특성 | 값 |
|------|-----|
| I2C 주소 | 0x68 (AD0=LOW) / 0x69 (AD0=HIGH) |
| 가속도 범위 | ±2g / ±4g / ±8g / ±16g |
| 자이로 범위 | ±250 / ±500 / ±1000 / ±2000 °/s |
| 내부 온도 센서 | 있음 |
| 추가 기능 | DMP (Digital Motion Processor) |

**주요 레지스터**:
| 주소 | 이름 | 설명 |
|------|------|------|
| 0x6B | PWR_MGMT_1 | 전원 관리 (초기화 시 쓰기 필요) |
| 0x3B | ACCEL_XOUT_H | 가속도 X축 상위 바이트 |
| 0x3C | ACCEL_XOUT_L | 가속도 X축 하위 바이트 |
| 0x3D | ACCEL_YOUT_H | 가속도 Y축 상위 바이트 |
| 0x3E | ACCEL_YOUT_L | 가속도 Y축 하위 바이트 |
| 0x3F | ACCEL_ZOUT_H | 가속도 Z축 상위 바이트 |
| 0x40 | ACCEL_ZOUT_L | 가속도 Z축 하위 바이트 |
| 0x41 | TEMP_OUT_H | 온도 상위 바이트 |
| 0x43 | GYRO_XOUT_H | 자이로 X축 상위 바이트 |

**오일러 각 계산 (Pitch/Roll)**:
```
Pitch = atan2(-Accel_X, sqrt(Accel_Y^2 + Accel_Z^2)) × 180/π
Roll  = atan2(Accel_Y, Accel_Z) × 180/π
```

### 2.6 SSD1306 OLED

SSD1306은 128×64 픽셀 모노크롬 OLED 드라이버입니다.

| 특성 | 값 |
|------|-----|
| I2C 주소 | 0x3C (일반) / 0x3D (일부) |
| 해상도 | 128×64 또는 128×32 |
| 구동 방식 | 페이지 주소 모드 / 수평 주소 모드 |
| 내장 | 128×64비트 GDDRAM |

**luma.oled 라이브러리**를 사용하여 쉽게 제어할 수 있습니다.

---

## 3. 실습 코드

### 실습 3-1: I2C 버스 스캔

```bash
# I2C_1 버스 스캔
i2cdetect -y -r 1

# 출력 예시:
#      0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
# 00:          -- -- -- -- -- -- -- -- -- -- -- -- --
# 10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
# 20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
# 30: -- -- -- -- -- -- -- -- -- -- -- 3c -- -- -- --
# 40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
# 50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
# 60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 68
# 70: -- -- -- -- -- -- -- -- 38
# (3c=SSD1306, 68=MPU6050, 38=AHT20)

# I2C_0 버스 스캔
i2cdetect -y -r 0
```

### 실습 3-2: AHT20 온습도 센서

**회로 구성**:
```
AHT20:
  VIN ──→ 3.3V (Jetson Pin 1)
  GND ──→ GND (Jetson Pin 6)
  SDA ──→ Pin 3 (I2C_1_SDA)
  SCL ──→ Pin 5 (I2C_1_SCL)
```

**코드**: `aht20_sensor.py`
```python
#!/usr/bin/env python3
"""
AHT20 온습도 센서 I2C 통신
데이터시트 기반 레지스터 읽기/쓰기 구현
"""

from smbus2 import SMBus
import time

# AHT20 I2C 주소
AHT20_ADDR = 0x38

# 명령어
CMD_INIT = 0xBE       # 초기화
CMD_TRIGGER = 0xAC    # 측정 트리거
CMD_SOFTRESET = 0xBA  # 소프트 리셋

# 상태 비트 마스크
STATUS_BUSY = 0x80      # Bit 7: 측정 중 = 1
STATUS_CALIBRATED = 0x08  # Bit 3: 보정 완료 = 1


class AHT20:
    """AHT20 온습도 센서 드라이버"""

    def __init__(self, bus=1):
        self.bus = SMBus(bus)
        self._init_sensor()

    def _init_sensor(self):
        """센서 초기화 및 보정"""
        # 소프트 리셋
        try:
            self.bus.write_byte(AHT20_ADDR, CMD_SOFTRESET)
        except OSError:
            pass  # 리셋 후 장치 응답 없음 (정상)
        time.sleep(0.02)

        # 초기화 명령
        self.bus.write_byte(AHT20_ADDR, CMD_INIT)
        time.sleep(0.01)

        # 보정 완료 대기
        for _ in range(50):
            status = self.bus.read_byte(AHT20_ADDR)
            if status & STATUS_CALIBRATED:
                print("AHT20 보정 완료")
                break
            time.sleep(0.01)
        else:
            print("경고: AHT20 보정 완료 응답 없음")

    def trigger_measurement(self):
        """측정 트리거"""
        self.bus.write_i2c_block_data(AHT20_ADDR, CMD_TRIGGER, [0x33, 0x00])

    def read_data(self):
        """6바이트 데이터 읽기"""
        data = self.bus.read_i2c_block_data(AHT20_ADDR, 0x00, 6)
        return data

    def is_busy(self, data):
        """측정 중인지 확인"""
        return bool(data[0] & STATUS_BUSY)

    def read_measurements(self):
        """온도/습도 읽기 (재시도 포함)"""
        self.trigger_measurement()
        time.sleep(0.08)  # 최소 75ms 대기 (데이터시트)

        for attempt in range(20):
            data = self.read_data()
            if not self.is_busy(data):
                break
            time.sleep(0.01)
        else:
            print("경고: 측정 완료 응답 없음")

        # 습도 (20비트)
        raw_humidity = ((data[1] << 16) | (data[2] << 8) | data[3]) >> 4
        humidity = (raw_humidity / 0x100000) * 100.0

        # 온도 (20비트)
        raw_temp = ((data[3] & 0x0F) << 16) | (data[4] << 8) | data[5]
        temperature = (raw_temp / 0x100000) * 200.0 - 50.0

        return temperature, humidity

    def close(self):
        self.bus.close()


def main():
    print("AHT20 온습도 센서 테스트")
    print("=" * 35)

    # I2C 주소 확인
    from smbus2 import SMBus
    check_bus = SMBus(1)
    try:
        check_bus.read_byte(AHT20_ADDR)
        print(f"  장치 발견: 0x{AHT20_ADDR:02X}")
    except OSError:
        print(f"  오류: 0x{AHT20_ADDR:02X} 주소에서 장치를 찾을 수 없음")
        check_bus.close()
        return
    check_bus.close()

    sensor = AHT20(bus=1)

    try:
        while True:
            temp, hum = sensor.read_measurements()
            print(f"  온도: {temp:6.2f}°C  |  습도: {hum:6.2f}%")
            time.sleep(2.0)
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        sensor.close()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 aht20_sensor.py
```

### 실습 3-3: MPU6050 IMU 센서

**회로 구성**:
```
MPU6050:
  VCC ──→ 3.3V (Jetson Pin 1)
  GND ──→ GND (Jetson Pin 6)
  SDA ──→ Pin 3 (I2C_1_SDA)
  SCL ──→ Pin 5 (I2C_1_SCL)
  ADO ──→ GND (주소 0x68)
```

**코드**: `mpu6050_reader.py`
```python
#!/usr/bin/env python3
"""
MPU6050 IMU 센서 I2C 통신
가속도/자이로/온도 읽기 + 오일러 각(Pitch/Roll) 계산
"""

from smbus2 import SMBus
import time
import math

MPU6050_ADDR = 0x68

# 레지스터 주소
REG_PWR_MGMT_1 = 0x6B
REG_ACCEL_XOUT_H = 0x3B
REG_GYRO_XOUT_H = 0x43
REG_CONFIG = 0x1A
REG_GYRO_CONFIG = 0x1B
REG_ACCEL_CONFIG = 0x1C


class MPU6050:
    """MPU6050 IMU 센서 드라이버"""

    def __init__(self, bus=1, addr=MPU6050_ADDR):
        self.bus = SMBus(bus)
        self.addr = addr

        # 센서 초기화
        self._write_byte(REG_PWR_MGMT_1, 0x00)  # 슬립 모드 해제
        time.sleep(0.1)

        # 기본 설정
        self._write_byte(REG_CONFIG, 0x00)        # 내부 필터
        self._write_byte(REG_GYRO_CONFIG, 0x00)   # 자이로 ±250°/s
        self._write_byte(REG_ACCEL_CONFIG, 0x00)  # 가속도 ±2g

        # 스케일 팩터
        self.accel_scale = 2.0 / 32768.0  # ±2g / 16-bit
        self.gyro_scale = 250.0 / 32768.0  # ±250°/s / 16-bit

        print("MPU6050 초기화 완료")

    def _write_byte(self, reg, value):
        self.bus.write_byte_data(self.addr, reg, value)

    def _read_word(self, reg):
        """16비트 레지스터 읽기 (빅엔디언)"""
        high = self.bus.read_byte_data(self.addr, reg)
        low = self.bus.read_byte_data(self.addr, reg + 1)
        value = (high << 8) | low
        # 2의 보수 (음수 변환)
        if value >= 0x8000:
            value -= 0x10000
        return value

    def read_accel(self):
        """3축 가속도 읽기 (g)"""
        x = self._read_word(REG_ACCEL_XOUT_H) * self.accel_scale
        y = self._read_word(REG_ACCEL_XOUT_H + 2) * self.accel_scale
        z = self._read_word(REG_ACCEL_XOUT_H + 4) * self.accel_scale
        return (x, y, z)

    def read_gyro(self):
        """3축 자이로 읽기 (°/s)"""
        x = self._read_word(REG_GYRO_XOUT_H) * self.gyro_scale
        y = self._read_word(REG_GYRO_XOUT_H + 2) * self.gyro_scale
        z = self._read_word(REG_GYRO_XOUT_H + 4) * self.gyro_scale
        return (x, y, z)

    def read_temp(self):
        """내부 온도 읽기 (°C)"""
        raw = self._read_word(0x41)
        return raw / 340.0 + 36.53

    def calc_orientation(self, ax, ay, az):
        """가속도로 Pitch/Roll 계산 (라디안 -> 도)"""
        pitch = math.atan2(-ax, math.sqrt(ay * ay + az * az)) * 180.0 / math.pi
        roll = math.atan2(ay, az) * 180.0 / math.pi
        return pitch, roll

    def close(self):
        self.bus.close()


def main():
    print("MPU6050 IMU 센서 테스트")
    print("=" * 45)

    try:
        sensor = MPU6050(bus=1)
    except OSError:
        print(f"MPU6050를 0x{MPU6050_ADDR:02X}에서 찾을 수 없습니다.")
        print("배선과 AD0 핀 연결을 확인하세요.")
        return

    print("\n실시간 센서 데이터 (Ctrl+C로 종료)")
    print("-" * 65)
    print(f"{'Accel X':>8} {'Y':>8} {'Z':>8}  |"
          f"{'Gyro X':>8} {'Y':>8} {'Z':>8}  |"
          f"{'Pitch':>7} {'Roll':>7}  {'Temp':>6}")
    print("-" * 65)

    try:
        while True:
            ax, ay, az = sensor.read_accel()
            gx, gy, gz = sensor.read_gyro()
            temp = sensor.read_temp()
            pitch, roll = sensor.calc_orientation(ax, ay, az)

            print(f"{ax:8.3f} {ay:8.3f} {az:8.3f}  |"
                  f"{gx:8.2f} {gy:8.2f} {gz:8.2f}  |"
                  f"{pitch:7.2f} {roll:7.2f}  {temp:6.2f}°C")

            time.sleep(0.1)  # 10Hz
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        sensor.close()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 mpu6050_reader.py
```

센서를 움직이면 가속도/자이로 값과 Pitch/Roll 각도가 실시간으로 변화하는 것을 확인합니다.

### 실습 3-4: SSD1306 OLED 디스플레이

**회로 구성**:
```
SSD1306 OLED (I2C):
  VCC ──→ 3.3V (Jetson Pin 1)
  GND ──→ GND (Jetson Pin 6)
  SDA ──→ Pin 3 (I2C_1_SDA)
  SCL ──→ Pin 5 (I2C_1_SCL)
```

**설치**:
```bash
sudo apt install -y python3-pil
pip3 install luma.oled
```

**코드**: `oled_display.py`
```python
#!/usr/bin/env python3
"""
SSD1306 OLED 128x64 디스플레이 예제
텍스트, 도형, 이미지 출력
"""

from luma.core.interface.serial import i2c
from luma.core.render import canvas
from luma.oled.device import ssd1306
from PIL import ImageFont, ImageDraw
import time
import math

# I2C 인터페이스 생성
serial = i2c(port=1, address=0x3C)

# OLED 장치 초기화
device = ssd1306(serial)

print(f"OLED 초기화 완료: {device.width}x{device.height}")


def draw_text():
    """텍스트 출력 데모"""
    # 기본 폰트 (크기 조정을 위해 TrueType 폰트 사용 가능)
    font_large = ImageFont.truetype("/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf", 16)
    font_small = ImageFont.truetype("/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", 12)

    with canvas(device) as draw:
        draw.rectangle((0, 0, device.width - 1, device.height - 1),
                       outline="white", fill="black")
        draw.text((10, 5), "Hello Jetson!", font=font_large, fill="white")
        draw.text((10, 28), "NVIDIA Nano", font=font_small, fill="white")
        draw.text((10, 45), "I2C OLED 128x64", font=font_small, fill="white")

    time.sleep(3)


def draw_shapes():
    """도형 그리기 데모"""
    with canvas(device) as draw:
        # 직사각형
        draw.rectangle((5, 5, 50, 30), outline="white", fill=None)

        # 채워진 직사각형
        draw.rectangle((60, 5, 105, 30), outline="white", fill="white")

        # 원 (타원)
        draw.ellipse((5, 35, 50, 60), outline="white", fill=None)

        # 라인
        draw.line((60, 35, 105, 60), fill="white", width=2)
        draw.line((105, 35, 60, 60), fill="white", width=2)

        # 점
        for x in range(0, 128, 8):
            for y in range(0, 64, 8):
                draw.point((x, y), fill="white")

    time.sleep(3)


def draw_animation():
    """애니메이션 — 움직이는 사인파"""
    font = ImageFont.truetype("/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", 10)

    for phase in range(0, 360, 5):
        with canvas(device) as draw:
            draw.text((0, 0), f"Sine Wave {phase}°", font=font, fill="white")

            # 사인파 그리기
            points = []
            for x in range(0, 128):
                rad = math.radians(x * 360 / 128 + phase)
                y = 32 + int(20 * math.sin(rad))
                points.append((x, y))

            draw.line(points, fill="white", width=1)

            # 기준선
            draw.line((0, 32, 127, 32), fill="white", width=1)


def sensor_display(temp, hum):
    """센서 값 표시 (외부 호출용)"""
    font_title = ImageFont.truetype(
        "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf", 14)
    font_value = ImageFont.truetype(
        "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", 20)

    with canvas(device) as draw:
        # 테두리
        draw.rectangle((0, 0, device.width - 1, device.height - 1),
                       outline="white", fill="black")

        # 제목
        draw.text((10, 2), "Environment Monitor", font=font_title, fill="white")

        # 온도
        draw.text((10, 22), f"{temp:.1f} C", font=font_value, fill="white")

        # 습도
        draw.text((10, 44), f"{hum:.1f} %", font=font_value, fill="white")


def main():
    print("\nOLED 데모 시작")
    print("1. 텍스트 출력")
    draw_text()

    print("2. 도형 그리기")
    draw_shapes()

    print("3. 사인파 애니메이션 (3초)")
    draw_animation()

    print("4. 센서 값 표시 (가상 데이터)")
    sensor_display(25.5, 60.0)
    time.sleep(3)

    print("OLED 데모 완료")
    device.clear()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 oled_display.py
```

---

## 4. 최종 실습 — 환경 모니터 스테이션 + MPU6050 자세 추정

### 4-A: 환경 모니터 스테이션

AHT20 + SSD1306 OLED를 결합하여 실시간 온습도 모니터를 만듭니다.

**코드**: `env_monitor.py`
```python
#!/usr/bin/env python3
"""
환경 모니터 스테이션
AHT20 온습도 + SSD1306 OLED 디스플레이
1초 간격 갱신
"""

from smbus2 import SMBus
from luma.core.interface.serial import i2c
from luma.core.render import canvas
from luma.oled.device import ssd1306
from PIL import ImageFont
import time
import threading

# ============================================================
# AHT20 드라이버 (실습 3-2에서 재사용)
# ============================================================

class AHT20:
    def __init__(self, bus=1):
        self.bus = SMBus(bus)
        self.addr = 0x38
        self._init_sensor()

    def _init_sensor(self):
        self.bus.write_byte(self.addr, 0xBA)  # soft reset
        time.sleep(0.02)
        self.bus.write_byte(self.addr, 0xBE)  # init
        time.sleep(0.01)
        for _ in range(50):
            status = self.bus.read_byte(self.addr)
            if status & 0x08:
                break
            time.sleep(0.01)

    def read(self):
        self.bus.write_i2c_block_data(self.addr, 0xAC, [0x33, 0x00])
        time.sleep(0.08)
        for _ in range(20):
            data = self.bus.read_i2c_block_data(self.addr, 0x00, 6)
            if not (data[0] & 0x80):
                break
            time.sleep(0.01)

        raw_hum = ((data[1] << 16) | (data[2] << 8) | data[3]) >> 4
        hum = (raw_hum / 0x100000) * 100.0
        raw_temp = ((data[3] & 0x0F) << 16) | (data[4] << 8) | data[5]
        temp = (raw_temp / 0x100000) * 200.0 - 50.0
        return temp, hum

    def close(self):
        self.bus.close()


# ============================================================
# OLED 드라이버
# ============================================================

class OLEDDisplay:
    def __init__(self):
        serial = i2c(port=1, address=0x3C)
        self.device = ssd1306(serial)
        self.font_small = ImageFont.truetype(
            "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", 10)
        self.font_large = ImageFont.truetype(
            "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf", 14)
        self.font_value = ImageFont.truetype(
            "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", 22)

    def update(self, temp, hum, count):
        """OLED에 온습도 + 카운터 표시"""
        with canvas(self.device) as draw:
            # 상태 표시줄
            draw.text((0, 0), f"Env Monitor #{count}", font=self.font_small,
                      fill="white")

            # 구분선
            draw.line((0, 12, 127, 12), fill="white", width=1)

            # 온도
            draw.text((5, 16), "Temp", font=self.font_small, fill="white")
            draw.text((5, 28), f"{temp:5.1f}C", font=self.font_value,
                      fill="white")

            # 구분선
            draw.line((64, 16, 64, 63), fill="white", width=1)

            # 습도
            draw.text((70, 16), "Humid", font=self.font_small, fill="white")
            draw.text((70, 28), f"{hum:5.1f}%", font=self.font_value,
                      fill="white")

            # 하단 정보
            draw.text((0, 55), time.strftime("%H:%M:%S"), font=self.font_small,
                      fill="white")

    def clear(self):
        self.device.clear()

    def close(self):
        self.device.cleanup()


# ============================================================
# 메인
# ============================================================

def main():
    print("환경 모니터 스테이션 시작")
    print("=" * 35)

    # I2C 장치 확인
    check_bus = SMBus(1)
    devices_found = []
    for addr in [0x38, 0x3C]:
        try:
            check_bus.read_byte(addr)
            devices_found.append(hex(addr))
        except OSError:
            pass
    check_bus.close()
    print(f"발견된 I2C 장치: {', '.join(devices_found)}")

    if "0x38" not in devices_found:
        print("오류: AHT20을 찾을 수 없습니다 (주소 0x38)")
        return
    if "0x3c" not in devices_found:
        print("오류: SSD1306을 찾을 수 없습니다 (주소 0x3C)")
        return

    sensor = AHT20()
    display = OLEDDisplay()

    count = 0
    try:
        while True:
            temp, hum = sensor.read()
            count += 1
            display.update(temp, hum, count)
            print(f"[{count:4d}] 온도: {temp:6.2f}C  습도: {hum:6.2f}%")
            time.sleep(1.0)
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        display.clear()
        display.close()
        sensor.close()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 env_monitor.py
```

OLED에 온도와 습도가 1초 간격으로 표시됩니다.

### 4-B: MPU6050 칼만 필터 기초

MPU6050의 가속도 + 자이로 데이터를 융합하여 안정적인 자세(Pitch/Roll)를 추정합니다.

**코드**: `mpu6050_kalman.py`
```python
#!/usr/bin/env python3
"""
MPU6050 자세 추정 — 칼만 필터 기초
가속도(저주파 신뢰) + 자이로(고주파 신뢰) 융합

원리:
  - 가속도: 장기적 안정, 단기적 노이즈
  - 자이로: 단기적 정확, 장기적 드리프트

상보 필터 (Complementary Filter):
  angle = 0.98 × (angle + gyro×dt) + 0.02 × accel_angle

칼만 필터 (1차):
  1. Predict: 각도 예측 = 이전 각도 + 자이로 적분
  2. Update: 가속도 측정값으로 보정
"""

from smbus2 import SMBus
import time
import math
import sys

MPU6050_ADDR = 0x68
REG_PWR_MGMT_1 = 0x6B
REG_ACCEL_XOUT_H = 0x3B
REG_GYRO_XOUT_H = 0x43


class MPU6050:
    def __init__(self, bus=1):
        self.bus = SMBus(bus)
        self.addr = MPU6050_ADDR
        self.bus.write_byte_data(self.addr, REG_PWR_MGMT_1, 0x00)
        time.sleep(0.1)
        self.accel_scale = 2.0 / 32768.0
        self.gyro_scale = 250.0 / 32768.0

    def read_word(self, reg):
        high = self.bus.read_byte_data(self.addr, reg)
        low = self.bus.read_byte_data(self.addr, reg + 1)
        val = (high << 8) | low
        if val >= 0x8000:
            val -= 0x10000
        return val

    def read_all(self):
        ax = self.read_word(REG_ACCEL_XOUT_H) * self.accel_scale
        ay = self.read_word(REG_ACCEL_XOUT_H + 2) * self.accel_scale
        az = self.read_word(REG_ACCEL_XOUT_H + 4) * self.accel_scale
        gx = self.read_word(REG_GYRO_XOUT_H) * self.gyro_scale
        gy = self.read_word(REG_GYRO_XOUT_H + 2) * self.gyro_scale
        gz = self.read_word(REG_GYRO_XOUT_H + 4) * self.gyro_scale
        return (ax, ay, az, gx, gy, gz)

    def close(self):
        self.bus.close()


class ComplementaryFilter:
    """상보 필터 — 가속도와 자이로 융합"""

    def __init__(self, alpha=0.98):
        self.alpha = alpha
        self.pitch = 0.0
        self.roll = 0.0
        self.last_time = time.time()

    def update(self, ax, ay, az, gx, gy, gz):
        now = time.time()
        dt = now - self.last_time
        self.last_time = now

        if dt <= 0 or dt > 0.1:
            dt = 0.01  # 안전장치

        # 가속도로 각도 계산
        accel_pitch = math.atan2(-ax, math.sqrt(ay*ay + az*az)) * 180.0 / math.pi
        accel_roll = math.atan2(ay, az) * 180.0 / math.pi

        # 자이로 적분
        gyro_pitch = self.pitch + gx * dt
        gyro_roll = self.roll + gy * dt

        # 상보 필터 융합
        self.pitch = self.alpha * gyro_pitch + (1 - self.alpha) * accel_pitch
        self.roll = self.alpha * gyro_roll + (1 - self.alpha) * accel_roll

        return self.pitch, self.roll


class SimpleKalmanFilter:
    """
    1차원 칼만 필터 (Pitch/Roll 각각 적용)

    상태: 각도
    측정: 가속도 기반 각도
    제어: 자이로
    """

    def __init__(self, Q=0.001, R=0.03):
        """
        Q: 프로세스 노이즈 (자이로 드리프트 불확실성)
           ↑ 크면 자이로 신뢰 ↓
        R: 측정 노이즈 (가속도 노이즈)
           ↑ 크면 가속도 신뢰 ↓
        """
        self.Q = Q
        self.R = R
        self.P = 1.0  # 초기 추정 오차 공분산
        self.K = 0.0  # 칼만 이득
        self.angle = 0.0
        self.bias = 0.0
        self.rate = 0.0
        self.last_time = time.time()

    def update(self, accel_angle, gyro_rate):
        """
        칼만 필터 1단계 업데이트

        accel_angle: 가속도로 계산한 각도 (측정값)
        gyro_rate: 자이로 각속도 (제어 입력)
        """
        now = time.time()
        dt = now - self.last_time
        self.last_time = now

        if dt <= 0 or dt > 0.1:
            dt = 0.01

        # === 예측 (Predict) ===
        # 자이로로 각도 예측
        self.angle += (gyro_rate - self.bias) * dt
        # 오차 공분산 예측
        self.P += self.Q * dt

        # === 업데이트 (Update) ===
        # 칼만 이득 계산
        self.K = self.P / (self.P + self.R)

        # 가속도 측정값으로 보정
        self.angle += self.K * (accel_angle - self.angle)

        # 바이어스 업데이트
        # (실제 구현에서는 2차 칼만 필터 필요)

        # 오차 공분산 업데이트
        self.P = (1 - self.K) * self.P

        return self.angle


def main():
    print("MPU6050 자세 추정 — 필터 비교")
    print("=" * 55)
    print(f"{'Time':>8} {'AccelPitch':>10} {'AccelRoll':>10}  |"
          f"{'CompPitch':>10} {'CompRoll':>10}  |"
          f"{'KalmanPitch':>10}")
    print("-" * 55)

    try:
        sensor = MPU6050()
    except OSError:
        print("MPU6050 연결 오류")
        sys.exit(1)

    comp = ComplementaryFilter(alpha=0.97)
    kalman_pitch = SimpleKalmanFilter(Q=0.001, R=0.03)
    kalman_roll = SimpleKalmanFilter(Q=0.001, R=0.03)

    # 오프셋 보정 (초기 1초간 평균)
    print("자이로 바이어스 보정 중... (센서를 평평하게 고정)")
    gx_offset, gy_offset, gz_offset = 0, 0, 0
    samples = 100
    for _ in range(samples):
        _, _, _, gx, gy, gz = sensor.read_all()
        gx_offset += gx
        gy_offset += gy
        gz_offset += gz
        time.sleep(0.01)
    gx_offset /= samples
    gy_offset /= samples
    gz_offset /= samples
    print(f"자이로 오프셋: X={gx_offset:.2f} Y={gy_offset:.2f} Z={gz_offset:.2f} °/s")

    try:
        while True:
            ax, ay, az, gx, gy, gz = sensor.read_all()

            # 자이로 오프셋 보정
            gx -= gx_offset
            gy -= gy_offset
            gz -= gz_offset

            # 가속도 기반 각도
            accel_pitch = math.atan2(-ax, math.sqrt(ay*ay + az*az)) * 180.0 / math.pi
            accel_roll = math.atan2(ay, az) * 180.0 / math.pi

            # 상보 필터
            comp_pitch, comp_roll = comp.update(ax, ay, az, gx, gy, gz)

            # 칼만 필터 (Pitch만)
            k_pitch = kalman_pitch.update(accel_pitch, gx)

            now = time.strftime("%H:%M:%S")
            print(f"{now}  {accel_pitch:+8.2f}  {accel_roll:+8.2f}  |"
                  f"{comp_pitch:+8.2f}  {comp_roll:+8.2f}  |"
                  f"{k_pitch:+8.2f}")

            time.sleep(0.02)  # 50Hz
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        sensor.close()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 mpu6050_kalman.py
```

센서를 움직이면 accel_pitch(노이즈 많음)와 comp_filter/칼만필터(부드러움)의 차이를 실시간으로 확인할 수 있습니다.

---

## 5. 확인 및 테스트

### 검증 절차

```
1. I2C 버스 스캔
   → i2cdetect -y -r 1 → AHT20(0x38), SSD1306(0x3C), MPU6050(0x68) 확인

2. AHT20 온습도
   → aht20_sensor.py → 2초 간격 온도/습도 정상 출력
   → 손으로 센서를 감싸면 온도 상승 확인

3. MPU6050 IMU
   → mpu6050_reader.py → 정지 시 Accel Z ≈ 1g 확인
   → 기울이면 Pitch/Roll 변화 확인

4. SSD1306 OLED
   → oled_display.py → 텍스트, 도형, 애니메이션 순차 출력

5. 최종 실습 4-A
   → env_monitor.py → OLED에 온도/습도 실시간 표시

6. 최종 실습 4-B
   → mpu6050_kalman.py → 가속도(노이즈) vs 칼만필터(부드러움) 비교
```

### 문제 해결

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| `i2cdetect`에 장치 안 보임 | 배선 오류 또는 전원 부족 | VCC, GND, SDA, SCL 연결 재확인 |
| AHT20 값이 0 또는 -50 | 통신 오류 | I2C 속도 100kHz로 낮춤 (smbus2에서) |
| MPU6050 값 고정 | 슬립 모드 | `PWR_MGMT_1`에 0x00 쓰기 확인 |
| OLED가 켜지지만 안 보임 | 명암비/대비 설정 | `device.contrast(255)` 설정 |
| OLED I2C 장치 충돌 | 동일 주소 장치 | TCA9548A 멀티플렉서 사용 |
| MPU6050 자이로 드리프트 | 바이어스 미보정 | 초기 오프셋 보정 코드 추가 |
| OLED 화면 깜빡임 | 갱신 속도 너무 빠름 | `time.sleep(0.1)` 추가 |

---

> **다음 모듈**: Module 04 — SPI (MCP3008 ADC, MAX7219 LED Matrix, RC522 RFID)

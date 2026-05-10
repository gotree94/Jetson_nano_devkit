# Module 02 — PWM (Pulse Width Modulation)

> **소요 시간**: 3~4시간  
> **난이도**: ★★☆☆☆  
> **준비물**: RGB LED(공통 캐소드), 저항 330Ω(3개), SG90 서보 모터, L298N 모터 드라이버, DC 모터, PCA9685 PWM 드라이버(선택), 로직 레벨 컨버터(3.3V↔5V)

---

## 1. 개요

PWM(Pulse Width Modulation)은 디지털 신호로 아날로그적인 출력을 만들어내는 기술입니다.  
LED 밝기 제어, 모터 속도 조절, 서보 모터 각도 제어 등에 널리 사용됩니다.

### 학습 목표
- PWM의 듀티 사이클과 주파수 개념을 이해한다
- Jetson Nano에서 하드웨어 PWM을 설정하고 사용할 수 있다
- 소프트웨어 PWM으로 타이밍을 직접 제어할 수 있다
- PWM으로 RGB LED의 색상을 혼합할 수 있다
- 서보 모터를 정확한 각도로 제어할 수 있다
- DC 모터의 속도와 방향을 제어할 수 있다

### 사전 지식
- Module 01의 GPIO 기본 지식
- 주파수와 주기의 관계: $f = 1/T$

---

## 2. 이론

### 2.1 PWM 원리

PWM은 디지털 신호를 빠르게 ON/OFF 반복하여 평균 전압을 제어합니다.

```
듀티 사이클 0%   ──────────────────────  (완전 OFF, 0V)

듀티 사이클 25%  ┏━━┓    ┏━━┓    ┏━━┓    (평균 0.825V)
                 ┃  ┃    ┃  ┃    ┃  ┃
                 ┗━━┻━━━━┻━━┻━━━━┻━━┻━━

듀티 사이클 50%  ┏━━━━━━┓    ┏━━━━━━┓    (평균 1.65V)
                 ┃      ┃    ┃      ┃
                 ┗━━━━━━┻━━━━┗━━━━━━┻━━━━

듀티 사이클 75%  ┏━━━━━━━━━━┓  ┏━━━━━━  (평균 2.475V)
                 ┃          ┃  ┃
                 ┗━━━━━━━━━━┻━━┻━━━━━━━

듀티 사이클 100% ━━━━━━━━━━━━━━━━━━━━━━  (완전 ON, 3.3V)
```

| 용어 | 설명 | 단위 |
|------|------|------|
| **주기 (Period)** | 한 번의 ON/OFF 사이클 시간 | 초 (s) |
| **주파수 (Frequency)** | 1초당 사이클 수 = 1/주기 | 헤르츠 (Hz) |
| **ON 시간** | 한 주기 내에서 HIGH인 시간 | 초 (s) |
| **듀티 사이클 (Duty Cycle)** | ON 시간 / 주기 × 100 | % (0~100) |

**예제**: 주파수 1kHz(주기=1ms), 듀티 사이클 50%
- ON 시간 = 0.5ms, OFF 시간 = 0.5ms
- 3.3V PWM의 평균 전압 = 3.3V × 50% = 1.65V

### 2.2 서보 모터 (SG90)

서보 모터는 PWM 신호로 각도를 제어합니다.

| 특성 | 값 |
|------|-----|
| 동작 전압 | 4.8V~6V |
| 제어 신호 | 50Hz PWM (주기 20ms) |
| 0° 펄스 폭 | 1.0ms (듀티 5%) |
| 90° 펄스 폭 | 1.5ms (듀티 7.5%) |
| 180° 펄스 폭 | 2.0ms (듀티 10%) |
| 최대 토크 | 1.8kg·cm (4.8V) |

```
SG90 PWM:
  20ms (50Hz)
  ┌────────────────────┐
  │                    │
  ┌┛                    └┐
  1.0ms = 0°
  1.5ms = 90°
  2.0ms = 180°
```

> ⚠️ SG90 동작 전압은 5V이므로 Jetson Nano의 3.3V PWM 핀으로는  
> 직접 구동이 어렵습니다. **별도 5V 전원 + 로직 레벨 컨버터**가 필요합니다.

### 2.3 Jetson Nano PWM

Jetson Nano의 40-pin 헤더에는 **전용 PWM 핀이 없습니다**.  
다음 방법 중 하나를 사용합니다:

#### 방법 A: sysfs PWM (Pin 32, GPIO 168)

Jetson Nano SoC에는 PWM 컨트롤러가 내장되어 있으며,  
Pin 32(LCD_BL_PWM, GPIO 168)에 할당할 수 있습니다.

단, 기본값은 GPIO이므로 **pinmux 재설정**이 필요합니다.

#### 방법 B: PCA9685 I2C PWM 드라이버 (권장)

I2C 버스에 연결하여 **16채널 PWM**을 12비트(4096단계) 분해능으로 제공합니다.  
서보 모터 제어에 특히 적합합니다.

```
Jetson Nano                 PCA9685
I2C_1_SDA (Pin 3) ──────── SDA
I2C_1_SCL (Pin 5) ──────── SCL
3.3V (Pin 1)      ──────── VCC
GND (Pin 6)       ──────── GND
5V (Pin 2)        ──────── V+
                            OE ── GND (enable)
```

#### 방법 C: 소프트웨어 PWM

GPIO를 직접 토글하여 PWM을 에뮬레이션합니다.  
정밀도는 떨어지지만 별도 하드웨어 없이 가능합니다.

```python
import time
import Jetson.GPIO as GPIO

def software_pwm(pin, frequency, duty_cycle, duration):
    """
    소프트웨어 PWM
    pin: GPIO 핀 번호
    frequency: PWM 주파수 (Hz)
    duty_cycle: 듀티 사이클 (0.0 ~ 1.0)
    duration: 실행 시간 (초)
    """
    period = 1.0 / frequency
    on_time = period * duty_cycle
    off_time = period * (1.0 - duty_cycle)

    start = time.time()
    while time.time() - start < duration:
        GPIO.output(pin, GPIO.HIGH)
        time.sleep(on_time)
        GPIO.output(pin, GPIO.LOW)
        time.sleep(off_time)
```

### 2.4 DC 모터 제어 (L298N)

H-Bridge 회로를 사용하여 DC 모터의 방향과 속도를 제어합니다.

```
L298N 모터 드라이버 핀맵:

  +12V ────── 외부 전원 (모터용)
  GND  ────── 전원 및 Jetson GND와 공통
  +5V  ────── (출력, Jetson에 연결하지 않음)
  
  IN1 ────── Jetson GPIO (방향 제어)
  IN2 ────── Jetson GPIO (방향 제어)
  ENA ────── Jetson PWM (속도 제어)
  
  OUT1 ────┐
            ├── DC 모터
  OUT2 ────┘

진리표:
  IN1  IN2  ENA  동작
  H    L    PWM  정방향 (PWM 속도)
  L    H    PWM  역방향 (PWM 속도)
  H    H    X    브레이크
  L    L    X    정지
```

---

## 3. 실습 코드

### 실습 2-1: 소프트웨어 PWM으로 LED 밝기 제어

**회로 구성**:
```
Jetson Nano Pin 11 (GPIO 50) ──→ 330Ω ──→ LED ──→ GND
```

**코드**: `soft_pwm_led.py`
```python
#!/usr/bin/env python3
"""
소프트웨어 PWM으로 LED 밝기 제어 (브레드리스 효과)
LED가 서서히 밝아졌다 어두워짐 반복
"""

import Jetson.GPIO as GPIO
import time
import math

LED_PIN = 11  # GPIO 50
FREQ = 200    # 200Hz (5ms 주기)

def setup():
    GPIO.setmode(GPIO.BOARD)
    GPIO.setup(LED_PIN, GPIO.OUT)
    GPIO.output(LED_PIN, GPIO.LOW)

def soft_pwm_smooth(pin, frequency, duration):
    """
    부드러운 PWM 페이드 인/아웃
    사인파 형태로 듀티 사이클 변화
    """
    period = 1.0 / frequency
    start = time.time()

    while time.time() - start < duration:
        # 0~π까지 사인파로 듀티 사이클 0~100%
        elapsed = time.time() - start
        # 4초 주기로 페이드 인/아웃
        duty = (math.sin(elapsed * math.pi * 0.25) + 1.0) * 0.5

        on_time = period * duty
        off_time = period * (1.0 - duty)

        GPIO.output(pin, GPIO.HIGH)
        time.sleep(on_time)
        GPIO.output(pin, GPIO.LOW)
        time.sleep(off_time)

def setup():
    GPIO.setmode(GPIO.BOARD)
    GPIO.setup(LED_PIN, GPIO.OUT)
    GPIO.output(LED_PIN, GPIO.LOW)

def main():
    setup()
    print("소프트웨어 PWM LED 페이드 인/아웃")
    print("Ctrl+C로 종료")

    try:
        while True:
            soft_pwm_smooth(LED_PIN, FREQ, 4.0)
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        GPIO.cleanup()

if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 soft_pwm_led.py
```

### 실습 2-2: PCA9685 I2C PWM 드라이버

**코드**: `pca9685_control.py`

```python
#!/usr/bin/env python3
"""
PCA9685 I2C PWM 드라이버 제어
공식 Adafruit_CircuitPython_ServoKit 라이브러리 사용
"""

import board
import busio
from adafruit_pca9685 import PCA9685
from adafruit_motor import servo
import time

# I2C 버스 초기화 (Jetson Nano I2C_1, Pin 3=SDA, Pin 5=SCL)
i2c = busio.I2C(board.SCL, board.SDA)

# PCA9685 초기화 (기본 주소 0x40)
pca = PCA9685(i2c)
pca.frequency = 50  # 서보 모터 표준 주파수

print("PCA9685 초기화 완료")

# ============================================================
# 서보 모터 제어
# ============================================================
servo1 = servo.Servo(pca.channels[0])
servo2 = servo.Servo(pca.channels[1])

def servo_sweep(servo_motor, start=0, end=180, step=5, delay=0.05):
    """서보 스윕"""
    for angle in range(start, end + 1, step):
        servo_motor.angle = angle
        time.sleep(delay)
    for angle in range(end, start - 1, -step):
        servo_motor.angle = angle
        time.sleep(delay)

print("서보 스윕 시작 (채널 0)")
servo_sweep(servo1)
print("서보 스윕 완료")

# ============================================================
# LED 밝기 제어 (PWM 직접)
# ============================================================
def led_fade(channel, duration=3):
    """PCA9685 PWM 채널로 LED 페이드"""
    for i in range(0, 0xFFFF, 0x100):
        channel.duty_cycle = i
        time.sleep(duration / 256)
    for i in range(0xFFFF, 0, -0x100):
        channel.duty_cycle = i
        time.sleep(duration / 256)

print("LED 페이드 (채널 2)")
led_fade(pca.channels[2])
print("LED 페이드 완료")

pca.deinit()
print("PCA9685 종료")
```

**설치**:
```bash
pip3 install adafruit-circuitpython-pca9685 adafruit-circuitpython-motor
```

### 실습 2-3: sysfs 하드웨어 PWM

Jetson Nano의 Pin 32(GPIO 168)를 PWM 모드로 전환합니다.

> ⚠️ pinmux를 변경해야 합니다. 최신 JetPack에서는 Jetson-IO 도구를 사용합니다.

```bash
# Jetson-IO 실행 (터미널 메뉴)
sudo /opt/nvidia/jetson-io/jetson-io.py

# 메뉴 탐색:
# 1. "Configure 40-pin expansion header" 선택
# 2. "Configure pin 32 for PWM" 선택 (또는 유사 옵션)
# 3. Overlay 적용 후 저장
# 4. 재부팅
```

**코드**: `hardware_pwm_led.py`
```python
#!/usr/bin/env python3
"""
sysfs 하드웨어 PWM 제어
Pin 32 (PWM0) — LED 밝기 제어

사용 전에 jetson-io로 pinmux 재설정 필요!
"""

import os
import time

# PWM sysfs 경로
PWM_CHIP = "/sys/class/pwm/pwmchip0"
PWM_CHANNEL = 0  # Pin 32에 해당하는 PWM 채널

def pwm_export():
    """PWM 내보내기 (sysfs 활성화)"""
    if not os.path.exists(f"{PWM_CHIP}/pwm{PWM_CHANNEL}"):
        with open(f"{PWM_CHIP}/export", "w") as f:
            f.write(str(PWM_CHANNEL))
    time.sleep(0.1)  # 장치 생성 대기

def pwm_unexport():
    """PWM 내보내기 해제"""
    if os.path.exists(f"{PWM_CHIP}/pwm{PWM_CHANNEL}"):
        with open(f"{PWM_CHIP}/unexport", "w") as f:
            f.write(str(PWM_CHANNEL))

def pwm_config(freq_hz, duty_percent):
    """
    PWM 주파수와 듀티 사이클 설정
    freq_hz: 주파수 (Hz)
    duty_percent: 듀티 사이클 (0.0 ~ 100.0)
    """
    period_ns = int(1_000_000_000 / freq_hz)  # 1초 / 주파수 → ns
    duty_ns = int(period_ns * duty_percent / 100.0)

    pwm_dir = f"{PWM_CHIP}/pwm{PWM_CHANNEL}"

    # 주기 설정
    with open(f"{pwm_dir}/period", "w") as f:
        f.write(str(period_ns))

    # 듀티 사이클 설정
    with open(f"{pwm_dir}/duty_cycle", "w") as f:
        f.write(str(duty_ns))

    # polarity (선택): normal = active high
    # with open(f"{pwm_dir}/polarity", "w") as f:
    #     f.write("normal")

def pwm_enable(enable=True):
    """PWM 출력 활성화/비활성화"""
    pwm_dir = f"{PWM_CHIP}/pwm{PWM_CHANNEL}"
    with open(f"{pwm_dir}/enable", "w") as f:
        f.write("1" if enable else "0")

def main():
    print("sysfs 하드웨어 PWM 제어")

    try:
        pwm_export()
        pwm_config(1000, 0)  # 1kHz, 듀티 0%
        pwm_enable(True)
        print("PWM 활성화 완료")

        # 페이드 인
        print("페이드 인...")
        for duty in range(0, 101, 5):
            pwm_config(1000, duty)
            time.sleep(0.1)

        # 페이드 아웃
        print("페이드 아웃...")
        for duty in range(100, -1, -5):
            pwm_config(1000, duty)
            time.sleep(0.1)

    except PermissionError:
        print("Permission denied! root 권한으로 실행하세요.")
        print("  sudo python3 hardware_pwm_led.py")
    except FileNotFoundError as e:
        print(f"PWM 디바이스를 찾을 수 없습니다: {e}")
        print("jetson-io로 pinmux를 재설정했는지 확인하세요.")
    finally:
        pwm_enable(False)
        pwm_unexport()
        print("PWM 정리 완료")

if __name__ == "__main__":
    main()
```

### 실습 2-4: 서보 모터 제어

SG90 서보 모터를 소프트웨어 PWM으로 직접 제어합니다.

**회로 구성**:
```
SG90 서보 모터:
  빨간선 (VCC)  ──→ 5V (Jetson Pin 2 또는 4, 외부 전원 권장)
  갈색선 (GND)  ──→ GND (Jetson Pin 6)
  주황선 (PWM)  ──→ Pin 32 (GPIO 168, 로직레벨컨버터 통해 5V 레벨로)
```

> ⚠️ SG90은 4.8~6V에서 동작합니다. Jetson Nano 3.3V GPIO로는 직접 구동이 불가능합니다.  
> **TXB0108 레벨 시프터** 또는 별도 **3.3V → 5V 레벨 컨버터**가 필요합니다.

**코드**: `servo_control.py`
```python
#!/usr/bin/env python3
"""
서보 모터 제어 (소프트웨어 PWM)
SG90 — 50Hz, 0.5ms~2.4ms 펄스
"""

import Jetson.GPIO as GPIO
import time

SERVO_PIN = 32  # GPIO 168 (PWM 가능 핀)

# SG90 캘리브레이션 값 (펄스 폭)
# 모터마다 차이가 있을 수 있으므로 조정 필요
PULSE_MIN = 0.5   # 0°  (ms)
PULSE_MID = 1.5   # 90° (ms)
PULSE_MAX = 2.4   # 180° (ms)

SERVO_PERIOD = 20.0  # 50Hz = 20ms

def setup():
    GPIO.setmode(GPIO.BOARD)
    GPIO.setup(SERVO_PIN, GPIO.OUT)

def angle_to_pulse(angle):
    """
    각도(0~180)를 펄스 폭(ms)으로 변환
    angle: 0 ~ 180
    """
    # 0° → PULSE_MIN, 180° → PULSE_MAX
    pulse = PULSE_MIN + (angle / 180.0) * (PULSE_MAX - PULSE_MIN)
    return pulse

def set_servo_angle(angle):
    """
    서보 모터를 지정 각도로 회전
    angle: 0 ~ 180
    """
    pulse_ms = angle_to_pulse(angle)
    pulse_s = pulse_ms / 1000.0
    off_s = (SERVO_PERIOD / 1000.0) - pulse_s

    GPIO.output(SERVO_PIN, GPIO.HIGH)
    time.sleep(pulse_s)
    GPIO.output(SERVO_PIN, GPIO.LOW)
    time.sleep(off_s)

def set_servo_continuous(angle, duration=1.0):
    """
    지정 각도를 duration 동안 유지
    """
    start = time.time()
    while time.time() - start < duration:
        set_servo_angle(angle)

def sweep():
    """0°에서 180°까지 스윕"""
    print("서보 스윕 시작")
    for angle in range(0, 181, 5):
        set_servo_continuous(angle, 0.3)
        print(f"  각도: {angle}°")
    for angle in range(180, -1, -5):
        set_servo_continuous(angle, 0.3)
        print(f"  각도: {angle}°")
    print("서보 스윕 완료")

def demo():
    """데모 시퀀스"""
    print("\n서보 모터 데모")
    print("  1. 90° 중앙 정렬")
    set_servo_continuous(90, 2.0)

    print("  2. 0° → 180° → 0°")
    sweep()

    print("  3. 포텐셔미터 흉내: 30°, 60°, 120°, 150° 각각 1초 유지")
    for angle in [30, 60, 120, 150]:
        set_servo_continuous(angle, 1.0)
        print(f"    {angle}° 유지")

def main():
    setup()
    try:
        demo()
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        GPIO.output(SERVO_PIN, GPIO.LOW)
        GPIO.cleanup()
        print("GPIO 정리 완료")

if __name__ == "__main__":
    main()
```

---

## 4. 최종 실습 — RGB LED 색상 혼합 + 서보 제어

### 회로 구성

**RGB LED (공통 캐소드)**:
```
RGB LED (Common Cathode):
        ┌──┐
  R ────┤  │
  G ────┤  ├──── GND
  B ────┤  │
        └──┘

연결:
  Pin 11 (GPIO 50) ──→ 330Ω ──→ R (빨강)
  Pin 13 (GPIO 14) ──→ 330Ω ──→ G (초록)
  Pin 15 (GPIO 194) ──→ 330Ω ──→ B (파랑)
  
  RGB 공통 핀 ──→ GND
```

**서보 모터**:
```
  Pin 32 (GPIO 168) ──→ 레벨 컨버터 ──→ SG90 PWM (주황)
  SG90 VCC (빨강) ──→ 5V (외부 전원)
  SG90 GND (갈색) ──→ GND 공통
```

### 코드: `rgb_servo_project.py`

```python
#!/usr/bin/env python3
"""
RGB LED 색상 혼합 + 서보 모터 제어
- 포텐셔미터 대신 키보드 입력으로 제어
- R/G/B 각각 소프트웨어 PWM으로 밝기 제어
- 서보는 0°~180° 스윕

실행: sudo python3 rgb_servo_project.py
"""

import Jetson.GPIO as GPIO
import time
import threading
import sys

# ============================================================
# 핀 설정
# ============================================================
RGB_PINS = {
    'R': 11,   # GPIO 50
    'G': 13,   # GPIO 14
    'B': 15,   # GPIO 194
}
SERVO_PIN = 32  # GPIO 168

# ============================================================
# PWM 설정
# ============================================================
PWM_FREQ = 200          # 200Hz

# 서보 설정
SERVO_FREQ = 50          # 50Hz (20ms)
PULSE_MIN = 0.5          # 0° 펄스 (ms)
PULSE_MAX = 2.4          # 180° 펄스 (ms)
SERVO_PERIOD_MS = 20.0   # 50Hz = 20ms

# RGB 값
rgb_values = {'R': 255, 'G': 0, 'B': 0}
current_angle = 90
servo_direction = 1
running = True


# ============================================================
# 소프트웨어 PWM 클래스
# ============================================================

class SoftwarePWM:
    """
    소프트웨어 PWM — 별도 스레드에서 실행
    지정된 GPIO 핀을 주어진 주파수와 듀티 사이클로 토글
    """

    def __init__(self, pin, frequency=200, initial_duty=0):
        self.pin = pin
        self.period = 1.0 / frequency
        self.duty = initial_duty  # 0.0 ~ 1.0
        self._running = False
        self._thread = None

    def start(self):
        self._running = True
        self._thread = threading.Thread(target=self._run, daemon=True)
        self._thread.start()

    def stop(self):
        self._running = False
        if self._thread:
            self._thread.join(timeout=1.0)

    def set_duty(self, duty):
        """듀티 사이클 설정 (0.0 ~ 1.0)"""
        self.duty = max(0.0, min(1.0, duty))

    def _run(self):
        while self._running:
            if self.duty <= 0.0:
                time.sleep(self.period)  # 완전 OFF
            elif self.duty >= 1.0:
                GPIO.output(self.pin, GPIO.HIGH)
                time.sleep(self.period)  # 완전 ON
            else:
                on_time = self.period * self.duty
                off_time = self.period * (1.0 - self.duty)
                GPIO.output(self.pin, GPIO.HIGH)
                time.sleep(on_time)
                GPIO.output(self.pin, GPIO.LOW)
                time.sleep(off_time)


# ============================================================
# RGB 제어
# ============================================================

def setup_rgb():
    GPIO.setmode(GPIO.BOARD)
    for pin in RGB_PINS.values():
        GPIO.setup(pin, GPIO.OUT)
        GPIO.output(pin, GPIO.LOW)


def setup_servo():
    GPIO.setup(SERVO_PIN, GPIO.OUT)
    GPIO.output(SERVO_PIN, GPIO.LOW)


def set_servo_angle(angle):
    """
    서보 모터 각도 제어 (소프트웨어 PWM)
    50Hz에서 angle에 해당하는 펄스 폭 생성
    """
    if angle < 0:
        angle = 0
    elif angle > 180:
        angle = 180

    pulse_ms = PULSE_MIN + (angle / 180.0) * (PULSE_MAX - PULSE_MIN)
    pulse_s = pulse_ms / 1000.0
    off_s = (SERVO_PERIOD_MS / 1000.0) - pulse_s

    # 연속 펄스 생성 (100ms 동안 유지)
    start = time.time()
    while time.time() - start < 0.1:
        GPIO.output(SERVO_PIN, GPIO.HIGH)
        time.sleep(pulse_s)
        GPIO.output(SERVO_PIN, GPIO.LOW)
        time.sleep(off_s)


def set_rgb(r, g, b):
    """
    RGB 값 설정 (0~255)
    실제 PWM 듀티 사이클로 변환
    """
    rgb_values['R'] = r
    rgb_values['G'] = g
    rgb_values['B'] = b


# ============================================================
# 키보드 입력 처리
# ============================================================

def print_menu():
    print("""
===== RGB + Servo Control Menu ====
  R+ / r-  : Red 증가/감소
  G+ / g-  : Green 증가/감소
  B+ / b-  : Blue 증가/감소
  
  S      : Servo 스윕 토글
  C      : 색상 프리셋 순환
  Q      : 종료
====================================
""")


def input_thread_func():
    """키보드 입력 처리 스레드"""
    global current_angle, servo_direction, running

    presets = [
        (255, 0, 0),     # Red
        (0, 255, 0),     # Green
        (0, 0, 255),     # Blue
        (255, 255, 0),   # Yellow
        (0, 255, 255),   # Cyan
        (255, 0, 255),   # Magenta
        (255, 128, 0),   # Orange
        (128, 0, 255),   # Purple
        (255, 255, 255), # White
    ]
    preset_idx = 0

    while running:
        cmd = input().strip().lower()

        if cmd == 'q':
            print("종료 중...")
            running = False
            break
        elif cmd == 'r+':
            set_rgb(min(255, rgb_values['R'] + 32), rgb_values['G'], rgb_values['B'])
        elif cmd == 'r-':
            set_rgb(max(0, rgb_values['R'] - 32), rgb_values['G'], rgb_values['B'])
        elif cmd == 'g+':
            set_rgb(rgb_values['R'], min(255, rgb_values['G'] + 32), rgb_values['B'])
        elif cmd == 'g-':
            set_rgb(rgb_values['R'], max(0, rgb_values['G'] - 32), rgb_values['B'])
        elif cmd == 'b+':
            set_rgb(rgb_values['R'], rgb_values['G'], min(255, rgb_values['B'] + 32))
        elif cmd == 'b-':
            set_rgb(rgb_values['R'], rgb_values['G'], max(0, rgb_values['B'] - 32))
        elif cmd == 'c':
            preset_idx = (preset_idx + 1) % len(presets)
            r, g, b = presets[preset_idx]
            set_rgb(r, g, b)
            print(f"  프리셋 {preset_idx}: RGB({r}, {g}, {b})")
        elif cmd == 's':
            servo_direction *= -1
            print(f"  서보 방향 변경: {'→' if servo_direction == 1 else '←'}")
        elif cmd == 'h':
            print_menu()

        # 현재 상태 출력
        print(f"  RGB({rgb_values['R']:3d}, {rgb_values['G']:3d}, {rgb_values['B']:3d})  "
              f"Servo: {current_angle:3d}°")


# ============================================================
# 메인
# ============================================================

def main():
    global current_angle, servo_direction, running

    print("RGB LED + 서보 모터 제어 프로그램")
    print("=" * 40)

    setup_rgb()
    setup_servo()
    print_menu()

    # RGB PWM 스레드 생성
    pwm_r = SoftwarePWM(RGB_PINS['R'], PWM_FREQ, 0.0)
    pwm_g = SoftwarePWM(RGB_PINS['G'], PWM_FREQ, 0.0)
    pwm_b = SoftwarePWM(RGB_PINS['B'], PWM_FREQ, 0.0)

    pwm_r.start()
    pwm_g.start()
    pwm_b.start()

    # 키보드 입력 스레드
    input_thread = threading.Thread(target=input_thread_func, daemon=True)
    input_thread.start()

    try:
        while running:
            # RGB 값 → PWM 듀티 업데이트
            pwm_r.set_duty(rgb_values['R'] / 255.0)
            pwm_g.set_duty(rgb_values['G'] / 255.0)
            pwm_b.set_duty(rgb_values['B'] / 255.0)

            # 서보 스윕
            current_angle += servo_direction
            if current_angle >= 180:
                current_angle = 180
                servo_direction = -1
            elif current_angle <= 0:
                current_angle = 0
                servo_direction = 1

            set_servo_angle(current_angle)
            time.sleep(0.02)  # ~50fps 제어

    except KeyboardInterrupt:
        print("\n종료 중...")
    finally:
        running = False
        pwm_r.stop()
        pwm_g.stop()
        pwm_b.stop()
        GPIO.output(SERVO_PIN, GPIO.LOW)
        for pin in RGB_PINS.values():
            GPIO.output(pin, GPIO.LOW)
        GPIO.cleanup()
        print("GPIO 정리 완료")


if __name__ == "__main__":
    main()
```

**실행**:
```bash
sudo python3 rgb_servo_project.py
```

**사용법**:
```
  R+ / r-  : Red 채널 밝기 증가/감소
  G+ / g-  : Green 채널 밝기 증가/감소
  B+ / b-  : Blue 채널 밝기 증가/감소
  S        : 서보 스윕 방향 전환
  C        : 색상 프리셋 순환 (Red→Green→Blue→Yellow→...)
  H        : 메뉴 출력
  Q        : 종료
```

---

## 5. 확인 및 테스트

### 검증 절차

```
1. 소프트웨어 PWM LED
   → soft_pwm_led.py 실행 → LED가 부드럽게 페이드 인/아웃

2. 서보 모터
   → servo_control.py 실행 → 0°→180° 스윕 후 30°,60°,120°,150° 순차 표시

3. RGB 최종 실습
   → sudo python3 rgb_servo_project.py 실행
   → R+ / G+ / B+ 로 각 채널 밝기 조절 → 색상 변화 확인
   → C로 프리셋 순환 확인
   → 서보가 0°~180° 반복 스윕
```

### 문제 해결

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| LED가 전혀 안 켜짐 | 소프트웨어 PWM 스레드 미실행 | `pwm_r.start()` 호출 확인 |
| RGB 색상이 이상함 | PWM 주파수太低(너무 낮음) | `PWM_FREQ`를 500~1000Hz로 증가 |
| 서보가 떨림 | 펄스 타이밍 불안정 | 실시간 스레드 우선순위 높이기 |
| 서보가 동작 안 함 | 전압 부족 | 외부 5V 전원 사용 (Jetson 5V 핀은 전류 제한) |
| Permission denied | GPIO 권한 부족 | `sudo`로 실행하거나 gpio 그룹 추가 |
| PCA9685 not found | I2C 주소 틀림 | `i2cdetect -y -r 1`로 주소 확인 |
| RGB LED가 너무 밝음 | 저항 값 부족 | 330Ω 대신 470Ω~1kΩ 사용 |
| 서보가 180°까지 안 감 | 펄스 범위 부족 | `PULSE_MAX`를 2.5ms까지 조정 |

---

> **다음 모듈**: Module 03 — I2C (AHT20, MPU6050, SSD1306 OLED)

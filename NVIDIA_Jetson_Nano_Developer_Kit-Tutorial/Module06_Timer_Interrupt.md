# Module 06 — 타이머 / 카운터 / 인터럽트

> **소요 시간**: 3~4시간  
> **난이도**: ★★★☆☆  
> **준비물**: HC-SR04 초음파 거리 센서, KY-040 로터리 엔코더, SSD1306 OLED(선택, Module 03 재사용), SG90 서보(Module 02 재사용)

---

## 1. 개요

타이머와 인터럽트는 실시간 시스템의 핵심 요소입니다.  
정확한 시간 측정, 주기적인 작업 실행, 외부 이벤트 감지를 가능하게 합니다.

### 학습 목표
- Linux에서 고정밀 타이머(`timerfd`, `clock_gettime`)를 사용할 수 있다
- 주기 태스크를 구현하고 스케줄링할 수 있다
- 하드웨어 타이머(WDT, PWM 타이머)의 개념을 이해한다
- GPIO 인터럽트의 엣지 트리거와 우선순위를 이해한다
- HC-SR04 초음파 센서로 거리를 측정할 수 있다 (인터럽트 기반)
- 로터리 엔코더(KY-040)로 회전 방향과 속도를 계산할 수 있다

### 사전 지식
- Module 01의 GPIO 인터럽트 기초
- Module 02의 서보 모터 제어

---

## 2. 이론

### 2.1 Linux 타이머 메커니즘

Linux는 여러 가지 타이머 인터페이스를 제공합니다:

| 인터페이스 | 정밀도 | 용도 |
|-----------|--------|------|
| `sleep()` / `usleep()` | ms / µs | 단순 대기 (정밀도 낮음) |
| `nanosleep()` | ns | 고정밀 대기 (신호 안전) |
| `clock_gettime()` | ns | 고정밀 시간 측정 |
| `timerfd` | ns | 파일 디스크립터 기반 타이머 (select/poll 호환) |
| `setitimer` | µs | 주기적 시그널 타이머 |
| `timer_create` | ns | POSIX 주기 타이머 |

**timerfd (권장)**:
```c
#include <sys/timerfd.h>

int fd = timerfd_create(CLOCK_MONOTONIC, 0);
struct itimerspec spec = {
    .it_interval = {.tv_sec = 0, .tv_nsec = 100000000},  // 주기: 100ms
    .it_value    = {.tv_sec = 0, .tv_nsec = 100000000},   // 초기 만료
};
timerfd_settime(fd, 0, &spec, NULL);

// read()가 만료될 때마다 1회 블로킹 해제
uint64_t expirations;
read(fd, &expirations, sizeof(expirations));
```

**clock_gettime (고정밀 시간 측정)**:
```python
import time

# Python의 time 모듈
start = time.perf_counter()  # 또는 time.monotonic()
# ... 작업 ...
elapsed = time.perf_counter() - start

# timeit 모듈 (벤치마크)
import timeit
execution_time = timeit.timeit('"-".join(str(n) for n in range(100))', number=10000)
```

### 2.2 GPIO 인터럽트 심화

Module 01에서 배운 GPIO 인터럽트를 더 깊이 이해합니다.

**엣지 트리거 종류**:
```
상승 엣지 (RISING):     ━━━┓        ┏━━━
                           ┃        ┃
                           ┗━━━━━━━━┛

하강 엣지 (FALLING):    ━━━┓        ┏━━━
                           ┃        ┃
                           ┗━━━━━━━━┛

양쪽 엣지 (BOTH):       ━━━┓        ┏━━━
                           ┃        ┃
                           ┗━━━━━━━━┛
```

**인터럽트 vs 폴링**:
| 항목 | 인터럽트 | 폴링 |
|------|---------|------|
| CPU 사용률 | 낮음 (이벤트 발생 시만) | 높음 (계속 확인) |
| 응답 시간 | 즉시 | 샘플링 주기에 의존 |
| 구현 복잡도 | 중간 | 낮음 |
| 다중 채널 | 제한적 | 용이 |

### 2.3 하드웨어 타이머

**Watchdog Timer (WDT)**:
시스템이 Hang 상태에 빠지면 자동으로 재부팅하는 하드웨어 타이머입니다.

```
정상 동작: ── keepalive ── keepalive ── keepalive ──
              (리셋)        (리셋)        (리셋)

Hang 발생: ── keepalive ── XXXXXXXXXX (타이머 만료!)
                                         ↓
                                      시스템 리셋
```

Linux watchdog 인터페이스:
```bash
# /dev/watchdog 열기 → 타이머 시작
# 주기적으로 write() → 타이머 리셋
# write() 중단 → 60초 후 리셋 (기본 타임아웃)
echo 'V' > /dev/watchdog  # keepalive 신호
```

### 2.4 HC-SR04 초음파 거리 센서

HC-SR04는 초음파를 발사하여 반사파가 돌아오는 시간으로 거리를 측정합니다.

| 특성 | 값 |
|------|-----|
| 동작 전압 | 5V |
| 측정 범위 | 2cm ~ 400cm |
| 측정 정밀도 | ±3mm |
| 측정 각도 | 15° |
| 동작 주파수 | 40kHz |

**동작 순서**:
```
1. Trigger (TRIG): 10µs HIGH 펄스 → 초음파 발사
2. Echo (ECHO): 초음파 발사 후 HIGH → 반사파 수신 후 LOW
3. 거리 계산: Echo HIGH 시간 × 343m/s ÷ 2

  TRIG: ━━━┓  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━
            ┃  ┃
            ┗━━┛
            10µs

  ECHO: ━━━━┓                              ┏━━━
              ┃  ↑ 거리 × 2 / 음속          ┃
              ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
              └─── pulse_width ────────────┘

  거리 = pulse_width × 34300 (cm/s) ÷ 2
       = pulse_width × 17150 (cm/s)
```

> ⚠️ HC-SR04는 **5V 로직**입니다. Echo 핀에서 5V 출력이 나오므로  
> **전압 분배기**로 3.3V로 낮춰야 Jetson Nano GPIO가 손상되지 않습니다.

**전압 분배 회로**:
```
HC-SR04 Echo (5V) ──→ 2.2kΩ ──┬──→ Jetson GPIO (3.3V)
                               │
                               └── 3.3kΩ ──→ GND
```

### 2.5 KY-040 로터리 엔코더

로터리 엔코더는 회전 방향과 각도를 전기적 펄스로 변환합니다.

**내부 구조**:
```
       A ──────┐            ┌──────┐
                └────────────┘      └────
       B ──┐        ┌──────┐
           └────────┘      └────────────
             CW (시계 방향)      CCW (반시계)
             A 선행               B 선행
```

| 핀 | 설명 |
|----|------|
| CLK (A) | 회전 시 펄스 출력 (주 채널) |
| DT (B) | 회전 시 펄스 출력 (CLK와 90° 위상차) |
| SW | 스위치 (누름) — 일반 버튼 |
| VCC | 3.3V |
| GND | GND |

**회전 방향 판별**:
```python
# CLK(A) 상승 엣지에서 DT(B) 상태 확인
if GPIO.input(DT_PIN) == GPIO.HIGH:
    direction = "CW"   # 시계 방향
else:
    direction = "CCW"  # 반시계 방향
```

---

## 3. 실습 코드

### 실습 6-1: 고정밀 타이머 (timerfd)

**코드**: `precision_timer.py`
```python
#!/usr/bin/env python3
"""
고정밀 타이머 — timerfd 스타일 구현
Python에서 select.select()로 타임아웃 구현
정확한 주기 태스크 실행
"""

import time
import select
import sys


class PrecisionTimer:
    """
    고정밀 주기 타이머

    Python의 time.perf_counter() + select.sleep() 기반
    주기 태스크를 정확한 간격으로 실행 (드리프트 보정)
    """

    def __init__(self, interval, callback):
        """
        interval: 주기 (초)
        callback: 실행할 함수
        """
        self.interval = interval
        self.callback = callback
        self._running = False
        self._next_tick = 0
        self._tick_count = 0
        self._drift_total = 0

    def start(self):
        """타이머 시작"""
        self._running = True
        self._next_tick = time.perf_counter() + self.interval
        self._run()

    def stop(self):
        self._running = False

    def _run(self):
        """타이머 메인 루프 — 드리프트 보정 포함"""
        while self._running:
            now = time.perf_counter()
            sleep_time = self._next_tick - now

            if sleep_time > 0:
                time.sleep(sleep_time)
            elif sleep_time < -self.interval:
                # 한 주기 이상 지연됨 → 다음 주기로 스킵
                print(f"[타이머] 드리프트 큼 ({sleep_time*1000:.1f}ms), 스킵")
                self._next_tick = now
                continue

            # 콜백 실행
            t0 = time.perf_counter()
            self.callback(self._tick_count)
            elapsed = time.perf_counter() - t0

            # 드리프트 계산 및 보정
            actual_interval = time.perf_counter() - (self._next_tick - self.interval)
            drift = actual_interval - self.interval
            self._drift_total += drift
            self._tick_count += 1

            # 다음 틱 = 이전 예정 시각 + 주기
            self._next_tick += self.interval

            # 10틱마다 통계 출력
            if self._tick_count % 10 == 0:
                avg_drift = self._drift_total / self._tick_count
                print(f"[타이머] 틱:{self._tick_count:4d}  "
                      f"간격:{actual_interval*1000:.3f}ms  "
                      f"드리프트:{drift*1000:.3f}ms  "
                      f"평균:{avg_drift*1000:.4f}ms")


def timer_callback(tick):
    """타이머 콜백 예제"""
    print(f"[{tick:4d}] Hello at {time.strftime('%H:%M:%S.%f')[:-3]}")


def main():
    print("고정밀 타이머 테스트")
    print("  주기: 500ms")
    print("  Ctrl+C로 종료")
    print("-" * 50)

    timer = PrecisionTimer(interval=0.5, callback=timer_callback)

    try:
        timer.start()
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        timer.stop()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 precision_timer.py
```

### 실습 6-2: HC-SR04 초음파 거리 측정 (폴링 방식)

**회로 구성**:
```
HC-SR04:
  VCC ──→ 5V (Jetson Pin 2)
  GND ──→ GND
  TRIG ──→ Pin 11 (GPIO 50)
  ECHO ──→ 전압 분배기(2.2kΩ+3.3kΩ) ──→ Pin 13 (GPIO 14)
```

**코드**: `hcsr04_polling.py`
```python
#!/usr/bin/env python3
"""
HC-SR04 초음파 거리 센서 — 폴링 방식
TRIG에 10µs 펄스 → ECHO HIGH 시간 측정 → 거리 계산
"""

import Jetson.GPIO as GPIO
import time

TRIG_PIN = 11  # GPIO 50
ECHO_PIN = 13  # GPIO 14

SOUND_SPEED = 34300  # cm/s (20°C 기준)

def setup():
    GPIO.setmode(GPIO.BOARD)
    GPIO.setup(TRIG_PIN, GPIO.OUT)
    GPIO.setup(ECHO_PIN, GPIO.IN)
    GPIO.output(TRIG_PIN, GPIO.LOW)
    time.sleep(0.5)  # 센서 안정화

def measure_distance():
    """
    단일 거리 측정

    반환: 거리(cm), 실패 시 None
    """
    # TRIG에 10µs HIGH 펄스
    GPIO.output(TRIG_PIN, GPIO.HIGH)
    time.sleep(0.00001)  # 10µs
    GPIO.output(TRIG_PIN, GPIO.LOW)

    # ECHO가 HIGH가 될 때까지 대기 (최대 100ms)
    timeout = time.perf_counter()
    while GPIO.input(ECHO_PIN) == GPIO.LOW:
        if time.perf_counter() - timeout > 0.1:
            return None  # 타임아웃

    # ECHO가 LOW가 될 때까지 시간 측정
    pulse_start = time.perf_counter()
    while GPIO.input(ECHO_PIN) == GPIO.HIGH:
        if time.perf_counter() - pulse_start > 0.1:
            return None  # 100ms = 약 17m (범위 초과)

    pulse_end = time.perf_counter()

    # 펄스 폭 (초)
    pulse_duration = pulse_end - pulse_start

    if pulse_duration <= 0:
        return None

    # 거리 = 시간 × 음속 ÷ 2 (왕복)
    distance = pulse_duration * SOUND_SPEED / 2.0

    return distance

def main():
    setup()
    print("HC-SR04 초음파 거리 측정 (폴링)")
    print("=" * 40)
    print(f"{'측정':>4} {'거리(cm)':>10} {'시간(µs)':>10} {'상태':>10}")
    print("-" * 40)

    count = 0
    error_count = 0

    try:
        while True:
            dist = measure_distance()
            count += 1

            if dist is None:
                error_count += 1
                print(f"{count:4d} {'--':>10s} {'--':>10s} {'TIMEOUT':>10s}")
            elif dist < 2 or dist > 400:
                error_count += 1
                print(f"{count:4d} {dist:9.2f}cm {'--':>10s} {'OUT OF RANGE':>10s}")
            else:
                # 시간 = 거리 × 2 / 음속
                pulse_us = dist * 2.0 / SOUND_SPEED * 1_000_000
                print(f"{count:4d} {dist:9.2f}cm {pulse_us:9.0f}µs {'OK':>10s}")

            time.sleep(0.5)  # 2Hz 측정

    except KeyboardInterrupt:
        print("\n종료")
        print(f"통계: 총 {count}회 측정, 오류 {error_count}회 "
              f"({error_count/count*100:.1f}%)" if count > 0 else "")
    finally:
        GPIO.cleanup()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 hcsr04_polling.py
```

물체를 가까이/멀리 이동시키며 거리 변화를 확인합니다.

### 실습 6-3: HC-SR04 인터럽트 방식

폴링 대신 GPIO 인터럽트로 ECHO 핀의 엣지를 감지하여 CPU 효율을 높입니다.

**코드**: `hcsr04_interrupt.py`
```python
#!/usr/bin/env python3
"""
HC-SR04 초음파 거리 센서 — 인터럽트 방식
ECHO 핀의 상승/하강 엣지를 인터럽트로 감지
"""

import Jetson.GPIO as GPIO
import time
import threading

TRIG_PIN = 11
ECHO_PIN = 13
SOUND_SPEED = 34300

# 공유 변수 (스레드 간)
pulse_start = 0
pulse_end = 0
echo_received = threading.Event()
measurement_ready = threading.Event()
last_distance = 0


def echo_rising(channel):
    """ECHO 상승 엣지 — 펄스 시작 시간 기록"""
    global pulse_start
    pulse_start = time.perf_counter()


def echo_falling(channel):
    """ECHO 하강 엣지 — 펄스 종료 시간 기록"""
    global pulse_end
    pulse_end = time.perf_counter()
    echo_received.set()


def setup():
    GPIO.setmode(GPIO.BOARD)
    GPIO.setup(TRIG_PIN, GPIO.OUT)
    GPIO.setup(ECHO_PIN, GPIO.IN)
    GPIO.output(TRIG_PIN, GPIO.LOW)

    # 인터럽트 등록
    GPIO.add_event_detect(ECHO_PIN, GPIO.RISING, callback=echo_rising)
    GPIO.add_event_detect(ECHO_PIN, GPIO.FALLING, callback=echo_falling)

    time.sleep(0.5)


def measure_distance():
    """
    인터럽트 기반 거리 측정
    poll_interrupt 방식: TRIG 펄스 → 인터럽트 대기 → 거리 계산
    """
    global last_distance
    echo_received.clear()

    # TRIG 펄스
    GPIO.output(TRIG_PIN, GPIO.HIGH)
    time.sleep(0.00001)
    GPIO.output(TRIG_PIN, GPIO.LOW)

    # 인터럽트 대기 (타임아웃 100ms)
    if echo_received.wait(timeout=0.1):
        pulse_duration = pulse_end - pulse_start
        if 0 < pulse_duration < 0.1:
            distance = pulse_duration * SOUND_SPEED / 2.0
            if 2 <= distance <= 400:
                last_distance = distance
                return distance
    return None


def main():
    setup()
    print("HC-SR04 거리 측정 (인터럽트)")
    print("=" * 40)

    try:
        while True:
            dist = measure_distance()
            if dist:
                print(f"\r거리: {dist:6.2f}cm  "
                      f"시간: {time.strftime('%H:%M:%S')}", end='', flush=True)
            else:
                print(f"\r거리: --.--cm  (타임아웃)", end='', flush=True)
            time.sleep(0.3)
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        GPIO.remove_event_detect(ECHO_PIN)
        GPIO.cleanup()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 hcsr04_interrupt.py
```

### 실습 6-4: KY-040 로터리 엔코더

**회로 구성**:
```
KY-040 로터리 엔코더:
  CLK ──→ Pin 11 (GPIO 50)
  DT  ──→ Pin 13 (GPIO 14)
  SW  ──→ Pin 15 (GPIO 194) (선택)
  VCC ──→ 3.3V (Pin 1)
  GND ──→ GND (Pin 6)
```

**코드**: `rotary_encoder.py`
```python
#!/usr/bin/env python3
"""
KY-040 로터리 엔코더 — 회전 방향/속도 감지
CLK 상승 엣지 인터럽트 → DT 상태로 방향 판별
"""

import Jetson.GPIO as GPIO
import time

CLK_PIN = 11  # GPIO 50
DT_PIN = 13   # GPIO 14
SW_PIN = 15   # GPIO 194 (버튼)

# 엔코더 상태
position = 0
last_position = 0
last_time = time.perf_counter()
speed_rps = 0.0  # 회전/초


def clk_callback(channel):
    """
    CLK 상승 엣지 인터럽트
    DT 상태로 CW/CCW 판별
    """
    global position, last_position, last_time, speed_rps

    now = time.perf_counter()
    dt = now - last_time

    if GPIO.input(DT_PIN) == GPIO.HIGH:
        position += 1  # CW (시계 방향)
        direction = "CW  (+)"

    else:
        position -= 1  # CCW (반시계)
        direction = "CCW (-)"

    # 속도 계산 (회전/초)
    if dt > 0:
        speed_rps = 1.0 / dt
    last_time = now

    print(f"\r위치: {position:+5d}  방향: {direction}  "
          f"속도: {speed_rps:5.1f} steps/s  "
          f"델타: {position - last_position:+4d}", end='', flush=True)

    last_position = position


def sw_callback(channel):
    """버튼 누름 — 위치 리셋"""
    global position
    position = 0
    print(f"\n[리셋] 위치 = 0")


def setup():
    GPIO.setmode(GPIO.BOARD)
    GPIO.setup(CLK_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
    GPIO.setup(DT_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
    GPIO.setup(SW_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)

    # CLK 상승 엣지 인터럽트
    GPIO.add_event_detect(CLK_PIN, GPIO.RISING,
                          callback=clk_callback, bouncetime=1)

    # 버튼 인터럽트
    GPIO.add_event_detect(SW_PIN, GPIO.FALLING,
                          callback=sw_callback, bouncetime=300)

    print("KY-040 로터리 엔코더 시작")
    print("  엔코더를 돌리면 위치 값이 변합니다.")
    print("  버튼을 누르면 위치가 리셋됩니다.")
    print("  Ctrl+C로 종료")
    print("-" * 50)


def main():
    setup()
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        GPIO.remove_event_detect(CLK_PIN)
        GPIO.remove_event_detect(SW_PIN)
        GPIO.cleanup()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 rotary_encoder.py
```

---

## 4. 최종 실습

### 4-A: HC-SR04 + OLED 거리계

초음파 센서로 측정한 거리를 OLED에 실시간 표시합니다.

**코드**: `ultrasonic_ruler.py`
```python
#!/usr/bin/env python3
"""
HC-SR04 초음파 거리계 + OLED 디스플레이
거리를 OLED에 표시 + 시리얼 플로터용 CSV 출력
"""

import Jetson.GPIO as GPIO
import time
from luma.core.interface.serial import i2c
from luma.core.render import canvas
from luma.oled.device import ssd1306
from PIL import ImageFont

# ============================================================
# 하드웨어 설정
# ============================================================
TRIG_PIN = 11
ECHO_PIN = 13
SOUND_SPEED = 34300
TEMP_C = 25  # 현재 온도 (음속 보정용)

# 온도 보정 음속: v = 331.3 × sqrt(1 + T/273.15)
def get_sound_speed(temp_c):
    return 331.3 * (1 + temp_c / 273.15) ** 0.5 * 100  # cm/s


# ============================================================
# HC-SR04 드라이버
# ============================================================

class HCSR04:
    def __init__(self, trig, echo, temp=25):
        self.trig = trig
        self.echo = echo
        self.temp = temp
        GPIO.setmode(GPIO.BOARD)
        GPIO.setup(trig, GPIO.OUT)
        GPIO.setup(echo, GPIO.IN)
        GPIO.output(trig, GPIO.LOW)
        time.sleep(0.3)

    def measure(self):
        """거리 측정 (cm), 실패 시 None"""
        speed = get_sound_speed(self.temp)
        GPIO.output(self.trig, GPIO.HIGH)
        time.sleep(0.00001)
        GPIO.output(self.trig, GPIO.LOW)

        timeout = time.perf_counter()
        while GPIO.input(self.echo) == GPIO.LOW:
            if time.perf_counter() - timeout > 0.1:
                return None

        start = time.perf_counter()
        while GPIO.input(self.echo) == GPIO.HIGH:
            if time.perf_counter() - start > 0.1:
                return None

        duration = time.perf_counter() - start
        dist = duration * speed / 2.0
        return dist if 2 <= dist <= 400 else None

    def cleanup(self):
        GPIO.cleanup()


# ============================================================
# 메인
# ============================================================

class OLEDDisplay:
    def __init__(self):
        serial = i2c(port=1, address=0x3C)
        self.device = ssd1306(serial)
        self.font_num = ImageFont.truetype(
            "/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf", 28)
        self.font_unit = ImageFont.truetype(
            "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", 14)
        self.font_info = ImageFont.truetype(
            "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", 10)

    def update(self, distance, count, temp_c):
        with canvas(self.device) as draw:
            # 상단 정보
            draw.text((0, 0), f"#{count}", font=self.font_info, fill="white")
            draw.text((90, 0), f"{temp_c}C", font=self.font_info, fill="white")

            # 구분선
            draw.line((0, 12, 127, 12), fill="white", width=1)

            if distance:
                # 거리 숫자 (큰 폰트)
                text = f"{distance:3.0f}"
                # 중앙 정렬을 위한 x 위치 계산
                bbox = self.device.draw.textsize(text, font=self.font_num)
                x = (128 - bbox[0]) // 2
                draw.text((x, 18), text, font=self.font_num, fill="white")

                # 단위
                draw.text((100, 38), "cm", font=self.font_unit, fill="white")
            else:
                draw.text((20, 25), "No Echo", font=self.font_unit, fill="white")

            # 하단 막대 그래프 (거리 시각화)
            bar_width = 0
            if distance:
                bar_width = min(128, int(distance * 128 / 100))
            draw.rectangle((0, 56, bar_width, 63), fill="white")

    def close(self):
        self.device.cleanup()


def main():
    print("초음파 거리계 + OLED")
    print("=" * 35)

    try:
        sensor = HCSR04(TRIG_PIN, ECHO_PIN, TEMP_C)
        display = OLEDDisplay()
    except Exception as e:
        print(f"초기화 오류: {e}")
        return

    # 이동 평균 필터
    buffer = []
    FILTER_SIZE = 5

    count = 0
    print("측정 시작 (Ctrl+C로 종료)")
    print("-" * 35)

    try:
        while True:
            dist = sensor.measure()
            count += 1

            # 이동 평균 필터
            if dist:
                buffer.append(dist)
                if len(buffer) > FILTER_SIZE:
                    buffer.pop(0)
                avg_dist = sum(buffer) / len(buffer)
            else:
                avg_dist = None
                if buffer:
                    avg_dist = buffer[-1]  # 마지막 유효값 유지

            # OLED 업데이트
            display.update(avg_dist, count, TEMP_C)

            # 콘솔 출력
            if dist:
                print(f"\r[{count:4d}] 거리: {dist:6.2f}cm  "
                      f"평균: {avg_dist:6.2f}cm  "
                      f"버퍼: {len(buffer)}", end='', flush=True)
            else:
                print(f"\r[{count:4d}] 측정 실패 (타임아웃/범위 초과)", end='', flush=True)

            # 측정 주기: 200ms (5Hz)
            time.sleep(0.2)

    except KeyboardInterrupt:
        print("\n종료")
    finally:
        display.close()
        sensor.cleanup()


if __name__ == "__main__":
    main()
```

### 4-B: 로터리 엔코더 + 서보 모터

로터리 엔코더로 서보 모터 각도를 제어합니다.

**코드**: `encoder_servo.py`
```python
#!/usr/bin/env python3
"""
로터리 엔코더(KY-040) → 서보 모터(SG90) 각도 제어
엔코더 돌리면 서보 각도 0°~180° 변경
버튼 누르면 중앙(90°) 리셋
"""

import Jetson.GPIO as GPIO
import time

# ============================================================
# 핀 설정
# ============================================================
ENC_CLK = 11   # GPIO 50
ENC_DT = 13    # GPIO 14
ENC_SW = 15    # GPIO 194 (버튼)
SERVO_PIN = 32  # GPIO 168 (PWM)

# ============================================================
# 서보 설정
# ============================================================
SERVO_FREQ = 50
PULSE_MIN = 0.5   # 0° (ms)
PULSE_MAX = 2.4   # 180° (ms)
PERIOD_MS = 20.0

# ============================================================
# 엔코더 상태
# ============================================================
servo_angle = 90  # 초기 각도
last_angle_print = 90
position = 0
last_clk_state = GPIO.HIGH
debounce_time = 0


def angle_to_pulse(angle):
    """각도 → 펄스 폭"""
    return PULSE_MIN + (angle / 180.0) * (PULSE_MAX - PULSE_MIN)


def set_servo(angle, duration=0.05):
    """서보 모터 각도 설정"""
    angle = max(0, min(180, angle))
    pulse_ms = angle_to_pulse(angle)
    pulse_s = pulse_ms / 1000.0
    off_s = (PERIOD_MS / 1000.0) - pulse_s

    start = time.perf_counter()
    while time.perf_counter() - start < duration:
        GPIO.output(SERVO_PIN, GPIO.HIGH)
        time.sleep(pulse_s)
        GPIO.output(SERVO_PIN, GPIO.LOW)
        time.sleep(off_s)


def clk_callback(channel):
    """엔코더 CLK 인터럽트 — 각도 조정"""
    global position, servo_angle, debounce_time

    now = time.perf_counter()
    if now - debounce_time < 0.001:  # 1ms 디바운스
        return
    debounce_time = now

    if GPIO.input(ENC_DT) == GPIO.HIGH:
        servo_angle = min(180, servo_angle + 2)  # CW: +2°
    else:
        servo_angle = max(0, servo_angle - 2)   # CCW: -2°


def sw_callback(channel):
    """버튼 — 중앙 리셋"""
    global servo_angle
    servo_angle = 90
    print(f"\n[리셋] 각도 = 90°")


def setup():
    GPIO.setmode(GPIO.BOARD)
    GPIO.setup(ENC_CLK, GPIO.IN, pull_up_down=GPIO.PUD_UP)
    GPIO.setup(ENC_DT, GPIO.IN, pull_up_down=GPIO.PUD_UP)
    GPIO.setup(ENC_SW, GPIO.IN, pull_up_down=GPIO.PUD_UP)
    GPIO.setup(SERVO_PIN, GPIO.OUT)

    GPIO.add_event_detect(ENC_CLK, GPIO.FALLING,
                          callback=clk_callback, bouncetime=1)
    GPIO.add_event_detect(ENC_SW, GPIO.FALLING,
                          callback=sw_callback, bouncetime=300)


def main():
    global servo_angle, last_angle_print

    setup()
    print("로터리 엔코더 → 서보 모터 제어")
    print("=" * 40)
    print("  엔코더를 돌리면 서보 각도가 변경됩니다.")
    print("  버튼을 누르면 90°로 리셋됩니다.")
    print("  Ctrl+C로 종료")
    print("-" * 40)

    # 초기 위치
    set_servo(90, 0.5)

    try:
        while True:
            set_servo(servo_angle, 0.05)

            # 각도 변경 시 출력
            if abs(servo_angle - last_angle_print) >= 2:
                bar = '█' * int(servo_angle / 5) + '░' * (36 - int(servo_angle / 5))
                print(f"\r각도: {servo_angle:3d}°  {bar}", end='', flush=True)
                last_angle_print = servo_angle

    except KeyboardInterrupt:
        print("\n종료")
    finally:
        GPIO.output(SERVO_PIN, GPIO.LOW)
        GPIO.remove_event_detect(ENC_CLK)
        GPIO.remove_event_detect(ENC_SW)
        GPIO.cleanup()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
sudo python3 encoder_servo.py
```

---

## 5. 확인 및 테스트

### 검증 절차

```
1. 고정밀 타이머
   → precision_timer.py → 500ms 간격으로 실행, 드리프트 < 1ms 확인

2. HC-SR04 폴링 방식
   → hcsr04_polling.py → 물체 가까이/멀리 거리 변화 확인
   → 자로 직접 측정하여 오차 확인

3. HC-SR04 인터럽트 방식
   → hcsr04_interrupt.py → 폴링 방식과 동일한 거리 출력 확인

4. KY-040 로터리 엔코더
   → rotary_encoder.py → CW: position 증가, CCW: 감소 확인
   → 버튼: position 리셋 확인

5. 최종 실습 4-A (초음파 거리계)
   → ultrasonic_ruler.py
   → OLED에 거리 숫자 + 막대 그래프 표시 확인
   → 물체 이동 시 실시간 갱신 확인

6. 최종 실습 4-B (엔코더 서보)
   → sudo python3 encoder_servo.py
   → 엔코더 돌리면 서보 각도 0°~180° 변경 확인
   → 버튼 누르면 90° 리셋 확인
```

### 문제 해결

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| HC-SR04 항상 0 | Echo 전압 문제 | 전압 분배기(2.2kΩ+3.3kΩ) 확인 |
| HC-SR04 측정 불규칙 | 음속 미보정 | 온도 보정 함수 사용 |
| 엔코더 방향 반대 | CLK/DT 반대 연결 | 두 핀 교체 |
| 엔코더 값 튐 | 디바운스 부족 | `bouncetime` 증가 또는 하드웨어 RC 필터 |
| 서보 떨림 | PWM 불안정 | 소프트웨어 PWM 주기 더 짧게 |
| OLED 안 켜짐 | I2C 주소 틀림 | `i2cdetect -y -r 1`로 0x3C 확인 |
| OLED 느린 갱신 | 렌더링 지연 | 폰트 크기 줄이기, 캔버스 최소화 |
| 타이머 드리프트 큼 | Python GC 지연 | `niceness` 설정 낮춤 |

---

> **다음 모듈**: Module 07 — 카메라 및 영상 처리 기초 (CSI 카메라, GStreamer, OpenCV)

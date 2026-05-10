# Module 01 — GPIO 디지털 입출력

> **소요 시간**: 3~4시간  
> **난이도**: ★☆☆☆☆  
> **준비물**: 브레드보드, LED(5mm x 5색), 저항 330Ω(5개), 푸시버튼(3개), 10kΩ 풀업 저항(3개), 7-Segment LED(공통 캐소드), 7447 디코더(선택)

---

## 1. 개요

GPIO(General Purpose Input/Output)는 마이크로컨트롤러나 SBC에서 가장 기본적인 하드웨어 제어 인터페이스입니다.  
Jetson Nano의 40-pin 헤더에는 12개 이상의 GPIO 핀이 있으며, 각 핀은 3.3V 로직 레벨로 동작합니다.

### 학습 목표
- Jetson Nano 40-pin 헤더의 핀맵을 이해하고 GPIO 번호를 식별할 수 있다
- `Jetson.GPIO` 라이브러리로 GPIO를 초기화하고 입출력 모드를 설정할 수 있다
- 디지털 출력으로 LED를 점멸할 수 있다
- 디지털 입력으로 버튼 상태를 읽고 디바운싱을 적용할 수 있다
- GPIO 인터럽트(이벤트 감지)를 사용할 수 있다
- 멀티스레드 환경에서 GPIO를 안전하게 제어할 수 있다

### 사전 지식
- Python 기본 문법 (함수, 루프, try/except)
- 디지털 논리 레벨 (HIGH=3.3V, LOW=0V)

---

## 2. 이론

### 2.1 Jetson Nano 40-pin 헤더 (J41)

Jetson Nano의 40-pin 헤더는 Raspberry Pi와 물리적으로 호환되지만,  
GPIO 번호 체계가 **완전히 다릅니다**.

**핀 배열 (리비전 A02)**:
```
          +------------------+------------------+
  Pin  1  |       3.3V DC    |    5.0V DC       |  Pin  2
  Pin  3  | I2C_1_SDA (GPIO) |    5.0V DC       |  Pin  4
  Pin  5  | I2C_1_SCL (GPIO) |      GND         |  Pin  6
  Pin  7  |   GPIO 216 (CLK) | UART_2_TX (GPIO) |  Pin  8
  Pin  9  |       GND        | UART_2_RX (GPIO) |  Pin 10
  Pin 11  |   GPIO  50       | I2S_4_SCLK (GPIO)|  Pin 12
  Pin 13  |   GPIO  14       |      GND         |  Pin 14
  Pin 15  |   GPIO 194       | SPI_2_CS1 (GPIO) |  Pin 16
  Pin 17  |       3.3V DC    | SPI_2_CS0 (GPIO) |  Pin 18
  Pin 19  |  SPI_1_MOSI (GPIO)|      GND        |  Pin 20
  Pin 21  |  SPI_1_MISO (GPIO)| SPI_2_MISO (GPIO)|  Pin 22
  Pin 23  |  SPI_1_SCK (GPIO) | SPI_1_CS0 (GPIO)|  Pin 24
  Pin 25  |       GND        | SPI_1_CS1 (GPIO) |  Pin 26
  Pin 27  |  I2C_0_SDA       | I2C_0_SCL        |  Pin 28
  Pin 29  |   GPIO 149       |       GND        |  Pin 30
  Pin 31  |   GPIO 200       | GPIO 168 (PWM)   |  Pin 32
  Pin 33  |   GPIO  38       |       GND        |  Pin 34
  Pin 35  | I2S_4_LRCK (GPIO)| UART_2_CTS (GPIO)|  Pin 36
  Pin 37  |   GPIO  12       | I2S_4_SDIN (GPIO)|  Pin 38
  Pin 39  |       GND        | I2S_4_SDOUT (GPIO)|  Pin 40
          +------------------+------------------+
```

> ⚠️ **중요**: GPIO 핀에 인가할 수 있는 최대 전압은 **3.3V**입니다.  
> 5V를 직접 인가하면 Jetson Nano가 손상될 수 있습니다.

### 2.2 GPIO 번호 체계

Jetson Nano의 GPIO는 **sysfs GPIO 번호**를 사용합니다.  
Raspberry Pi의 BCM 번호와 직접적인 관계가 없습니다.

| 물리 핀 번호 | sysfs GPIO 번호 | 기본 기능 | 비고 |
|-------------|-----------------|----------|------|
| 7 | 216 | AUDIO_MCLK | GPIO로 사용 가능 |
| 11 | 50 | UART_2_RTS | GPIO로 사용 가능 |
| 12 | 79 | I2S_4_SCLK | GPIO로 사용 가능 |
| 13 | 14 | LCD_TE | GPIO로 사용 가능 |
| 15 | 194 | SPI_2_CS1 | GPIO로 사용 가능 |
| 16 | 232 | SPI_2_CS1 | GPIO로 사용 가능 |
| 18 | 15 | SPI_2_CS0 | GPIO로 사용 가능 |
| 19 | 16 | SPI_1_MOSI | GPIO로 사용 가능 |
| 21 | 17 | SPI_1_MISO | GPIO로 사용 가능 |
| 22 | 13 | SPI_2_MISO | GPIO로 사용 가능 |
| 23 | 18 | SPI_1_SCK | GPIO로 사용 가능 |
| 24 | 19 | SPI_1_CS0 | GPIO로 사용 가능 |
| 26 | 20 | SPI_1_CS1 | GPIO로 사용 가능 |
| 29 | 149 | CAM_AF_EN | GPIO로 사용 가능 |
| 31 | 200 | GPIO_PZ0 | GPIO 전용 |
| 32 | 168 | LCD_BL_PWM | GPIO/PWM |
| 33 | 38 | GPIO_PE6 | GPIO 전용 |
| 35 | 76 | I2S_4_LRCK | GPIO로 사용 가능 |
| 37 | 12 | SPI_2_MOSI | GPIO로 사용 가능 |
| 38 | 77 | I2S_4_SDIN | GPIO로 사용 가능 |
| 40 | 78 | I2S_4_SDOUT | GPIO로 사용 가능 |

### 2.3 Jetson.GPIO 라이브러리

NVIDIA 공식 GPIO 라이브러리로, Raspberry Pi의 RPi.GPIO와 유사한 API를 제공합니다.

**핀 번호 지정 방식**:
- `BOARD` — 물리적 핀 번호 사용 (1~40)
- `BCM` — Broadcom SOC 채널 번호 (RPi 호환, Jetson에서는 제한적)

```python
import Jetson.GPIO as GPIO

# 핀 번호 모드 설정
GPIO.setmode(GPIO.BOARD)  # 물리 핀 번호 사용 (권장)

# 또는
GPIO.setmode(GPIO.BCM)   # sysfs GPIO 번호 사용
```

### 2.4 풀업/풀다운 저항

플로팅 상태(Floating)를 방지하기 위해 풀업 또는 풀다운 저항을 사용합니다.

```
          3.3V              GPIO
            |                 |
          10kΩ              입력
            |                 |
  버튼 --/\/\/-- GND     10kΩ
  (누르면 GND)               |
                            GND
   ↑ 풀업 저항              ↑ 풀다운 저항
   (버튼 안 누르면 HIGH)    (버튼 안 누르면 LOW)
```

Jetson.GPIO는 내부 풀업/풀다운을 지원합니다:
```python
GPIO.setup(pin, GPIO.IN, pull_up_down=GPIO.PUD_UP)   # 풀업
GPIO.setup(pin, GPIO.IN, pull_up_down=GPIO.PUD_DOWN) # 풀다운
```

> ⚠️ Jetson Nano의 TXB0108 레벨 시프터는 내부 풀업/풀다운 설정에 따라  
> 예기치 않은 동작이 발생할 수 있습니다. 외부 풀업/풀다운 저항 사용을 권장합니다.

### 2.5 디바운싱 (Debouncing)

기계적 스위치는 접점이 붙었다 떨어질 때 수 ms 동안 여러 번 신호가 튀는 현상이 발생합니다.

```
  이상적인 신호:  ━━━┓        ┏━━━
                     ┃        ┃
                     ┗━━━━━━━━┛

  실제 신호:     ━┓┏┓┏┓┏┓  ┏┓┏┓┏━
                   ┃┃┃┃┃┃  ┃┃┃┃┃
                   ┗┛┗┛┗┛┗┛┗┛┗┛┗
                     ←튀는 구간→
```

디바운싱 방법:
1. **소프트웨어 지연**: 상태 변화 후 일정 시간(보통 20~50ms) 무시
2. **하드웨어 필터**: RC 로우패스 필터 (저항 + 커패시터)
3. **GPIO 이벤트 debounce**: `add_event_detect()`의 `bouncetime=` 인자

---

## 3. 실습 코드

### 실습 1-1: LED 점멸 (Blink)

**회로 구성**:
```
Jetson Nano Pin 11 (GPIO 50) ──→ 330Ω ──→ LED(Anode) ──→ LED(Cathode) ──→ GND
```

**코드**: `blink.py`
```python
#!/usr/bin/env python3
"""
LED Blink — GPIO 디지털 출력 기본 예제
1초 간격으로 LED를 켰다 껐다 반복
"""

import Jetson.GPIO as GPIO
import time

# 사용할 핀 번호 (BOARD 모드 = 물리 핀 번호)
LED_PIN = 11  # GPIO 50

def setup():
    """GPIO 초기화"""
    GPIO.setmode(GPIO.BOARD)     # 물리 핀 번호 사용
    GPIO.setup(LED_PIN, GPIO.OUT)  # 출력 모드
    GPIO.output(LED_PIN, GPIO.LOW) # 시작 시 OFF

def blink():
    """1초 간격 LED 점멸"""
    try:
        while True:
            print("LED ON")
            GPIO.output(LED_PIN, GPIO.HIGH)  # LED 켜기
            time.sleep(1)

            print("LED OFF")
            GPIO.output(LED_PIN, GPIO.LOW)   # LED 끄기
            time.sleep(1)

    except KeyboardInterrupt:
        print("\n종료합니다.")
    finally:
        GPIO.cleanup()  # GPIO 리소스 해제

if __name__ == "__main__":
    setup()
    blink()
```

**실행**:
```bash
python3 blink.py
```

### 실습 1-2: 버튼 입력 읽기

**회로 구성**:
```
Jetson Nano Pin 13 (GPIO 14) ──→ ─+── 10kΩ ──→ 3.3V
                                   |
                                  ─┴─ 버튼
                                   |
                                  GND
```

**코드**: `button_input.py`
```python
#!/usr/bin/env python3
"""
버튼 입력 읽기 — GPIO 디지털 입력 기본 예제
버튼 상태를 읽어 콘솔에 출력, LED도 함께 제어
"""

import Jetson.GPIO as GPIO
import time

BUTTON_PIN = 13  # 물리 핀 13 (GPIO 14)
LED_PIN = 11     # 물리 핀 11 (GPIO 50)

def setup():
    GPIO.setmode(GPIO.BOARD)
    GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)  # 내부 풀업
    GPIO.setup(LED_PIN, GPIO.OUT)
    GPIO.output(LED_PIN, GPIO.LOW)

def read_button():
    """버튼 상태를 읽어 LED 제어"""
    previous_state = GPIO.HIGH  # 풀업이므로 기본 HIGH

    try:
        while True:
            current_state = GPIO.input(BUTTON_PIN)

            if current_state != previous_state:
                if current_state == GPIO.LOW:
                    print("버튼 눌림 (PRESSED)")
                    GPIO.output(LED_PIN, GPIO.HIGH)
                else:
                    print("버튼 뗌 (RELEASED)")
                    GPIO.output(LED_PIN, GPIO.LOW)
                previous_state = current_state

            time.sleep(0.05)  # 50ms 샘플링 (디바운스 효과)

    except KeyboardInterrupt:
        print("\n종료합니다.")
    finally:
        GPIO.cleanup()

if __name__ == "__main__":
    setup()
    read_button()
```

**실행**:
```bash
python3 button_input.py
```

### 실습 1-3: GPIO 인터럽트 (이벤트 감지)

폴링 방식 대신 이벤트 콜백을 등록하여 CPU 효율을 높입니다.

**코드**: `gpio_interrupt.py`
```python
#!/usr/bin/env python3
"""
GPIO 인터럽트 — 이벤트 감지 콜백 사용
버튼 누를 때마다 LED 토글
"""

import Jetson.GPIO as GPIO
import time

BUTTON_PIN = 13
LED_PIN = 11

# 콜백 함수에서 사용할 변수
led_state = False

def button_callback(channel):
    """
    버튼 인터럽트 콜백
    channel: 인터럽트가 발생한 핀 번호
    """
    global led_state
    led_state = not led_state
    GPIO.output(LED_PIN, GPIO.HIGH if led_state else GPIO.LOW)
    print(f"인터럽트! 핀 {channel} → LED {'ON' if led_state else 'OFF'}")

def setup():
    GPIO.setmode(GPIO.BOARD)
    GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
    GPIO.setup(LED_PIN, GPIO.OUT)
    GPIO.output(LED_PIN, GPIO.LOW)

    # 하강 엣지(Falling edge)에서 인터럽트 등록
    # 버튼 누를 때: HIGH → LOW (하강)
    # bouncetime=300ms: 300ms 내 중복 트리거 방지
    GPIO.add_event_detect(
        BUTTON_PIN,
        GPIO.FALLING,          # 하강 엣지 감지
        callback=button_callback,
        bouncetime=300
    )
    print("인터럽트 등록 완료. 버튼을 눌러보세요.")

def main():
    setup()
    try:
        # 메인 루프는 아무 일도 안 함 (인터럽트가 처리)
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("\n종료합니다.")
    finally:
        GPIO.remove_event_detect(BUTTON_PIN)  # 이벤트 제거
        GPIO.cleanup()

if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 gpio_interrupt.py
```

### 실습 1-4: 멀티스레드 GPIO

서로 다른 주기로 여러 LED를 각각의 스레드에서 제어합니다.

**코드**: `multi_thread_gpio.py`
```python
#!/usr/bin/env python3
"""
멀티스레드 GPIO — 각 LED를 별도 스레드로 제어
LED1: 0.5초 간격
LED2: 1.0초 간격
LED3: 1.5초 간격
버튼 누르면 모든 LED OFF 후 종료
"""

import Jetson.GPIO as GPIO
import threading
import time

# 핀 설정
LED_PINS = [11, 13, 15]     # GPIO 50, 14, 194
BUTTON_PIN = 29              # GPIO 149

# 종료 플래그 (스레드 간 공유)
stop_event = threading.Event()

def led_thread(pin, interval, name):
    """
    LED 제어 스레드 함수
    pin: GPIO 핀 번호
    interval: 점멸 간격 (초)
    name: 스레드 이름 (로깅용)
    """
    while not stop_event.is_set():
        GPIO.output(pin, GPIO.HIGH)
        print(f"[{name}] ON")
        # stop_event.wait()는 타임아웃 동안 중단 요청이 있는지 확인
        if stop_event.wait(interval * 0.5):
            break
        GPIO.output(pin, GPIO.LOW)
        print(f"[{name}] OFF")
        if stop_event.wait(interval * 0.5):
            break
    GPIO.output(pin, GPIO.LOW)  # 종료 시 OFF 보장
    print(f"[{name}] 스레드 종료")

def button_monitor():
    """버튼을 감시하여 누르면 종료 트리거"""
    while not stop_event.is_set():
        if GPIO.input(BUTTON_PIN) == GPIO.LOW:
            print("\n[SYSTEM] 버튼 눌림 → 모든 LED OFF 후 종료")
            stop_event.set()
            break
        time.sleep(0.1)

def setup():
    GPIO.setmode(GPIO.BOARD)
    for pin in LED_PINS:
        GPIO.setup(pin, GPIO.OUT)
        GPIO.output(pin, GPIO.LOW)
    GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)

def main():
    setup()
    print("멀티스레드 GPIO 시작! (버튼 누르면 종료)")

    # LED 스레드 생성
    threads = []
    configs = [
        (LED_PINS[0], 0.5, "LED1(0.5s)"),
        (LED_PINS[1], 1.0, "LED2(1.0s)"),
        (LED_PINS[2], 1.5, "LED3(1.5s)"),
    ]

    for pin, interval, name in configs:
        t = threading.Thread(target=led_thread, args=(pin, interval, name))
        t.daemon = True
        threads.append(t)
        t.start()

    # 버튼 모니터 스레드
    btn_thread = threading.Thread(target=button_monitor)
    btn_thread.daemon = True
    btn_thread.start()

    try:
        # 모든 스레드 종료 대기
        for t in threads:
            t.join()
    except KeyboardInterrupt:
        stop_event.set()
    finally:
        GPIO.cleanup()
        print("GPIO 정리 완료.")

if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 multi_thread_gpio.py
```

---

## 4. 최종 실습 — 7-Segment LED 카운터 + 버튼 토글

### 회로 구성

**7-Segment LED (공통 캐소드) 직접 제어**:

```
Segment 핀맵 (Common Cathode):
         a
       ┏━━━━┓
    f  ┃    ┃  b
       ┣━━━━┫
    e  ┃    ┃  c
       ┗━━━━┛
         d

Jetson Nano 연결:
  Pin 11 (GPIO 50) ──→ 330Ω ──→ a (세그먼트)
  Pin 13 (GPIO 14) ──→ 330Ω ──→ b
  Pin 15 (GPIO 194) ──→ 330Ω ──→ c
  Pin 19 (GPIO 16)  ──→ 330Ω ──→ d
  Pin 21 (GPIO 17)  ──→ 330Ω ──→ e
  Pin 23 (GPIO 18)  ──→ 330Ω ──→ f
  Pin 24 (GPIO 19)  ──→ 330Ω ──→ g

  Pin 29 (GPIO 149) ──→ 10kΩ ──→ 3.3V   (버튼 풀업)
                  └───→ 버튼 ──→ GND
  
  Common Cathode ──→ GND
```

### 코드: `seven_segment_counter.py`

```python
#!/usr/bin/env python3
"""
7-Segment LED 카운터 + 버튼 토글

기능:
- 버튼 누를 때마다 0→1→2→...→9→0 숫자 증가
- 버튼 길게 누르면(1초 이상) 리셋(0)
- 인터럽트 기반 (폴링 없음)
- 디바운스 적용

7-Segment Common Cathode 세그먼트 맵:
    a=0x01, b=0x02, c=0x04, d=0x08,
    e=0x10, f=0x20, g=0x40

숫자별 세그먼트 패턴 (0=OFF, 1=ON):
    0: 0x3F (a b c d e f)
    1: 0x06 (    b c    )
    2: 0x5B (a b   d e g)
    3: 0x4F (a b c d   g)
    4: 0x66 (  b c   f g)
    5: 0x6D (a   c d f g)
    6: 0x7D (a   c d e f g)
    7: 0x07 (a b c      )
    8: 0x7F (a b c d e f g)
    9: 0x6F (a b c d   f g)
"""

import Jetson.GPIO as GPIO
import time

# ============================================================
# 하드웨어 설정
# ============================================================

# 7-Segment 제어 핀 (BOARD 모드)
SEG_PINS = [11, 13, 15, 19, 21, 23, 24]  # a, b, c, d, e, f, g
BUTTON_PIN = 29  # GPIO 149

# 숫자 0~9에 대한 세그먼트 패턴 (a=LSB, g=MSB)
# 각 비트: [g f e d c b a]
SEG_PATTERNS = [
    0b0111111,  # 0: a b c d e f
    0b0000110,  # 1:   b c
    0b1011011,  # 2: a b   d e g
    0b1001111,  # 3: a b c d   g
    0b1100110,  # 4:   b c   f g
    0b1101101,  # 5: a   c d f g
    0b1111101,  # 6: a   c d e f g
    0b0000111,  # 7: a b c
    0b1111111,  # 8: a b c d e f g
    0b1101111,  # 9: a b c d   f g
]


def setup():
    """GPIO 초기화"""
    GPIO.setmode(GPIO.BOARD)

    # 세그먼트 핀을 출력으로 설정
    for pin in SEG_PINS:
        GPIO.setup(pin, GPIO.OUT)
        GPIO.output(pin, GPIO.LOW)

    # 버튼 핀을 입력으로 설정 (내부 풀업)
    GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)


def display_digit(digit):
    """
    숫자(0~9)를 7-Segment에 표시

    digit: 표시할 숫자 (0~9)
    """
    if digit < 0 or digit > 9:
        digit = 0

    pattern = SEG_PATTERNS[digit]

    # 각 세그먼트 핀에 패턴 출력
    for i, pin in enumerate(SEG_PINS):
        # i번째 비트가 1이면 HIGH, 0이면 LOW
        state = GPIO.HIGH if (pattern >> i) & 1 else GPIO.LOW
        GPIO.output(pin, state)


def display_off():
    """모든 세그먼트 OFF"""
    for pin in SEG_PINS:
        GPIO.output(pin, GPIO.LOW)


def digit_clear():
    """잠시 모든 세그먼트 OFF (디바운스 중 시각 피드백)"""
    display_off()


# ============================================================
# 글로벌 상태
# ============================================================

current_digit = 0
last_press_time = 0
debounce_delay = 0.3       # 300ms 디바운스
long_press_threshold = 1.0  # 1초 이상 = 길게 누름
button_pressed = False


def button_callback(channel):
    """
    버튼 인터럽트 콜백
    - 짧게 누름: 숫자 증가
    - 길게 누름(1초+): 리셋(0)
    """
    global current_digit, last_press_time, button_pressed

    now = time.time()

    # 디바운스: 마지막 눌림으로부터 300ms 미만이면 무시
    if now - last_press_time < debounce_delay:
        return

    # 버튼이 눌린 시점 기록
    button_pressed = True
    press_start = now

    # 버튼이 떼질 때까지 대기 (짧은 폴링)
    # 실제로는 별도 타이머로 처리하는 것이 더 좋지만,
    # 콜백 내에서는 단순화
    released = False
    while not released:
        if GPIO.input(BUTTON_PIN) == GPIO.HIGH:  # 버튼 뗌
            elapsed = time.time() - press_start
            if elapsed >= long_press_threshold:
                # 길게 누름 → 리셋
                current_digit = 0
                display_digit(current_digit)
                print(f"[리셋] 숫자 0 (누른 시간: {elapsed:.2f}s)")
            else:
                # 짧게 누름 → 증가
                current_digit = (current_digit + 1) % 10
                display_digit(current_digit)
                print(f"[증가] 숫자 {current_digit}")
            released = True
        time.sleep(0.01)  # 10ms 폴링

    last_press_time = time.time()
    button_pressed = False


def run_with_polling():
    """
    폴링 방식 메인 루프 (인터럽트 대신)
    더 안정적이며 디바운스 제어가 용이함
    """
    global current_digit

    print("7-Segment 카운터 시작!")
    print("  - 버튼 짧게 누름: 숫자 증가 (0→1→2→...→9→0)")
    print("  - 버튼 길게 누름(1초+): 리셋 (0)")
    print("  - Ctrl+C: 종료")
    print(f"\n현재 숫자: {current_digit}")
    display_digit(current_digit)

    prev_button = GPIO.HIGH  # 풀업이므로 기본 HIGH
    press_start_time = 0

    try:
        while True:
            curr_button = GPIO.input(BUTTON_PIN)

            if prev_button == GPIO.HIGH and curr_button == GPIO.LOW:
                # 버튼 눌림 (하강 엣지)
                press_start_time = time.time()
                display_off()  # 시각 피드백

            elif prev_button == GPIO.LOW and curr_button == GPIO.HIGH:
                # 버튼 뗌 (상승 엣지)
                elapsed = time.time() - press_start_time

                if elapsed >= long_press_threshold:
                    # 길게 누름
                    current_digit = 0
                    print(f"[리셋] 숫자 0 (누른 시간: {elapsed:.2f}s)")
                elif elapsed >= 0.05:  # 50ms 미만은 노이즈로 간주
                    # 짧게 누름 (유효한 클릭)
                    current_digit = (current_digit + 1) % 10
                    print(f"[증가] 숫자 {current_digit}")

                display_digit(current_digit)

            prev_button = curr_button
            time.sleep(0.01)  # 10ms 샘플링

    except KeyboardInterrupt:
        print("\n종료합니다.")
    finally:
        display_off()
        GPIO.cleanup()


def run_with_interrupt():
    """
    인터럽트 방식 메인 루프
    단, 콜백 내에서 긴 대기를 하므로 실제로는 폴링 방식보다 비효율적일 수 있음.
    Jetson.GPIO의 인터럽트는 기본적으로 스레드에서 동작합니다.
    """
    print("7-Segment 카운터 시작! (인터럽트 모드)")
    display_digit(0)

    # 인터럽트 등록
    GPIO.add_event_detect(
        BUTTON_PIN,
        GPIO.FALLING,
        callback=button_callback,
        bouncetime=debounce_delay * 1000  # ms 단위
    )

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("\n종료합니다.")
    finally:
        GPIO.remove_event_detect(BUTTON_PIN)
        display_off()
        GPIO.cleanup()


def test_sequence():
    """
    테스트 시퀀스: 모든 세그먼트와 숫자를 순차 표시
    하드웨어 연결 확인용
    """
    print("테스트 모드: 모든 세그먼트 점등")
    display_digit(8)  # 모든 세그먼트 ON
    time.sleep(2)

    print("테스트 모드: 숫자 순차 표시")
    for i in range(10):
        display_digit(i)
        print(f"  숫자 {i}")
        time.sleep(0.5)

    print("테스트 종료")
    display_off()


# ============================================================
# 메인
# ============================================================

if __name__ == "__main__":
    setup()

    # 테스트 시퀀스를 먼저 실행하려면 주석 해제
    # test_sequence()

    # 폴링 방식 실행 (권장)
    run_with_polling()

    # 인터럽트 방식 실행 (대체)
    # run_with_interrupt()
```

### 실행 방법

```bash
# 1. 회로 구성 확인 (연결 상태 검증)
python3 -c "
import Jetson.GPIO as GPIO
GPIO.setmode(GPIO.BOARD)
import time
for pin in [11, 13, 15, 19, 21, 23, 24]:
    GPIO.setup(pin, GPIO.OUT)
    GPIO.output(pin, GPIO.HIGH)
    time.sleep(0.3)
    GPIO.output(pin, GPIO.LOW)
    time.sleep(0.3)
GPIO.cleanup()
print('모든 세그먼트 연결 OK')
"

# 2. 카운터 실행
python3 seven_segment_counter.py
```

---

## 5. 확인 및 테스트

### 하드웨어 연결 검증 절차

```
1. LED 점멸 테스트
   → blink.py 실행 → LED가 1초 간격으로 ON/OFF 반복

2. 버튼 입력 테스트
   → button_input.py 실행 → 버튼 누를 때 콘솔에 PRESSED/RELEASED 출력 확인

3. 인터럽트 테스트
   → gpio_interrupt.py 실행 → 버튼 누를 때마다 LED ON/OFF 토글

4. 멀티스레드 테스트
   → multi_thread_gpio.py 실행 → 3개 LED가 각각 다른 속도로 점멸
   → 버튼 누르면 모두 OFF 후 종료

5. 최종 실습 테스트
   → seven_segment_counter.py 실행
   → 버튼 짧게 누름: 0→1→2→...→9→0 순환
   → 버튼 길게 누름(1초+): 0으로 리셋
```

### 문제 해결

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| LED가 전혀 안 켜짐 | 극성 반대 | LED 긴 다리(Anode)를 GPIO 쪽으로 연결 |
| LED가 매우 어두움 | 저항 값이 큼 | 330Ω 이하 저항 사용 (최소 100Ω) |
| 버튼 입력이 불안정 | 디바운스 부족 | `bouncetime`을 200~500ms로 증가 |
| 버튼 입력이 안 읽힘 | 풀업/풀다운 오류 | 외부 10kΩ 풀업 저항을 3.3V에 연결 |
| `Jetson.GPIO` import 오류 | 라이브러리 미설치 | `pip install Jetson.GPIO` |
| `Permission denied` | GPIO 권한 없음 | gpio 그룹에 사용자 추가 후 재부팅 |
| 7-Segment 숫자가 깨짐 | 세그먼트 핀 순서 오류 | `SEG_PINS` 순서를 실제 연결과 일치시킬 것 |

### 응용 아이디어

- **RGB LED + 버튼**: 버튼 누를 때마다 R→G→B→RGB 순환
- **PIR 센서**: 인체 감지 센서로 LED ON/OFF (GPIO 입력)
- **릴레이 모듈**: GPIO로 220V 가전제품 ON/OFF 제어
- **LCD 1602 + I2C**: GPIO + I2C로 문자 출력 (Module 03 연계)

---

> **다음 모듈**: Module 02 — PWM (RGB LED, 서보 모터, DC 모터)

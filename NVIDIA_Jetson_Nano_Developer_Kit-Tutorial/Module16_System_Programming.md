# Module 16 — 시스템 프로그래밍

> **소요 시간**: 4~5시간  
> **난이도**: ★★★★☆  
> **준비물**: DS3231 RTC 모듈, 점퍼 와이어

---

## 1. 개요

Linux 시스템 수준의 프로그래밍을 다룹니다. Watchdog, RTC, Device Tree, 저수준 GPIO, 부트 체인까지 이해합니다.

### 학습 목표
- Watchdog Timer를 설정하고 테스트할 수 있다
- RTC 모듈을 시스템 시간과 동기화할 수 있다
- Device Tree 구조를 이해하고 Overlay를 작성할 수 있다
- `/dev/mem`으로 저수준 GPIO에 접근할 수 있다
- Jetson Nano의 부트 체인을 이해한다

---

## 2. 실습 코드

### 실습 16-1: Watchdog Timer

```bash
# watchdog 확인
ls -l /dev/watchdog*
cat /dev/watchdog  # (최초 open 시 타이머 시작, 60초 후 리셋)

# watchdog 데몬 설치
sudo apt install -y watchdog
sudo systemctl enable watchdog
sudo systemctl start watchdog
```

**코드**: `watchdog_test.py`
```python
#!/usr/bin/env python3
"""
Watchdog Timer 테스트
/dev/watchdog에 주기적으로 keepalive 쓰기
중단하면 60초 후 시스템 리부트
"""

import os
import time

WDT_PATH = "/dev/watchdog"

def main():
    print("Watchdog Timer 테스트")
    print("  주기적 keepalive 전송 (5초 간격)")
    print("  Ctrl+C로 중단 → 60초 후 리부트")
    print("-" * 40)

    try:
        with open(WDT_PATH, 'wb') as wdt:
            count = 0
            while True:
                wdt.write(b'V')  # keepalive 문자
                wdt.flush()
                count += 1
                print(f"[{count:4d}] Watchdog keepalive sent")
                time.sleep(5)
    except KeyboardInterrupt:
        print("\nWatchdog 중단 — 60초 후 리부트됩니다!")
        print("(실제 리부트를 원하지 않으면 전원 차단)")
        time.sleep(65)

if __name__ == "__main__":
    main()
```

### 실습 16-2: RTC (DS3231)

**회로 구성**:
```
DS3231 RTC:
  VCC ──→ 3.3V
  GND ──→ GND
  SDA ──→ Pin 3
  SCL ──→ Pin 5
```

```bash
# RTC 모듈 확인
i2cdetect -y -r 1
# 0x68 또는 0x57 주소 확인

# DS3231 드라이버 로드
sudo modprobe rtc_ds1307
echo ds3231 0x68 > /sys/bus/i2c/devices/i2c-1/new_device

# RTC 확인
hwclock -r

# 시스템 시간을 RTC에 쓰기
sudo hwclock -w

# 부팅 시 RTC 자동 로드
echo "rtc_ds1307" | sudo tee -a /etc/modules
```

**코드**: `rtc_alarm.py`
```python
#!/usr/bin/env python3
"""
DS3231 RTC 알람 — 특정 시간에 GPIO 트리거
"""

import smbus2
import time
import Jetson.GPIO as GPIO

RTC_ADDR = 0x68
ALARM_PIN = 11

# RTC 시간 읽기
def read_rtc():
    bus = smbus2.SMBus(1)
    data = bus.read_i2c_block_data(RTC_ADDR, 0x00, 7)
    bus.close()
    return {
        'second': bcd_to_dec(data[0]),
        'minute': bcd_to_dec(data[1]),
        'hour': bcd_to_dec(data[2] & 0x3F),
        'day': bcd_to_dec(data[4]),
        'month': bcd_to_dec(data[5] & 0x1F),
        'year': bcd_to_dec(data[6]) + 2000
    }

def bcd_to_dec(bcd):
    return (bcd >> 4) * 10 + (bcd & 0x0F)

def dec_to_bcd(dec):
    return ((dec // 10) << 4) | (dec % 10)

# 알람 설정 (매 분 0초)
def set_alarm():
    bus = smbus2.SMBus(1)
    # Alarm 1: 초 단위 (매 분 30초)
    bus.write_byte_data(RTC_ADDR, 0x07, dec_to_bcd(30))   # 초
    bus.write_byte_data(RTC_ADDR, 0x08, 0x80)             # 분 (무시)
    bus.write_byte_data(RTC_ADDR, 0x09, 0x80)             # 시 (무시)
    bus.write_byte_data(RTC_ADDR, 0x0A, 0x80)             # 일 (무시)
    bus.close()
    print("알람 설정: 매분 30초")

GPIO.setmode(GPIO.BOARD)
GPIO.setup(ALARM_PIN, GPIO.OUT)

while True:
    rtc = read_rtc()
    now = time.localtime()
    print(f"\rRTC: {rtc['hour']:02d}:{rtc['minute']:02d}:{rtc['second']:02d}", end='')
    if rtc['second'] == 30:
        GPIO.output(ALARM_PIN, GPIO.HIGH)
        time.sleep(0.1)
        GPIO.output(ALARM_PIN, GPIO.LOW)
    time.sleep(0.5)
```

---

## 3. 부트 체인

Jetson Nano 부트 순서:

```
전원 ON → CBoot (boot ROM) → U-Boot (2차 부트로더)
  → Kernel (device tree 적용) → initramfs → rootfs (/)


1. CBoot: HW 초기화, pinmux 설정 (device tree 적용)
2. U-Boot: extlinux.conf 읽기, kernel/DTB 로드
3. Kernel: 드라이버 초기화, /sbin/init 실행
4. Systemd: 서비스 시작, getty 실행
```

---

> **모든 모듈 완료!**  
> 이제 Jetson Nano의 GPIO 제어부터 AI 추론까지 전체 과정을 학습했습니다.

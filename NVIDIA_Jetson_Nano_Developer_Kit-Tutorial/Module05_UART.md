# Module 05 — UART (Universal Asynchronous Receiver/Transmitter)

> **소요 시간**: 3~4시간  
> **난이도**: ★★☆☆☆  
> **준비물**: USB-UART 변환기(CP2102/CH340), GPS 수신기(NEO-6M/NEO-8M), HC-06 블루투스 모듈

---

## 1. 개요

UART는 비동기 시리얼 통신의 가장 기본적인 형태로, 두 개의 선(TX, RX)만으로 데이터를 주고받습니다.  
GPS, 블루투스, 디버그 콘솔 등 다양한 장치와의 통신에 사용됩니다.

### 학습 목표
- UART 프로토콜의 프레임 구조(Start/Data/Parity/Stop)를 이해한다
- Jetson Nano의 UART 포트(`/dev/ttyTHS1`)를 식별하고 설정할 수 있다
- `pyserial`로 UART 데이터를 송수신할 수 있다
- GPS NMEA-0183 프로토콜을 파싱하여 위치 정보를 추출할 수 있다
- 블루투스 모듈(HC-06)을 AT 커맨드로 설정할 수 있다
- 시리얼 콘솔을 통해 커널 디버그 메시지를 확인할 수 있다

### 사전 지식
- Module 00의 시리얼 콘솔 접속 경험
- 아스키(ASCII) 코드 이해

---

## 2. 이론

### 2.1 UART 프로토콜

UART는 비동기식이므로 **클럭 선이 없습니다**. 송수신자가 미리 약속된 **Baud rate**로 동기화합니다.

**프레임 구조**:
```
Idle   Start    Bit 0   Bit 1   Bit 2   Bit 3   Bit 4   Bit 5   Bit 6   Bit 7   Stop   Idle
 HIGH    LOW    [LSB]   [..]    [..]    [..]    [..]    [..]    [..]    [MSB]   HIGH    HIGH
```

| 구성 요소 | 설명 |
|----------|------|
| **Start Bit** | 항상 LOW (1비트) — 데이터 시작 신호 |
| **Data Bits** | 5~9비트 (보통 8비트), LSB 우선 전송 |
| **Parity Bit** | 선택 — 오류 검출 (Even/Odd/None) |
| **Stop Bit** | 항상 HIGH (1~2비트) — 데이터 끝 |

**Baud Rate**:
- 1초당 전송되는 심볼(비트)의 수
- 일반적인 값: 9600, 19200, 38400, 57600, 115200
- 양측이 동일한 Baud rate로 설정되어야 함

> ⚠️ **Baud rate ≠ bits per second** (최신 UART에서는 보통 같음)

### 2.2 Jetson Nano UART

Jetson Nano의 40-pin 헤더에는 다음과 같은 UART가 있습니다:

| 포트 | 장치 파일 | 핀 | 용도 |
|------|----------|-----|------|
| **UART_2** | `/dev/ttyTHS1` | TX=Pin 8, RX=Pin 10 | **일반 시리얼 통신** |
| **UART_1** | `/dev/ttyS0` | Pin 27, 28(I2C와 충돌) | 블루투스 모듈용 |
| **디버그 UART** | `/dev/ttyS0` | 보드 헤더(J44) | 커널 디버그 콘솔 |

> **실습에서 사용**: **`/dev/ttyTHS1`** (Pin 8=TX, Pin 10=RX)

**참고**: Jetson Nano의 UART는 **3.3V 로직 레벨**입니다.  
5V 장치(GPS, HC-06 등) 연결 시 레벨 변환이 필요할 수 있습니다.

### 2.3 pyserial 라이브러리

```python
import serial

# 시리얼 포트 열기
ser = serial.Serial(
    port='/dev/ttyTHS1',
    baudrate=115200,
    bytesize=serial.EIGHTBITS,   # 8데이터 비트
    parity=serial.PARITY_NONE,   # 패리티 없음
    stopbits=serial.STOPBITS_ONE, # 1스톱 비트
    timeout=1.0                   # 읽기 타임아웃 (초)
)

# 쓰기
ser.write(b'Hello World\n')

# 읽기 (1바이트)
data = ser.read(1)

# 읽기 (타임아웃까지 모두)
data = ser.read(100)

# 한 줄 읽기 (개행 문자까지)
line = ser.readline()

# 읽을 수 있는 바이트 수 확인
available = ser.in_waiting

# 포트 닫기
ser.close()
```

### 2.4 NMEA-0183 (GPS 프로토콜)

GPS 수신기는 표준 NMEA-0183 형식으로 데이터를 전송합니다.

**주요 NMEA 문장**:

| 문장 | 설명 | 예시 |
|------|------|------|
| **$GPGGA** | 위치 정보 (위도, 경도, 고도) | `$GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47` |
| **$GPRMC** | 권장 최소 정보 (속도, 방위) | `$GPRMC,123519,A,4807.038,N,01131.000,E,022.4,084.4,230394,003.1,W*6A` |
| **$GPGSV** | 위성 정보 | `$GPGSV,3,1,11,10,80,071,46,24,59,096,45,32,54,060,43,20,46,087,*74` |

**$GPGGA 파싱 예시**:
```
$GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47
│      │      │         │ │         │ │ │  │   │    │ │  │   ││  └─ checksum
│      │      │         │ │         │ │ │  │   │    │ │  │   │└─── checksum prefix
│      │      │         │ │         │ │ │  │   │    │ │  │   └──── 고도 단위 (M=미터)
│      │      │         │ │         │ │ │  │   │    │ │  └────── 고도
│      │      │         │ │         │ │ │  │   │    │ └───────── 단위 (M=미터)
│      │      │         │ │         │ │ │  │   │    └─────────── 지오이드 분리
│      │      │         │ │         │ │ │  │   └─────────────── HDOP
│      │      │         │ │         │ │ │  └─────────────────── 사용 위성 수
│      │      │         │ │         │ │ └────────────────────── 품질 (1=GPS, 2=DGPS)
│      │      │         │ │         │ └──────────────────────── 동/서(E/W)
│      │      │         │ │         └────────────────────────── 경도 (DDDMM.MMMM)
│      │      │         │ └──────────────────────────────────── 북/남(N/S)
│      │      │         └────────────────────────────────────── 위도 (DDMM.MMMM)
│      │      └──────────────────────────────────────────────── UTC 시간 (HHMMSS.SS)
│      └─────────────────────────────────────────────────────── 문장 ID
└────────────────────────────────────────────────────────────── 시작 문자
```

**위도/경도 변환** (NMEA 포맷 → 도):

```
위도 DDMM.MMMM (예: 4807.038)
  = DD + MM.MMMM / 60
  = 48 + 07.038 / 60
  = 48.1173° N

경도 DDDMM.MMMM (예: 01131.000)
  = DDD + MM.MMMM / 60
  = 11 + 31.000 / 60
  = 11.5167° E
```

### 2.5 HC-06 블루투스 모듈

HC-06은 UART-블루투스 브릿지 모듈입니다.

| 특성 | 값 |
|------|-----|
| 칩셋 | CSR BC417 |
| 블루투스 버전 | 2.0 (SPP 프로파일) |
| 동작 전압 | 3.6V~6V |
| 통신 거리 | 약 10m |
| 기본 Baud rate | 9600 |
| AT 명령어 | 설정 변경용 |

**HC-06 핀맵**:
```
  ENA ─── (일부 모델, 연결 불필요)
  VCC ──→ 5V (또는 3.3V)
  GND ──→ GND
  TXD ──→ Jetson RX (Pin 10)
  RXD ──→ Jetson TX (Pin 8)
  STATE─→ (LED 상태 표시, 연결 불필요)
```

> ⚠️ HC-06은 **5V 로직**이지만, 3.3V Jetson과 직접 통신 가능한 모델이 많습니다.  
> 불안정할 경우 로직 레벨 컨버터 사용을 권장합니다.

---

## 3. 실습 코드

### 실습 5-1: UART 기본 통신 (루프백)

**회로 구성**:
```
USB-UART 변환기(CP2102):       Jetson Nano:
  TXD ────────────────────────── Pin 10 (UART_RX)
  RXD ────────────────────────── Pin 8  (UART_TX)
  GND ────────────────────────── GND
```

**코드**: `uart_basic.py`
```python
#!/usr/bin/env python3
"""
UART 기본 통신 — PC와 시리얼 통신
Jetson → PC: "Hello from Jetson!" 1초 간격 전송
PC → Jetson: 수신한 데이터를 그대로 출력
"""

import serial
import time

# UART 포트 설정
UART_PORT = '/dev/ttyTHS1'
BAUDRATE = 115200

def main():
    print(f"UART 통신 시작: {UART_PORT} @ {BAUDRATE} bps")
    print("=" * 45)
    print("PC의 시리얼 모니터(115200 baud)에서 확인하세요.")
    print("PC에서 메시지를 보내면 여기에 출력됩니다.")
    print("Ctrl+C로 종료")

    try:
        # 시리얼 포트 열기
        ser = serial.Serial(
            port=UART_PORT,
            baudrate=BAUDRATE,
            bytesize=serial.EIGHTBITS,
            parity=serial.PARITY_NONE,
            stopbits=serial.STOPBITS_ONE,
            timeout=0.1
        )

        count = 0
        while True:
            # 1초 간격 메시지 전송
            message = f"Hello from Jetson! #{count}\n"
            ser.write(message.encode('utf-8'))
            print(f"[송신] {message.strip()}")

            # 수신 데이터 확인
            if ser.in_waiting > 0:
                received = ser.readline().decode('utf-8', errors='replace').strip()
                if received:
                    print(f"[수신] {received}")

            count += 1
            time.sleep(1.0)

    except serial.SerialException as e:
        print(f"시리얼 포트 오류: {e}")
        print(f"  포트 {UART_PORT}가 존재하는지 확인:")
        print(f"  ls -l {UART_PORT}")
        print(f"  사용자 권한: sudo usermod -aG dialout $USER")
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        if 'ser' in locals() and ser.is_open:
            ser.close()
            print("포트 닫힘")


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 uart_basic.py
```

### 실습 5-2: GPS NMEA 파싱

**회로 구성**:
```
GPS 모듈 (NEO-6M):
  VCC ──→ 3.3V (Jetson Pin 1)
  GND ──→ GND (Jetson Pin 6)
  TXD ──→ Jetson Pin 10 (UART_RX)
  RXD ──→ (연결 불필요, GPS는 송신만)
```

**코드**: `gps_nmea_parser.py`
```python
#!/usr/bin/env python3
"""
GPS NMEA-0183 파서
NEO-6M/NEO-8M GPS 모듈에서 데이터 읽기 → 위도/경도/고도/속도 추출
"""

import serial
import time
import re
import math


class NMEAParser:
    """NMEA-0183 문장 파서"""

    def __init__(self):
        self.latitude = 0.0
        self.longitude = 0.0
        self.altitude = 0.0
        self.speed_kmh = 0.0
        self.course = 0.0
        self.utc_time = ""
        self.satellites = 0
        self.fix_quality = 0  # 0=No fix, 1=GPS, 2=DGPS
        self.last_update = 0

    def parse(self, sentence):
        """NMEA 문장 파싱"""
        if not sentence.startswith('$'):
            return False

        # 체크섬 검증 (선택)
        if not self._checksum_verify(sentence):
            return False

        # 문장 타입 확인
        if sentence.startswith('$GPGGA') or sentence.startswith('$GNGGA'):
            return self._parse_gga(sentence)
        elif sentence.startswith('$GPRMC') or sentence.startswith('$GNRMC'):
            return self._parse_rmc(sentence)

        return False

    def _checksum_verify(self, sentence):
        """NMEA 체크섬 검증"""
        star_pos = sentence.find('*')
        if star_pos == -1:
            return True  # 체크섬 없음 → 검증 생략

        try:
            # '$'와 '*' 사이의 XOR 계산
            calc = 0
            for ch in sentence[1:star_pos]:
                calc ^= ord(ch)

            # '*' 이후의 16진수 체크섬
            given = int(sentence[star_pos + 1:star_pos + 3], 16)
            return calc == given
        except (ValueError, IndexError):
            return False

    def _nmea_to_decimal(self, nmea_str, direction):
        """
        NMEA 위도/경도 포맷 → 소수점 도(degree) 변환

        위도: DDMM.MMMM → DD + MM.MMMM/60
        경도: DDDMM.MMMM → DDD + MM.MMMM/60
        """
        if not nmea_str or not direction:
            return 0.0

        nmea_str = nmea_str.strip()
        if len(nmea_str) < 4:
            return 0.0

        try:
            # 소수점 위치 찾기
            dot_pos = nmea_str.find('.')
            if dot_pos == -1:
                return 0.0

            # 위도(DDMM) vs 경도(DDDMM) 구분
            if direction in ['N', 'S']:
                degrees = int(nmea_str[:2])
                minutes = float(nmea_str[2:])
                decimal = degrees + minutes / 60.0
            else:  # E, W
                # 3자리 도
                degree_len = 3
                if len(nmea_str[:dot_pos]) < 3:
                    degree_len = 2
                degrees = int(nmea_str[:degree_len])
                minutes = float(nmea_str[degree_len:])
                decimal = degrees + minutes / 60.0

            if direction in ['S', 'W']:
                decimal = -decimal

            return decimal
        except (ValueError, IndexError):
            return 0.0

    def _parse_gga(self, sentence):
        """
        $GPGGA 파싱
        예: $GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47
        """
        parts = sentence.split(',')

        if len(parts) < 10:
            return False

        try:
            self.utc_time = parts[1]  # HHMMSS.SS
            self.latitude = self._nmea_to_decimal(parts[2], parts[3])
            self.longitude = self._nmea_to_decimal(parts[4], parts[5])
            self.fix_quality = int(parts[6]) if parts[6] else 0
            self.satellites = int(parts[7]) if parts[7] else 0
            self.altitude = float(parts[9]) if parts[9] else 0.0
            self.last_update = time.time()
            return True
        except (ValueError, IndexError):
            return False

    def _parse_rmc(self, sentence):
        """
        $GPRMC 파싱
        예: $GPRMC,123519,A,4807.038,N,01131.000,E,022.4,084.4,230394,003.1,W*6A
        """
        parts = sentence.split(',')

        if len(parts) < 10:
            return False

        try:
            self.utc_time = parts[1]
            status = parts[2]
            if status == 'A':  # A=Active (유효), V=Void(무효)
                self.latitude = self._nmea_to_decimal(parts[3], parts[4])
                self.longitude = self._nmea_to_decimal(parts[5], parts[6])
                self.speed_kmh = float(parts[7]) * 1.852 if parts[7] else 0.0  # Knot → km/h
                self.course = float(parts[8]) if parts[8] else 0.0
                self.last_update = time.time()
            return True
        except (ValueError, IndexError):
            return False

    def has_fix(self):
        """GPS Fix 여부"""
        return self.fix_quality > 0

    def get_position_str(self):
        """위치 문자열 반환"""
        if not self.has_fix():
            return "No fix"

        lat_dir = 'N' if self.latitude >= 0 else 'S'
        lon_dir = 'E' if self.longitude >= 0 else 'W'
        return (f"{abs(self.latitude):.6f}°{lat_dir}, "
                f"{abs(self.longitude):.6f}°{lon_dir}")

    def get_gmaps_url(self):
        """Google Maps URL 생성"""
        if not self.has_fix():
            return ""
        return (f"https://www.google.com/maps?"
                f"q={self.latitude},{self.longitude}")


def main():
    print("GPS NMEA 파서 — 실시간 위치 추적")
    print("=" * 50)

    UART_PORT = '/dev/ttyTHS1'
    GPS_BAUD = 9600  # 대부분 GPS 모듈 기본 9600

    try:
        ser = serial.Serial(
            port=UART_PORT,
            baudrate=GPS_BAUD,
            bytesize=serial.EIGHTBITS,
            parity=serial.PARITY_NONE,
            stopbits=serial.STOPBITS_ONE,
            timeout=0.5
        )
        print(f"GPS 포트 열기: {UART_PORT} @ {GPS_BAUD} bps")
    except serial.SerialException as e:
        print(f"포트 오류: {e}")
        return

    parser = NMEAParser()
    raw_count = 0
    parse_count = 0

    print("GPS 신호 대기 중... (실내에서는 Fix 어려움, 창가/야외 필요)")
    print("-" * 50)

    try:
        while True:
            # NMEA 문장 읽기
            try:
                raw = ser.readline().decode('ascii', errors='replace').strip()
            except UnicodeDecodeError:
                continue

            if not raw:
                continue

            raw_count += 1

            # 파싱
            if parser.parse(raw):
                parse_count += 1

                # 1초마다 상태 출력
                if time.time() - parser.last_update < 1.0:
                    continue

                print(f"\r{'=' * 50}", end='')
                print(f"\n  UTC 시간 : {parser.utc_time}")
                print(f"  위도     : {parser.latitude:.6f}")
                print(f"  경도     : {parser.longitude:.6f}")
                print(f"  위치     : {parser.get_position_str()}")
                print(f"  고도     : {parser.altitude:.1f} m")
                print(f"  속도     : {parser.speed_kmh:.1f} km/h")
                print(f"  방위     : {parser.course:.1f}°")
                print(f"  위성 수  : {parser.satellites}")
                print(f"  Fix 품질 : {parser.fix_quality} "
                      f"({'Fix OK' if parser.has_fix() else 'No Fix'})")
                if parser.has_fix():
                    print(f"  GMaps    : {parser.get_gmaps_url()}")
                print(f"  수신/파싱: {raw_count}/{parse_count} 문장")

                # 잠시 대기 (출력 빈도 제한)
                time.sleep(0.5)
            else:
                # 디버그: 미파싱 문장 출력 (최초 5개)
                if parse_count == 0 and raw_count <= 5:
                    print(f"[RAW] {raw}")

    except KeyboardInterrupt:
        print("\n종료")
    finally:
        if 'ser' in locals() and ser.is_open:
            ser.close()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 gps_nmea_parser.py
```

> GPS Fix는 실내에서 어렵습니다. 창가나 야외에서 테스트하세요.

### 실습 5-3: HC-06 블루투스 AT 커맨드

**회로 구성**:
```
HC-06 블루투스 모듈:
  VCC ──→ 5V (Jetson Pin 2)
  GND ──→ GND
  TXD ──→ Pin 10 (UART_RX)
  RXD ──→ Pin 8  (UART_TX)
```

**코드**: `hc06_at.py`
```python
#!/usr/bin/env python3
"""
HC-06 블루투스 AT 커맨드 설정
기본 Baud rate: 9600 (HC-06 기본값)
AT 커맨드 (각 명령 뒤에 \r\n 필수):
  AT          — 응답 확인 (OK)
  AT+VERSION  — 펌웨어 버전
  AT+NAME    — 장치 이름 설정 (예: AT+NAMEJetsonBT)
  AT+PIN     — PIN 코드 설정 (예: AT+PIN4321)
  AT+BAUD    — Baud rate 설정 (예: AT+BAUD8 → 115200)
"""

import serial
import time


class HC06:
    """HC-06 블루투스 모듈 컨트롤러"""

    BAUDRATE_MAP = {
        1: 1200, 2: 2400, 3: 4800, 4: 9600,
        5: 19200, 6: 38400, 7: 57600, 8: 115200,
        9: 230400,
    }

    def __init__(self, port='/dev/ttyTHS1', baud=9600):
        self.port = port
        self.baud = baud
        self.ser = None

    def connect(self):
        """HC-06 연결 및 AT 모드 진입"""
        self.ser = serial.Serial(
            port=self.port,
            baudrate=self.baud,
            bytesize=serial.EIGHTBITS,
            parity=serial.PARITY_NONE,
            stopbits=serial.STOPBITS_ONE,
            timeout=2.0
        )
        time.sleep(0.5)  # 모듈 안정화 대기
        return self

    def send_at(self, command, wait=1.0):
        """
        AT 명령어 전송 및 응답 대기
        command: "AT" 또는 "AT+VERSION" 등
        wait: 응답 대기 시간 (초)
        """
        self.ser.write(f"{command}\r\n".encode())
        time.sleep(wait)
        response = self.ser.read(self.ser.in_waiting).decode(errors='replace')
        return response.strip()

    def get_version(self):
        return self.send_at("AT+VERSION")

    def set_name(self, name):
        return self.send_at(f"AT+NAME{name}")

    def set_pin(self, pin):
        return self.send_at(f"AT+PIN{pin}")

    def set_baud_rate(self, baud_index):
        """
        Baud rate 변경
        baud_index: 1=1200, 2=2400, 3=4800, 4=9600,
                   5=19200, 6=38400, 7=57600, 8=115200, 9=230400
        """
        return self.send_at(f"AT+BAUD{baud_index}")

    def close(self):
        if self.ser and self.ser.is_open:
            self.ser.close()


def main():
    print("HC-06 블루투스 AT 커맨드 설정")
    print("=" * 40)
    print("1. 현재 설정 확인")
    print("2. 장치 이름 변경")
    print("3. PIN 코드 변경")
    print("4. Baud rate 변경")
    print("=" * 40)

    try:
        bt = HC06(port='/dev/ttyTHS1', baud=9600).connect()
    except serial.SerialException as e:
        print(f"연결 오류: {e}")
        return

    # 기본 AT 테스트
    print("\n[AT 테스트]")
    resp = bt.send_at("AT", wait=0.5)
    print(f"  AT → {resp}")

    # 버전 확인
    print("\n[버전 확인]")
    resp = bt.get_version()
    print(f"  AT+VERSION → {resp}")

    # 이름 변경
    print("\n[이름 변경: AT+NAMEJetsonBT]")
    resp = bt.set_name("JetsonBT")
    print(f"  → {resp}")

    # PIN 변경
    print("\n[PIN 변경: AT+PIN4321]")
    resp = bt.set_pin("4321")
    print(f"  → {resp}")

    # Baud rate 변경 (선택)
    # print("\n[Baud rate → 115200]")
    # resp = bt.set_baud_rate(8)
    # print(f"  → {resp}")

    print("\n설정 완료!")
    print("  장치 이름: JetsonBT")
    print("  PIN: 4321")
    print("  스마트폰에서 'JetsonBT'를 검색하세요.")

    bt.close()


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 hc06_at.py
```

> **참고**: HC-06의 AT 명령어는 대소문자를 구분하며, 명령어 뒤에 공백 없이 붙여씁니다.  
> `AT+NAMEJetsonBT` (O), `AT+NAME JetsonBT` (X)

---

## 4. 최종 실습 — GPS 로거 (KML 변환)

GPS 데이터를 1초 간격으로 기록하고, Google Earth에서 볼 수 있는 KML 파일로 변환합니다.

### 코드: `gps_logger.py`

```python
#!/usr/bin/env python3
"""
GPS 로거 — 위치 데이터 기록 및 KML 변환
1초 간격으로 GPS 위치를 기록 → GPX/KML 파일로 저장
"""

import serial
import time
import os
from datetime import datetime


class NMEAParser:
    """GPS NMEA 파서 (실습 5-2와 동일, 간략화)"""

    def __init__(self):
        self.latitude = 0.0
        self.longitude = 0.0
        self.altitude = 0.0
        self.speed_kmh = 0.0
        self.course = 0.0
        self.utc_time = ""
        self.satellites = 0
        self.fix_quality = 0
        self.last_update = 0

    def parse(self, sentence):
        if not sentence or not sentence.startswith('$'):
            return False
        if sentence.startswith('$GPGGA') or sentence.startswith('$GNGGA'):
            return self._parse_gga(sentence)
        if sentence.startswith('$GPRMC') or sentence.startswith('$GNRMC'):
            return self._parse_rmc(sentence)
        return False

    def _nmea_to_decimal(self, val, direction):
        if not val or not direction:
            return 0.0
        try:
            dot = val.find('.')
            if direction in ['N', 'S']:
                d = int(val[:2])
                m = float(val[2:])
            else:
                d = int(val[:3]) if len(val[:dot]) >= 3 else int(val[:2])
                m = float(val[len(str(d)):])
            dec = d + m / 60.0
            if direction in ['S', 'W']:
                dec = -dec
            return dec
        except (ValueError, IndexError):
            return 0.0

    def _parse_gga(self, s):
        p = s.split(',')
        if len(p) < 10:
            return False
        try:
            self.utc_time = p[1]
            self.latitude = self._nmea_to_decimal(p[2], p[3])
            self.longitude = self._nmea_to_decimal(p[4], p[5])
            self.fix_quality = int(p[6]) if p[6] else 0
            self.satellites = int(p[7]) if p[7] else 0
            self.altitude = float(p[9]) if p[9] else 0.0
            self.last_update = time.time()
            return True
        except (ValueError, IndexError):
            return False

    def _parse_rmc(self, s):
        p = s.split(',')
        if len(p) < 8:
            return False
        try:
            if p[2] == 'A':
                self.latitude = self._nmea_to_decimal(p[3], p[4])
                self.longitude = self._nmea_to_decimal(p[5], p[6])
                self.speed_kmh = float(p[7]) * 1.852 if p[7] else 0.0
                self.course = float(p[8]) if p[8] else 0.0
            return True
        except (ValueError, IndexError):
            return False

    def has_fix(self):
        return self.fix_quality > 0

    def get_position_tuple(self):
        return (self.latitude, self.longitude, self.altitude,
                self.speed_kmh, self.course)


class GPSLogger:
    """GPS 로거 — 기록 및 KML 생성"""

    def __init__(self, output_dir="."):
        self.output_dir = output_dir
        self.track_points = []  # [(lat, lon, alt, time), ...]
        self.start_time = datetime.now()

    def add_point(self, lat, lon, alt, timestamp):
        """위치 포인트 추가"""
        if lat == 0.0 and lon == 0.0:
            return
        self.track_points.append((lat, lon, alt, timestamp))

    def save_kml(self, filename=None):
        """
        Google Earth KML 파일로 저장
        KML은 XML 기반的地理 공간 데이터 포맷
        """
        if not self.track_points:
            print("저장할 트랙 포인트가 없습니다.")
            return None

        if filename is None:
            filename = f"gps_track_{self.start_time.strftime('%Y%m%d_%H%M%S')}.kml"

        filepath = os.path.join(self.output_dir, filename)

        # KML 헤더
        kml = []
        kml.append('<?xml version="1.0" encoding="UTF-8"?>')
        kml.append('<kml xmlns="http://www.opengis.net/kml/2.2">')
        kml.append('  <Document>')
        kml.append(f'    <name>Jetson Nano GPS Track</name>')
        kml.append(f'    <description>Recorded at {self.start_time.isoformat()}</description>')

        # 스타일
        kml.append('    <Style id="trackStyle">')
        kml.append('      <LineStyle>')
        kml.append('        <color>ff0000ff</color>')  # 빨간색
        kml.append('        <width>3</width>')
        kml.append('      </LineStyle>')
        kml.append('    </Style>')

        # 트랙 (경로)
        kml.append('    <Placemark>')
        kml.append('      <name>Track</name>')
        kml.append('      <styleUrl>#trackStyle</styleUrl>')
        kml.append('      <LineString>')
        kml.append('        <extrude>1</extrude>')
        kml.append('        <altitudeMode>absolute</altitudeMode>')
        kml.append('        <coordinates>')

        for lat, lon, alt, ts in self.track_points:
            kml.append(f'          {lon:.6f},{lat:.6f},{alt:.1f}')

        kml.append('        </coordinates>')
        kml.append('      </LineString>')
        kml.append('    </Placemark>')

        # 각 포인트에 Placemark (Waypoint)
        kml.append('    <Folder>')
        kml.append('      <name>Waypoints</name>')

        for i, (lat, lon, alt, ts) in enumerate(self.track_points):
            if i % 10 == 0:  # 10개마다 마커 표시 (너무 많으면 느려짐)
                kml.append('      <Placemark>')
                kml.append(f'        <name>Point {i}</name>')
                kml.append(f'        <description>Time: {ts}</description>')
                kml.append('        <Point>')
                kml.append(f'          <coordinates>{lon:.6f},{lat:.6f},{alt:.1f}</coordinates>')
                kml.append('        </Point>')
                kml.append('      </Placemark>')

        kml.append('    </Folder>')
        kml.append('  </Document>')
        kml.append('</kml>')

        # 파일 쓰기
        with open(filepath, 'w', encoding='utf-8') as f:
            f.write('\n'.join(kml))

        print(f"\nKML 파일 저장 완료: {filepath}")
        print(f"  총 포인트: {len(self.track_points)}개")
        print(f"  Google Earth에서 열어보세요!")
        return filepath

    def save_csv(self, filename=None):
        """CSV 파일로 저장 (Excel 호환)"""
        if not self.track_points:
            return None

        if filename is None:
            filename = f"gps_track_{self.start_time.strftime('%Y%m%d_%H%M%S')}.csv"

        filepath = os.path.join(self.output_dir, filename)

        with open(filepath, 'w', encoding='utf-8') as f:
            f.write("index,latitude,longitude,altitude_m,timestamp\n")
            for i, (lat, lon, alt, ts) in enumerate(self.track_points):
                f.write(f"{i},{lat:.6f},{lon:.6f},{alt:.1f},{ts}\n")

        print(f"CSV 파일 저장 완료: {filepath}")
        return filepath


def main():
    print("=" * 55)
    print("  Jetson Nano GPS Logger")
    print("  1초 간격 위치 기록 → KML/CSV 변환")
    print("=" * 55)

    # 출력 디렉토리
    output_dir = "gps_logs"
    os.makedirs(output_dir, exist_ok=True)

    # UART 설정
    UART_PORT = '/dev/ttyTHS1'
    GPS_BAUD = 9600

    try:
        ser = serial.Serial(UART_PORT, GPS_BAUD, timeout=0.5)
    except serial.SerialException as e:
        print(f"시리얼 포트 오류: {e}")
        return

    parser = NMEAParser()
    logger = GPSLogger(output_dir=output_dir)

    log_interval = 1.0  # 1초 기록 간격
    last_log_time = 0
    fix_acquired = False
    start_time = time.time()

    print("\nGPS 신호 대기 중... (Ctrl+C로 기록 종료 및 저장)")
    print("-" * 55)

    try:
        while True:
            raw = ser.readline().decode('ascii', errors='replace').strip()
            if not raw:
                continue

            if parser.parse(raw):
                now = time.time()

                # 첫 Fix 획득 메시지
                if parser.has_fix() and not fix_acquired:
                    elapsed = now - start_time
                    print(f"\n✅ GPS Fix 획득! (소요 시간: {elapsed:.0f}초)")
                    print(f"   위치: {parser.latitude:.4f}, {parser.longitude:.4f}")
                    print(f"   위성 수: {parser.satellites}")
                    fix_acquired = True

                # 기록 (1초 간격)
                if parser.has_fix() and (now - last_log_time) >= log_interval:
                    lat, lon, alt, speed, course = parser.get_position_tuple()
                    timestamp = datetime.now().strftime("%H:%M:%S")

                    logger.add_point(lat, lon, alt, timestamp)
                    last_log_time = now

                    elapsed = now - start_time
                    print(f"\r  [{int(elapsed):4d}s] "
                          f"위치: {lat:.6f}, {lon:.6f}  "
                          f"위성: {parser.satellites:2d}  "
                          f"고도: {alt:5.1f}m  "
                          f"'속도: {speed:4.1f}km/h  "
                          f"저장: {len(logger.track_points):4d}pt", end='', flush=True)

    except KeyboardInterrupt:
        print("\n\n기록 종료 중...")
    finally:
        if 'ser' in locals() and ser.is_open:
            ser.close()

    # 저장
    elapsed = time.time() - start_time
    print(f"\n총 기록 시간: {elapsed:.0f}초")
    print(f"수집된 포인트: {len(logger.track_points)}개")

    if logger.track_points:
        logger.save_kml()
        logger.save_csv()
    else:
        print("저장할 데이터가 없습니다.")

    print("\n완료!")


if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 gps_logger.py
```

**KML 파일 확인**: 생성된 `.kml` 파일을 Google Earth에서 열면 이동 경로가 지도에 표시됩니다.

---

## 5. 확인 및 테스트

### 검증 절차

```
1. UART 기본 통신
   → uart_basic.py 실행
   → PC 시리얼 모니터(115200 baud)에 "Hello from Jetson!" 1초 간격 표시 확인
   → PC에서 메시지 보내면 Jetson에서 수신 출력 확인

2. GPS 파싱
   → gps_nmea_parser.py 실행 (창가나 야외에서)
   → GPS Fix 획득 후 위도/경도/고도/속도 표시 확인
   → Google Maps URL이 생성되는지 확인

3. 블루투스 AT 명령
   → hc06_at.py 실행
   → "AT → OK" 응답 확인
   → 스마트폰에서 "JetsonBT" 검색 가능 확인

4. 최종 실습
   → gps_logger.py 실행 (야외에서 5분 이상)
   → Ctrl+C로 종료
   → gps_logs/ 디렉토리에 .kml, .csv 파일 생성 확인
   → KML 파일을 Google Earth에서 열어 트랙 확인
```

### 문제 해결

| 문제 | 원인 | 해결 방법 |
|------|------|----------|
| 시리얼 포트 없음 | UART 비활성화 | `ls -l /dev/ttyTHS1` 확인, JetPack 기본 활성화 |
| Permission denied | dialout 그룹 없음 | `sudo usermod -aG dialout $USER` 후 재부팅 |
| GPS Fix 안 됨 | 실내 신호 약함 | 창가 또는 야외로 이동 |
| GPS 문장만 출력됨 | Baud rate 불일치 | GPS 모듈 데이터시트 확인 (보통 9600) |
| HC-06 AT 응답 없음 | Baud rate 불일치 | HC-06 기본 9600, 일부 모델 115200 |
| HC-06 응답 깨짐 | 3.3V vs 5V 레벨 | 로직 레벨 컨버터 사용 |
| KML Google Earth 안 열림 | UTF-8 BOM 문제 | 파일을 UTF-8 without BOM으로 저장 확인 |
| GPS 속도 0 | 정지 상태 | 움직이면서 테스트 |
| HC-06 페어링 실패 | PIN mismatch | `AT+PIN`으로 PIN 확인 및 변경 |

---

> **다음 모듈**: Module 06 — 타이머/카운터/인터럽트 (HC-SR04 초음파, 로터리 엔코더)

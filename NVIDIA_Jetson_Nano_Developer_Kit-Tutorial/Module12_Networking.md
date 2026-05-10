# Module 12 — 네트워크 프로그래밍

> **소요 시간**: 3~4시간  
> **난이도**: ★★★☆☆  
> **준비물**: 이더넷/WiFi 연결, 스마트폰 또는 PC (테스트용)

---

## 1. 개요

Jetson Nano를 네트워크에 연결하여 원격 제어, 데이터 스트리밍, IoT 클라우드 연동을 실습합니다.

### 학습 목표
- TCP/UDP 소켓 프로그래밍을 이해한다
- Flask로 REST API 서버를 구축할 수 있다
- MQTT로 센서 데이터를 발행/구독할 수 있다
- WebSocket으로 실시간 대시보드를 만들 수 있다
- MJPEG로 실시간 영상을 스트리밍할 수 있다

---

## 2. 실습 코드

### 실습 12-1: Flask GPIO 웹 제어

**코드**: `flask_gpio_web.py`
```python
#!/usr/bin/env python3
"""
Flask 웹 GPIO 제어 — 브라우저에서 LED ON/OFF
"""

from flask import Flask, render_template_string, request
import Jetson.GPIO as GPIO

app = Flask(__name__)
LED_PIN = 11
GPIO.setmode(GPIO.BOARD)
GPIO.setup(LED_PIN, GPIO.OUT)
GPIO.output(LED_PIN, GPIO.LOW)

HTML = """
<!DOCTYPE html>
<html>
<head><title>Jetson GPIO Control</title></head>
<body>
  <h1>Jetson Nano GPIO Control</h1>
  <p>LED Status: <strong>{{ status }}</strong></p>
  <a href="/on"><button>LED ON</button></a>
  <a href="/off"><button>LED OFF</button></a>
</body>
</html>
"""

@app.route('/')
def index():
    status = "ON" if GPIO.input(LED_PIN) else "OFF"
    return render_template_string(HTML, status=status)

@app.route('/on')
def led_on():
    GPIO.output(LED_PIN, GPIO.HIGH)
    return index()

@app.route('/off')
def led_off():
    GPIO.output(LED_PIN, GPIO.LOW)
    return index()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

**실행**: `sudo python3 flask_gpio_web.py` → 브라우저에서 `http://jetson-ip:5000` 접속

### 실습 12-2: MQTT 센서 데이터 발행

**코드**: `mqtt_sensor.py`
```python
#!/usr/bin/env python3
"""
MQTT로 센서 데이터 발행
"""

import paho.mqtt.client as mqtt
import time
import json
from smbus2 import SMBus

# MQTT 브로커 (Mosquitto 필요)
BROKER = "localhost"
TOPIC = "jetson/sensor/temperature"

class AHT20:
    def __init__(self):
        self.bus = SMBus(1)
        self.addr = 0x38
        self.bus.write_byte(self.addr, 0xBE)
        time.sleep(0.01)

    def read(self):
        self.bus.write_i2c_block_data(self.addr, 0xAC, [0x33, 0x00])
        time.sleep(0.08)
        data = self.bus.read_i2c_block_data(self.addr, 0x00, 6)
        raw_temp = ((data[3] & 0x0F) << 16) | (data[4] << 8) | data[5]
        temp = (raw_temp / 0x100000) * 200.0 - 50.0
        raw_hum = ((data[1] << 16) | (data[2] << 8) | data[3]) >> 4
        hum = (raw_hum / 0x100000) * 100.0
        return temp, hum

sensor = AHT20()
client = mqtt.Client()
client.connect(BROKER)

while True:
    temp, hum = sensor.read()
    payload = json.dumps({"temperature": temp, "humidity": hum})
    client.publish(TOPIC, payload)
    print(f"Published: {payload}")
    time.sleep(5)
```

---

> **다음 모듈**: Module 13 — USB & 블루투스

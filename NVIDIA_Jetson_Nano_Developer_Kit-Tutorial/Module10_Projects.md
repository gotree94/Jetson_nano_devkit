# Module 10 — 종합 프로젝트

> **소요 시간**: 6~10시간 (프로젝트당 3~5시간)  
> **난이도**: ★★★★☆  
> **준비물**: 지금까지의 모든 모듈에서 사용한 하드웨어

---

## 1. 개요

Module 1~9에서 배운 모든 기술을 통합하여 실제 동작하는 시스템을 만듭니다.  
5개의 프로젝트 중 선택하거나 변형하여 진행합니다.

### 공통 학습 목표
- 복수의 하드웨어 모듈을 통합하는 방법을 이해한다
- 실시간 시스템에서의 동기화/타이밍 이슈를 해결한다
- 종단 간(end-to-end) 시스템을 설계하고 디버깅한다

---

## 프로젝트 A — 스마트 CCTV

**통합 모듈**: Module 7(카메라), 8(컴퓨터 비전), 9(AI), 12(네트워크)

### 기능
1. CSI 카메라로 실시간 영상 캡처
2. YOLO 객체 탐지로 사람/차량 감지
3. 움직임 감지 시 Telegram/MQTT 알림 전송
4. 웹 인터페이스로 실시간 스트리밍

### 코드 구조

```
smart_cctv/
├── main.py                  # 메인 루프
├── detector.py              # YOLO/TensorRT 객체 탐지
├── motion.py                # 움직임 감지 (Module 7)
├── notifier.py              # Telegram/MQTT 알림
├── streamer.py              # Flask MJPEG 스트리머
└── config.py                # 설정 파일
```

**코드**: `main.py` (개요)
```python
#!/usr/bin/env python3
"""
스마트 CCTV — 종합 프로젝트
"""

import cv2
import threading
import time
from detector import ObjectDetector
from motion import MotionDetector
from notifier import TelegramNotifier
from streamer import start_streamer

CONFIG = {
    "motion_threshold": 25,
    "detect_interval": 2.0,  # 객체 탐지 간격 (초)
    "notify_cooldown": 30.0,  # 알림 쿨다운 (초)
    "stream_port": 5000,
    "telegram_token": "YOUR_BOT_TOKEN",
    "telegram_chat_id": "YOUR_CHAT_ID",
}

def main():
    cap = cv2.VideoCapture(CSI_PIPELINE, cv2.CAP_GSTREAMER)

    detector = ObjectDetector(model="ssd-mobilenet-v2")
    motion = MotionDetector(threshold=CONFIG["motion_threshold"])
    notifier = TelegramNotifier(CONFIG["telegram_token"], CONFIG["telegram_chat_id"])

    # 웹 스트리머 시작 (별도 스레드)
    stream_thread = threading.Thread(
        target=start_streamer, args=(cap, CONFIG["stream_port"]), daemon=True
    )
    stream_thread.start()

    last_detect_time = 0
    last_notify_time = 0

    while True:
        ret, frame = cap.read()
        if not ret:
            break

        # 1. 움직임 감지
        has_motion, _ = motion.detect(frame)

        # 2. 주기적 객체 탐지
        now = time.time()
        if has_motion and now - last_detect_time > CONFIG["detect_interval"]:
            detections = detector.detect(frame)
            last_detect_time = now

            # 3. 사람 감지 시 알림
            persons = [d for d in detections if d["class"] == "person"]
            if persons and now - last_notify_time > CONFIG["notify_cooldown"]:
                notifier.send_alert(frame, persons)
                last_notify_time = now

        # 화면 표시
        cv2.imshow("Smart CCTV", frame)
        if cv2.waitKey(1) & 0xFF == 27:
            break

    cap.release()
    cv2.destroyAllWindows()

if __name__ == "__main__":
    main()
```

---

## 프로젝트 B — 라인 트레이서 RC카

**통합 모듈**: Module 1(GPIO), 2(PWM), 4(SPI), 7(카메라)

### 기능
1. 카메라로 차선(검은색 선) 인식
2. 차선 중심에서 벗어난 정도 계산
3. 모터 드라이버(L298N)로 DC 모터 속도/방향 제어
4. PID 제어로 차선 추종

### 코드 개요
```python
def get_line_position(frame):
    """카메라 영상에서 차선 위치 검출"""
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    _, binary = cv2.threshold(gray, 127, 255, cv2.THRESH_BINARY_INV)
    h, w = binary.shape
    roi = binary[h//2:, :]  # 하단 절반만 분석
    moments = cv2.moments(roi)
    if moments["m00"] > 0:
        cx = int(moments["m10"] / moments["m00"])
        return cx - w//2  # 중앙으로부터 편차
    return None

class PIDController:
    def __init__(self, kp=0.5, ki=0.01, kd=0.1):
        self.kp, self.ki, self.kd = kp, ki, kd
        self.prev_error = 0
        self.integral = 0

    def compute(self, error, dt):
        self.integral += error * dt
        derivative = (error - self.prev_error) / dt
        self.prev_error = error
        return self.kp * error + self.ki * self.integral + self.kd * derivative
```

---

## 프로젝트 C — 제스처 제어 로봇팔

**통합 모듈**: Module 2(PWM), 7(카메라), 8(Computer Vision), 9(AI)

### 기능
1. MediaPipe 또는 CNN으로 손동작 인식
2. 손가락 관절 각도를 서보 모터 각도로 매핑
3. 5-DOF 로봇팔 제어 (SG90 서보 5개)
4. PCA9685로 서보 동시 제어

---

## 프로젝트 D — IoT 환경 모니터링

**통합 모듈**: Module 3(I2C), 5(UART), 6(Timer), 12(네트워크)

### 기능
1. AHT20 온습도 + MPU6050 자세 센서 데이터 수집
2. MQTT로 클라우드 전송 (AWS IoT / ThingsBoard)
3. OLED에 실시간 표시
4. WebSocket 대시보드

---

## 프로젝트 E — 자율 주행 시뮬레이터 (JetRacer)

**통합 모듈**: Module 7(카메라), 8(CV), 9(AI), 2(PWM)

### 설명
NVIDIA JetRacer 또는 DonkeyCar 프레임워크 기반으로,  
카메라 영상을 입력으로 받아 차선 유지 + 장애물 회피를 수행합니다.

```bash
# DonkeyCar 설치
git clone https://github.com/autorope/donkeycar
cd donkeycar
pip install -e .[nano]

# JetRacer 설치
git clone https://github.com/NVIDIA-AI-IOT/jetracer
cd jetracer
python3 setup.py install
```

---

## 2. 프로젝트 진행 가이드

### 공통 체크리스트

- [ ] 요구사항 분석 및 기능 명세 작성
- [ ] 하드웨어 회로도 작성
- [ ] 모듈별 단위 테스트
- [ ] 통합 테스트
- [ ] 안정성 테스트 (24시간 연속)
- [ ] 문서화

### 평가 기준

| 항목 | 배점 |
|------|------|
| 하드웨어 연결 정확성 | 20% |
| 소프트웨어 설계 및 코드 품질 | 30% |
| 필수 기능 동작 | 30% |
| 추가 기능 및 창의성 | 10% |
| 문서 및 발표 | 10% |

---

> **다음 모듈**: Module 11 — I2S 디지털 오디오

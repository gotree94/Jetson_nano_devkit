# NVIDIA Jetson Nano Developer Kit — 종합 교육 커리큘럼

> **대상**: 임베디드 시스템/Linux 기초 지식이 있는 개발자  
> **목표**: Jetson Nano 하드웨어 제어부터 AI 추론까지 전 과정 습득  
> **총 구성**: 16개 모듈, 50+ 실습

---

[목차]
   * 모듈 0 — 개발 환경 구축 및 시스템 초기화
   * 모듈 1 — GPIO 디지털 입출력
   * 모듈 2 — PWM (Pulse Width Modulation)
   * 모듈 3 — I2C (Inter-Integrated Circuit)
   * 모듈 4 — SPI (Serial Peripheral Interface)
   * 모듈 5 — UART (시리얼 통신)
   * 모듈 6 — 타이머 / 카운터 / 인터럽트
   * 모듈 7 — 카메라 및 영상 처리 기초
   * 모듈 8 — 영상 처리 심화 및 컴퓨터 비전
   * 모듈 9 — NVIDIA Jetson AI / 딥러닝
   * 모듈 10 — 종합 프로젝트
   * 모듈 11 — I2S 디지털 오디오
   * 모듈 12 — 네트워크 프로그래밍
   * 모듈 13 — USB & 블루투스
   * 모듈 14 — CAN 통신
   * 모듈 15 — CUDA 기초
   * 모듈 16 — 시스템 프로그래밍

---

## 모듈 0 — 개발 환경 구축 및 시스템 초기화

| # | 주제 | 실습 내용 |
|---|------|----------|
| 0.1 | JetPack SDK 설치 | SD 카드 이미지 플래싱, 부트 확인 |
| 0.2 | Headless 초기 설정 | 시리얼 콘솔 / SSH 접속, WiFi 설정 |
| 0.3 | 시스템 확인 | ` jetson_clocks`, `tegrastats`, `nvidia-smi` |
| 0.4 | 개발 도구 설치 | Python3, pip, `Jetson.GPIO`, `wiringPi`, `libi2c-dev` |
| 0.5 | 크로스 컴파일 환경 | 호스트 PC에서 aarch64 크로스 컴파일 |

**실습**: `tegrastats`로 CPU/GPU/온도 모니터링, 파워 모드 전환

---

## 모듈 1 — GPIO 디지털 입출력

| # | 주제 | 실습 내용 |
|---|------|----------|
| 1.1 | GPIO 핀맵 이해 | 40-pin 헤더, 3.3V/5V/1.8V 레벨, 핀 번호 vs GPIO 번호 |
| 1.2 | `Jetson.GPIO` 라이브러리 | 라이브러리 초기화, 핀 모드 설정 (input/output) |
| 1.3 | 디지털 출력 | `output(pin, HIGH/LOW)` — LED 점멸 |
| 1.4 | 디지털 입력 | `input(pin)` — 푸시버튼 읽기, 풀업/풀다운 저항 |
| 1.5 | 인터럽트 (이벤트 감지) | `add_event_detect()` — 버튼 눌림 감지, debounce |
| 1.6 | 멀티스레드 GPIO | `threading` + GPIO 폴링/이벤트 |

**실습 1-A**: LED 2개 + 버튼 1개 — 버튼 누르면 LED 토글  
**실습 1-B**: 7-Segment LED (7447 디코더 or 직접 제어) — 숫자 표시 카운터

---

## 모듈 2 — PWM (Pulse Width Modulation)

| # | 주제 | 실습 내용 |
|---|------|----------|
| 2.1 | PWM 원리 | 듀티 사이클, 주파수, 분해능 |
| 2.2 | Jetson Nano PWM | 사용 가능한 PWM 핀 (PCA9685 or 소프트웨어 PWM) |
| 2.3 | 하드웨어 PWM | `sysfs` PWM 인터페이스 `/sys/class/pwm/` |
| 2.4 | 소프트웨어 PWM | `RPi.GPIO` 호환 방식, 타이머 기반 PWM |
| 2.5 | 서보 모터 제어 | SG90 — 50Hz, 1ms~2ms 펄스, 각도 제어 |
| 2.6 | DC 모터 속도 제어 | L298N 모터 드라이버 + PWM으로 속도 조절 |

**실습 2-A**: RGB LED — PWM으로 색상 혼합 (R/G/B 각각 PWM)  
**실습 2-B**: 서보 모터 — 0°~180° 스윕 + 포텐셔미터로 수동 제어

---

## 모듈 3 — I2C (Inter-Integrated Circuit)

| # | 주제 | 실습 내용 |
|---|------|----------|
| 3.1 | I2C 프로토콜 | SDA/SCL, 7비트 주소, Start/Stop/Ack 조건 |
| 3.2 | Jetson Nano I2C 버스 | `i2cdetect -y -r 1` 으로 디바이스 스캔 |
| 3.3 | `smbus2` / `python-periphery` | I2C read/write, 레지스터 접근 |
| 3.4 | 온도/습도 센서 | AHT20 / SHT30 — I2C로 데이터 시트 따라 읽기 |
| 3.5 | IMU 센서 | MPU6050 — 가속도/자이로 읽기, 오일러 각 계산 |
| 3.6 | OLED 디스플레이 | SSD1306 128x64 — I2C로 텍스트/도형 그리기 |
| 3.7 | I2C 멀티플렉싱 | TCA9548A로 여러 동일 주소 디바이스 사용 |

**실습 3-A**: 환경 모니터 스테이션 — AHT20(온습도) + SSD1306(OLED 표시)  
**실습 3-B**: MPU6050姿态(자세) 추정 — 칼만 필터 기초 적용

---

## 모듈 4 — SPI (Serial Peripheral Interface)

| # | 주제 | 실습 내용 |
|---|------|----------|
| 4.1 | SPI 프로토콜 | MOSI/MISO/SCLK/CS, Mode 0~3, CPOL/CPHA |
| 4.2 | Jetson Nano SPI | SPI 버스 활성화 (`config.txt` or device tree) |
| 4.3 | `spidev` 라이브러리 | `spi-xfer()`, full-duplex 통신 |
| 4.4 | MCP3008 ADC | SPI로 아날로그 입력 읽기 (조도 센서, 가변 저항) |
| 4.5 | MAX7219 LED Matrix | SPI로 8x8 LED 매트릭스 제어 |
| 4.6 | SD 카드 / RFID | SPI 모드 RFID 리더 (RC522) 연동 |

**실습 4-A**: CDS 조도 센서(MCP3008) → 밝기에 따라 LED PWM 밝기 제어  
**실습 4-B**: MAX7219 8x8 매트릭스에 스크롤 텍스트 표시

---

## 모듈 5 — UART (시리얼 통신)

| # | 주제 | 실습 내용 |
|---|------|----------|
| 5.1 | UART 프로토콜 | TX/RX, Baud rate, Start/Stop bit, Parity |
| 5.2 | Jetson Nano UART | `/dev/ttyTHS1`, `/dev/ttyS0`, UART 핀 매핑 |
| 5.3 | `pyserial` | `serial.Serial()`, read/write, flow control |
| 5.4 | GPS 수신기 | NMEA-0183 파싱 (GGA/RMC 문장 → 위도/경도) |
| 5.5 | 블루투스 모듈 | HC-06 AT 커맨드 설정, UART-블루투스 브릿지 |
| 5.6 | 디버그 콘솔 | UART를 통한 커널 디버그 로그 출력 |

**실습 5-A**: USB-UART 변환기로 PC ↔ Jetson Nano 양방향 통신  
**실습 5-B**: GPS 로거 — 1초 간격 위치 기록 → KML 파일로 변환

---

## 모듈 6 — 타이머 / 카운터 / 인터럽트

| # | 주제 | 실습 내용 |
|---|------|----------|
| 6.1 | Linux 타이머 | `timerfd`, `clock_gettime()`, `nanosleep()` |
| 6.2 | 주기 태스크 | `setitimer`, `signal(SIGALRM)` / `threading.Timer` |
| 6.3 | 하드웨어 타이머 | Jetson Nano PWM Timer, watchdog 타이머 |
| 6.4 | GPIO 인터럽트 심화 | 엣지 트리거(rising/falling/both), 인터럽트 우선순위 |
| 6.5 | 초음파 거리 측정 | HC-SR04 — TRIG/PIN 인터럽트로 펄스 폭 측정 |
| 6.6 | 로터리 엔코더 | KY-040 — A/B상 인터럽트로 회전 방향/속도 계산 |

**실습 6-A**: HC-SR04 초음파 + OLED — 거리 측정 디스플레이  
**실습 6-B**: 로터리 엔코더 + 서보 모터 — 다이얼 돌리면 서보 각도 변경

---

## 모듈 7 — 카메라 및 영상 처리 기초

| # | 주제 | 실습 내용 |
|---|------|----------|
| 7.1 | CSI 카메라 인터페이스 | `nvgstcapture-1.0`, `v4l2-ctl` |
| 7.2 | GStreamer 파이프라인 | `nvarguscamerasrc` → GPU 가속 비디오 파이프라인 |
| 7.3 | OpenCV 설치 및 빌드 | `libopencv-dev`, CUDA 가속 빌드 옵션 |
| 7.4 | OpenCV 기본 | `cv2.imread`, 카메라 캡처, 프레임 처리 |
| 7.5 | 실시간 영상 처리 | 그레이스케일, Canny Edge, Blur, Threshold |
| 7.6 | GPU 가속 OpenCV | `cv2.cuda` 모듈, CUDA Stream 처리 |
| 7.7 | USB 카메라 | OpenCV `VideoCapture(0)` vs CSI 카메라 성능 비교 |

**실습 7-A**: 실시간 Canny Edge 검출 — CSI 카메라 → GPU 처리 → 화면 출력  
**실습 7-B**: 움직임 감지 카메라 — 프레임 차분(FlameDiff) → 모션 이벤트 발생 + 로깅

---

## 모듈 8 — 영상 처리 심화 및 컴퓨터 비전

| # | 주제 | 실습 내용 |
|---|------|----------|
| 8.1 | 색상 기반 객체 추적 | HSV 색공간 변환, inRange 마스킹, 컨투어 검출 |
| 8.2 | 특징점 검출 | SIFT, ORB, FAST — 특징점 매칭 |
| 8.3 | Optical Flow | Lucas-Kanade, Farneback — 움직임 벡터 |
| 8.4 | ArUco 마커 | 마커 생성, 포즈 추정 (PnP) |
| 8.5 | 스테레오 비전 | OpenCV StereoBM / SGBM → 깊이 맵 생성 |
| 8.6 | 실시간 객체 감지 (HOG) | HOG + SVM 사람 감지 (OpenCV 내장) |
| 8.7 | Camera Calibration | 체커보드 캘리브레이션, 왜곡 보정 |

**실습 8-A**: 색상 기반 볼 추적 — 빨간 공을 따라 서보 카메라 마운트 Pan/Tilt  
**실습 8-B**: ArUco 마커 기반 증강현실 — 마커 위에 3D 큐브 오버레이

---

## 모듈 9 — NVIDIA Jetson AI / 딥러닝

| # | 주제 | 실습 내용 |
|---|------|----------|
| 9.1 | NVIDIA TAO Toolkit / Transfer Learning Toolkit | 모델 Fine-tuning 개념 |
| 9.2 | TensorRT 소개 | FP16/INT8 양자화, ONNX → TensorRT 엔진 변환 |
| 9.3 | `jetson-inference` 라이브러리 | `imageNet()`, `detectNet()`, `segNet()` |
| 9.4 | 분류 (Classification) | ImageNet 사전학습 모델로 실시간 이미지 분류 |
| 9.5 | 객체 탐지 (Object Detection) | SSD-Mobilenet / YOLO — 실시간 탐지 + 바운딩 박스 |
| 9.6 | 의미적 분할 (Semantic Segmentation) | FCN-ResNet / UNet — 픽셀 단위 분할 |
| 9.7 | Face Recognize | `faceNet` / `ArcFace` — 실시간 얼굴 인식 시스템 |
| 9.8 | Pose Estimation | `openpose` / `trt_pose` — 키포인트 검출 |
| 9.9 | Custom Model 배포 | PyTorch → ONNX → TensorRT 엔진 변환 파이프라인 |
| 9.10 | 멀티 모델 파이프라인 | 객체 탐지 → 크롭 → 분류 (2-stage inference) |
| 9.11 | DeepStream SDK | 멀티 카메라 스트리밍, 메타데이터, 분석 파이프라인 |

**실습 9-A**: 실시간 YOLO 객체 탐지 — CSI 카메라 입력 → TensorRT 추론 → 바운딩 박스 출력  
**실습 9-B**: 얼굴 인식 출입 시스템 — 얼굴 탐지 → 특징 추출 → DB 매칭 → OLED/릴레이 출력  
**실습 9-C**: Custom Model 배포 — PyTorch로 학습한 모델 → ONNX → TensorRT → Jetson Nano에서 추론

---

## 모듈 10 — 종합 프로젝트

| 프로젝트 | 활용 모듈 | 설명 |
|---------|----------|------|
| 스마트 CCTV | 7, 8, 9 | 움직임 감지 + 객체 탐지 + Telegram 알림 |
| 라인 트레이서 RC카 | 1, 2, 4, 7 | 카메라 영상 → 차선 인식 → 모터 PWM 제어 |
| 제스처 제어 로봇팔 | 4, 7, 9 | 손동작 인식 → 서보 모터 제어 (5-DOF) |
| IoT 환경 모니터링 | 3, 5, 6 | 온습도 + GPS + I2C 센서 → 클라우드 전송 |
| 자율 주행 시뮬레이터 | 4, 7, 8, 9 | DonkeyCar / JetRacer — 차선 유지 + 장애물 회피 |

---

## 모듈 11 — I2S 디지털 오디오

| # | 주제 | 실습 내용 |
|---|------|----------|
| 11.1 | I2S 프로토콜 | SCLK / LRCK / SDIN / SDOUT, Frame Sync, Word Select |
| 11.2 | Jetson Nano I2S 핀 | pin 12(SCLK), 35(LRCK), 38(SDIN), 40(SDOUT), 7(MCLK) |
| 11.3 | I2S MEMS 마이크 | INMP441 — PDM/PCM 데이터 캡처, 음성 녹음 |
| 11.4 | I2S 앰프 출력 | MAX98357 — WAV 파일 재생, PCM 스트리밍 |
| 11.5 | FFT 주파수 분석 | `numpy.fft` / `scipy.signal` — 실시간 스펙트럼 |
| 11.6 | Audio Loopback | I2S 입력 → Python 처리 → I2S 출력 실시간 파이프라인 |

**실습 11-A**: INMP441 음성 녹음기 — 버튼 누르면 녹음 → WAV 파일 저장  
**실습 11-B**: 실시간 스펙트럼 분석기 — FFT → OLED / MAX7219 가시화  
**실습 11-C**: 음성 명령 감지 — 특정 주파수 대역 검출 → GPIO 액추에이터 트리거

---

## 모듈 12 — 네트워크 프로그래밍

| # | 주제 | 실습 내용 |
|---|------|----------|
| 12.1 | 소켓 프로그래밍 기초 | TCP Server/Client, UDP broadcast |
| 12.2 | Flask 웹 서버 | REST API로 GPIO/LED 원격 제어 |
| 12.3 | MQTT 통신 | Mosquitto 브로커 — 센서 데이터 publish/subscribe |
| 12.4 | WebSocket 실시간 대시보드 | `flask-socketio` — GPIO 상태 모니터링 웹 UI |
| 12.5 | HTTP 스트리밍 | Flask + OpenCV — MJPEG 실시간 영상 스트리밍 |
| 12.6 | IoT 클라우드 연동 | AWS IoT Core / ThingsBoard — 텔레메트리 업로드 |

**실습 12-A**: Flask 웹 GPIO 제어판 — 브라우저에서 LED ON/OFF + PWM 슬라이더  
**실습 12-B**: MQTT 센서 모니터 — 온습도(I2C) → MQTT publish → Node-RED 대시보드  
**실습 12-C**: 실시간 영상 스트리밍 — CSI 카메라 → Flask MJPEG → 모바일 브라우저 시청

---

## 모듈 13 — USB & 블루투스

| # | 주제 | 실습 내용 |
|---|------|----------|
| 13.1 | USB Gadget 모드 | ConfigFS — USB CDC ACM / Ethernet Gadget |
| 13.2 | USB 디바이스 모드 | Jetson Nano를 키보드/마우스/시리얼 장치로 에뮬레이션 |
| 13.3 | USB Host 프로그래밍 | `libusb` / `pyusb` — USB HID, USB 시리얼 |
| 13.4 | Bluetooth Classic | `bluetooth` / `pybluez` — RFCOMM SPP 프로파일 |
| 13.5 | BLE (Bluetooth Low Energy) | `bluepy` / `bleak` — BLE 스캔, advertisement, GATT |
| 13.6 | 조이스틱 / 게임패드 | `pygame.joystick` / `evdev` — USB 조이스틱 입력 처리 |

**실습 13-A**: USB Gadget — Jetson Nano를 USB 시리얼 장치로 → PC에서 `/dev/ttyACM0` 접속  
**실습 13-B**: BLE 환경 센서 — I2C 온습도 → BLE Advertisement → 스마트폰 앱 수신  
**실습 13-C**: USB 조이스틱 → RC카 제어 — 조이스틱 축/버튼 → 모터 PWM + 서보

---

## 모듈 14 — CAN 통신

| # | 주제 | 실습 내용 |
|---|------|----------|
| 14.1 | CAN 프로토콜 | CAN 2.0A/B, Data Frame / Remote Frame, Arbitration |
| 14.2 | CAN 하드웨어 | MCP2515 (SPI CAN 컨트롤러) + SN65HVD230 / TJA1050 트랜시버 |
| 14.3 | Linux CAN 프레임워크 | `SocketCAN` — `can-utils`, `candump`, `cansend` |
| 14.4 | CAN 메시지 송수신 | `python-can` —周期性 전송, ID 필터링 |
| 14.5 | CAN FD | Flexible Data-rate, 최대 64바이트 |
| 14.6 | OBD-II 차량 진단 | ELM327 / MCP2515 — 차량 PID 요청, RPM/속도/냉각수 온도 |

**실습 14-A**: 두 대의 Jetson Nano CAN 통신 — MCP2515 + CAN 트랜시버로 메시지 교환  
**실습 14-B**: OBD-II 차량 데이터 리더 — RPM + 차속 + 엔진 온도 실시간 모니터링  
**실습 14-C**: CAN → MQTT 브릿지 — CAN 버스 데이터 → MQTT → IoT 대시보드

---

## 모듈 15 — CUDA 기초

| # | 주제 | 실습 내용 |
|---|------|----------|
| 15.1 | GPU 아키텍처 | Maxwell (Jetson Nano), SM, Warp, Thread Block |
| 15.2 | CUDA 개발 환경 | `nvcc`, `cuda-gdb`, `nsys` profiler |
| 15.3 | Kernel 작성 기초 | `__global__`, `threadIdx`/`blockIdx`, Grid/Block 설정 |
| 15.4 | 벡터 연산 | Vector Add — CPU vs GPU 성능 비교 |
| 15.5 | 행렬 곱셈 | Shared Memory / Tiling 최적화 |
| 15.6 | CUDA Events & Streams | 비동기 커널 실행, Overlap transfer & compute |
| 15.7 | OpenCV + CUDA Interop | `cv::cuda::GpuMat` → custom CUDA kernel |
| 15.8 | CUDA 성능 프로파일링 | `nvprof`/`nsys` — occupancy, memory bandwidth |

**실습 15-A**: 벡터 덧셈 — CPU (numpy) vs GPU (CUDA) 속도 비교 + 그래프  
**실습 15-B**: Sobel Edge Detection CUDA 구현 — OpenCV 결과와 정확도 비교  
**실습 15-C**: CUDA + TensorRT — custom CUDA plugin으로 전처리 커널 작성 → 추론 파이프라인

---

## 모듈 16 — 시스템 프로그래밍

| # | 주제 | 실습 내용 |
|---|------|----------|
| 16.1 | Watchdog Timer | `/dev/watchdog` — 시스템 감시, 강제 리부트 실습 |
| 16.2 | RTC (Real-Time Clock) | DS3231 (I2C) — `hwclock`, kernel RTC 연동 |
| 16.3 | Linux Device Tree | DTS 구조, pinmux overlay, 커스텀 DTB 빌드 |
| 16.4 | 저수준 GPIO (mmap) | `/dev/mem` / `/dev/gpiochip` — `libgpiod` / `gpiocli` |
| 16.5 | Linux DMA | `dma-buf`, DMA 영역 메모리 매핑 |
| 16.6 | Power Management | `/sys/devices/system/cpu/cpu0/cpufreq` — DVFS, 클럭 제어 |
| 16.7 | 부트 체인 이해 | CBoot → U-Boot → Kernel → initramfs → rootfs |

**실습 16-A**: Watchdog 데몬 작성 — 메인 프로세스 Hang 감지 → WDT 리부트  
**실습 16-B**: RTC 알람 시계 — DS3231 알람 설정 → GPIO 인터럽트 → LED 점멸  
**실습 16-C**: 커스텀 Device Tree Overlay — 새로운 SPI 칩 select 핀 할당 → 빌드 → 부팅 테스트

---

## 부록 A — NVIDIA 고유 API (Jetson Nano 전용)

| API | 설명 | 교육 난이도 | 활용 실습 |
|-----|------|-----------|----------|
| **CUDA** | GPU 병렬 연산 | ★★☆ | 행렬 곱, 이미지 필터, 커널 최적화 |
| **VPI (Vision Programming Interface)** | GPU/CPU/PVA 가속 영상 처리 | ★★★ | 피라미드 OptFlow, Stereo Disparity |
| **NvMedia** | 하드웨어 비디오 인코더/디코더 (H.264/H.265) | ★★★ | 실시간 인코딩, 트랜스코딩 |
| **Multimedia API (gst-nv)** | GStreamer 하드웨어 가속 파이프라인 | ★★☆ | `nvarguscamerasrc` → `nvvidconv` → `nvegltransform` |
| **libargus** | CSI 카메라 하드웨어 제어 API | ★★☆ | 노출/셔터/ISO 직접 제어, RAW 캡처 |
| **TensorRT** | 추론 최적화 엔진 (FP16/INT8) | ★★☆ | ONNX → TRT 엔진 변환, 동적 배치 |
| **DeepStream SDK** | 멀티 카메라 AI 분석 파이프라인 | ★★★ | 8채널 동시 객체 탐지, 메타데이터 |

---

## 부록 B — 핀맵 참조 (Jetson Nano 40-pin Header)

```
Jetson Nano 40-Pin Header (리비전 A02)
+-----+-----+---------+--------+               +--------+---------+-----+-----+
| SDA | 3V3 |   I2S   |  GND   |               |  GND   |  I2S    | 3V3 | SCL |
|  3  |  2  |   27     |  39    |               |  40    |  38     |  1  |  5  |
+-----+-----+---------+--------+               +--------+---------+-----+-----+
...
```

(전체 핀맵은 공식 Jetson Nano GPIO Pinout 문서 참조)

---

## 부록 C — 헤더별 지원 인터페이스 요약

| 인터페이스 | 40-pin 헤더 지원 | 보드 레벨 | 비고 |
|-----------|-----------------|-----------|------|
| GPIO | ✅ 12+ 핀 | — | 3.3V 로벨, TXB0108 레벨 시프터 |
| I2C | ✅ 2개 버스 | — | 핀 3/5 (I2C_1), 27/28 (I2C_0) |
| SPI | ✅ 2개 버스 | — | SPI_0 (칩셀렉트 2개), SPI_1 |
| UART | ✅ 1개 | 추가 1개 (M.2) | `/dev/ttyTHS1`, 디버그 UART |
| I2S | ✅ 1개 | M.2 Key E | SCLK/LRCK/SDIN/SDOUT |
| PWM | △ pinmux 필요 | — | 기본 GPIO → PWM 재할당 |
| ADC | ❌ 없음 | — | MCP3008(SPI ADC) 필수 |
| CAN | ❌ 미할당 | — | MCP2515(SPI) + 트랜시버 필수 |
| Ethernet | — | ✅ Gigabit | RJ45, IEEE 802.3af PoE 지원 |
| USB 3.0 | — | ✅ 4포트 | USB Hub 내장 |
| HDMI 2.0 | — | ✅ 1포트 | 4K@60Hz 출력 |
| CSI-2 | — | ✅ 12-lane | IMX219, Raspberry Pi Camera V2 |
| M.2 Key E | — | ✅ PCIe x1 + UART + I2S + I2C | WiFi/BT 카드 장착 |

---

## 필수 준비물 목록

| 품목 | 용도 | 관련 모듈 |
|------|------|----------|
| NVIDIA Jetson Nano (4GB 권장) | 메인 보드 | 전 모듈 |
| 32GB+ microSD (UHS-I U3) | OS 및 데이터 저장 | 0 |
| 5V⎓4A DC 어댑터 (Micro-USB or Barrel Jack) | 전원 공급 | 0 |
| CSI 카메라 (IMX219) | 영상 처리 모듈 | 7, 8, 9 |
| 브레드보드 + 점퍼 케이블 (M-M, M-F) | 회로 구성 | 1~6 |
| LED 5mm x 5색, 저항 330Ω | GPIO 출력 실습 | 1, 2 |
| 푸시버튼 x 5, 10kΩ 풀업 저항 | GPIO 입력 실습 | 1, 6 |
| HC-SR04 초음파 센서 | 타이머/인터럽트 실습 | 6 |
| SG90 서보 모터 | PWM 실습 | 2, 8 |
| MPU6050 IMU 센서 | I2C 실습 | 3 |
| SSD1306 OLED 128x64 (I2C) | I2C + 디스플레이 실습 | 3, 6, 11 |
| MCP3008 ADC + CDS 조도 센서 | SPI 실습 | 4 |
| USB-UART 변환기 (CP2102/CH340) | UART 실습 | 5 |
| INMP441 I2S MEMS 마이크 | I2S 오디오 입력 실습 | 11 |
| MAX98357 I2S 앰프 + 3W 스피커 | I2S 오디오 출력 실습 | 11 |
| MCP2515 CAN 모듈 + SN65HVD230 | CAN 통신 실습 | 14 |
| DS3231 RTC 모듈 | 시스템 프로그래밍 실습 | 16 |
| USB 조이스틱 / 게임패드 | USB 입력 실습 | 13 |
| HC-06 블루투스 모듈 | UART-블루투스 실습 | 5, 13 |

---

## 참고 자료

| 자료 | 링크 |
|------|------|
| NVIDIA Jetson Nano Developer Kit User Guide | https://developer.nvidia.com/embedded/dlc/jetson-nano-dev-kit-user-guide |
| Jetson GPIO Library (python-periphery) | https://github.com/NVIDIA/jetson-gpio |
| NVIDIA Jetson Inference | https://github.com/dusty-nv/jetson-inference |
| TensorRT Documentation | https://docs.nvidia.com/deeplearning/tensorrt/ |
| VPI (Vision Programming Interface) | https://developer.nvidia.com/embedded/vpi |
| NVIDIA L4T Development Guide | https://docs.nvidia.com/jetson/archives/ |
| Jetson Nano Pinmux Spreadsheet | https://developer.nvidia.com/embedded/dlc/jetson-nano-40-pin-expansion-header-1.2 |
| CUDA Toolkit Documentation | https://docs.nvidia.com/cuda/ |
| DeepStream SDK | https://developer.nvidia.com/deepstream-sdk |
| JetPack SDK | https://developer.nvidia.com/embedded/jetpack |
| JetsonHacks (커뮤니티 자료) | https://jetsonhacks.com |

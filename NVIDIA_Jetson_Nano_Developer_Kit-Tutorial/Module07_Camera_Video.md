# Module 07 — 카메라 및 영상 처리 기초

> **소요 시간**: 3~4시간  
> **난이도**: ★★★☆☆  
> **준비물**: CSI 카메라(IMX219, Raspberry Pi Camera V2), USB 카메라(선택)

---

## 1. 개요

Jetson Nano의 강점은 GPU 가속 영상 처리입니다.  
CSI 카메라 인터페이스, GStreamer 파이프라인, CUDA 가속 OpenCV를 다룹니다.

### 학습 목표
- CSI 카메라를 연결하고 `nvgstcapture`로 캡처할 수 있다
- GStreamer 파이프라인의 기본 구조를 이해한다
- OpenCV를 CUDA 가속으로 빌드/설치할 수 있다
- OpenCV로 카메라 영상을 캡처하고 기본 처리를 수행할 수 있다
- GPU 가속 OpenCV(cv2.cuda)를 사용할 수 있다
- USB 카메라와 CSI 카메라의 성능 차이를 이해한다

### 사전 지식
- Python 기본 문법
- Module 00의 JetPack 설치 완료

---

## 2. 이론

### 2.1 CSI 카메라 인터페이스

Jetson Nano의 CSI( Camera Serial Interface)는 MIPI CSI-2 표준을 사용합니다.

| 항목 | 사양 |
|------|------|
| 커넥터 | 15-pin FFC (Raspberry Pi Camera 호환) |
| 레인 수 | 12-lane (3×4 또는 4×2) |
| 최대 속도 | 1.5 Gb/s per lane |
| 지원 카메라 | IMX219 (RPi Camera V2), OV5640 등 |

**CSI vs USB 카메라**:

| 항목 | CSI 카메라 | USB 카메라 |
|------|-----------|-----------|
| 대역폭 | 전용 MIPI 레인 | USB 3.0 공유 |
| CPU 부하 | 낮음 (하드웨어 처리) | 높음 (USB isochronous) |
| GPU 가속 | NVIDIA 하드웨어 가속 | 제한적 |
| 최대 해상도 | 3280×2464 (IMX219) | 다양 (최대 4K) |
| 지연 시간 | 낮음 | 중간 |

### 2.2 GStreamer 파이프라인

Jetson Nano에서 카메라 영상 처리는 GStreamer 파이프라인을 통해 이루어집니다.

**기본 CSI 카메라 파이프라인**:
```
nvarguscamerasrc → nvvidconv → nvegltransform → nveglglessink
```

| 요소 | 역할 |
|------|------|
| `nvarguscamerasrc` | CSI 카메라에서 RAW 데이터 캡처 (GPU 가속) |
| `nvvidconv` | 색공간/해상도 변환 (GPU 가속) |
| `nvegltransform` | EGL 이미지 변환 |
| `nveglglessink` | 디스플레이 출력 |
| `appsink` | OpenCV 등 애플리케이션으로 전달 |

**OpenCV용 파이프라인**:
```python
# CSI 카메라 → OpenCV
gst_pipeline = (
    "nvarguscamerasrc ! "
    "video/x-raw(memory:NVMM), width=1280, height=720, format=NV12, framerate=30/1 ! "
    "nvvidconv ! "
    "video/x-raw, format=BGRx ! "
    "videoconvert ! "
    "video/x-raw, format=BGR ! "
    "appsink drop=1"
)
```

### 2.3 OpenCV CUDA 가속

Jetson Nano의 JetPack에는 CUDA 가속 OpenCV가 포함되어 있습니다.

```python
import cv2

# CUDA 가속 사용 가능 확인
print(cv2.cuda.getCudaEnabledDeviceCount())  # 1 이상이면 CUDA 사용 가능

# GPU 업로드
src_gpu = cv2.cuda_GpuMat()
src_gpu.upload(cpu_mat)

# GPU 처리
gray_gpu = cv2.cuda.cvtColor(src_gpu, cv2.COLOR_BGR2GRAY)

# GPU → CPU 다운로드
result = gray_gpu.download()
```

---

## 3. 실습 코드

### 실습 7-1: CSI 카메라 기본 캡처

```bash
# nvgstcapture로 카메라 테스트
nvgstcapture-1.0 --camsrc=0 --cap-time=5000

# 특정 해상도
nvgstcapture-1.0 --camsrc=0 --cap-time=3000 --cus-prev-width=1920 --cus-prev-height=1080
```

**코드**: `csi_camera_basic.py`
```python
#!/usr/bin/env python3
"""
CSI 카메라 기본 캡처 — OpenCV + GStreamer
"""

import cv2
import numpy as np

# CSI 카메라 GStreamer 파이프라인
CSI_PIPELINE = (
    "nvarguscamerasrc ! "
    "video/x-raw(memory:NVMM), width=1280, height=720, format=NV12, framerate=30/1 ! "
    "nvvidconv flip-method=0 ! "
    "video/x-raw, format=BGRx ! "
    "videoconvert ! "
    "video/x-raw, format=BGR ! "
    "appsink drop=1"
)

def main():
    print("CSI 카메라 캡처 시작")
    print("  해상도: 1280×720 @ 30fps")
    print("  ESC 키: 종료")
    print("  s 키  : 스크린샷 저장")
    print("-" * 40)

    cap = cv2.VideoCapture(CSI_PIPELINE, cv2.CAP_GSTREAMER)

    if not cap.isOpened():
        print("카메라를 열 수 없습니다.")
        print("1) CSI 카메라가 올바르게 연결되었는지 확인")
        print("2) ls /dev/video* 로 비디오 장치 확인")
        return

    frame_count = 0

    while True:
        ret, frame = cap.read()
        if not ret:
            print("프레임 읽기 실패")
            break

        frame_count += 1

        # 프레임 정보 표시
        h, w = frame.shape[:2]
        fps = cap.get(cv2.CAP_PROP_FPS)

        cv2.putText(frame, f"CSI Camera {w}x{h} @ {fps:.0f}fps",
                    (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)
        cv2.putText(frame, f"Frame #{frame_count}",
                    (10, 60), cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 0), 2)

        # 화면 출력
        cv2.imshow("Jetson Nano CSI Camera", frame)

        key = cv2.waitKey(1) & 0xFF
        if key == 27:  # ESC
            break
        elif key == ord('s'):
            filename = f"capture_{frame_count:04d}.jpg"
            cv2.imwrite(filename, frame)
            print(f"저장: {filename}")

    cap.release()
    cv2.destroyAllWindows()
    print(f"\n총 {frame_count} 프레임 캡처")

if __name__ == "__main__":
    main()
```

**실행**:
```bash
python3 csi_camera_basic.py
```

### 실습 7-2: 실시간 영상 처리 (GPU 가속)

**코드**: `gpu_video_processing.py`
```python
#!/usr/bin/env python3
"""
GPU 가속 실시간 영상 처리
Canny Edge Detection — CPU vs GPU 성능 비교
"""

import cv2
import numpy as np
import time

CSI_PIPELINE = (
    "nvarguscamerasrc ! "
    "video/x-raw(memory:NVMM), width=1280, height=720, format=NV12, framerate=30/1 ! "
    "nvvidconv flip-method=0 ! "
    "video/x-raw, format=BGRx ! "
    "videoconvert ! "
    "video/x-raw, format=BGR ! "
    "appsink drop=1"
)


def process_cpu(frame):
    """CPU 기반 Canny Edge"""
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    blurred = cv2.GaussianBlur(gray, (5, 5), 1.0)
    edges = cv2.Canny(blurred, 50, 150)
    return edges


def process_gpu(frame):
    """GPU(CUDA) 기반 Canny Edge"""
    # CPU → GPU 업로드
    src_gpu = cv2.cuda_GpuMat()
    src_gpu.upload(frame)

    # GPU 처리
    gray_gpu = cv2.cuda.cvtColor(src_gpu, cv2.COLOR_BGR2GRAY)
    blurred_gpu = cv2.cuda.GaussianBlur(gray_gpu, (5, 5), 1.0)
    edges_gpu = cv2.cuda.Canny(blurred_gpu, 50, 150)

    # GPU → CPU 다운로드
    edges = edges_gpu.download()
    return edges


def main():
    print("GPU 가속 영상 처리 — CPU vs GPU 성능 비교")
    print("  ESC: 종료,  c: CPU 모드,  g: GPU 모드")
    print("-" * 50)

    cap = cv2.VideoCapture(CSI_PIPELINE, cv2.CAP_GSTREAMER)
    if not cap.isOpened():
        print("카메라 열기 실패")
        return

    use_gpu = True
    cpu_times = []
    gpu_times = []

    while True:
        ret, frame = cap.read()
        if not ret:
            break

        start = time.perf_counter()

        if use_gpu and cv2.cuda.getCudaEnabledDeviceCount() > 0:
            edges = process_gpu(frame)
            mode = "GPU"
        else:
            edges = process_cpu(frame)
            mode = "CPU"

        elapsed = (time.perf_counter() - start) * 1000  # ms

        # 성능 통계
        if mode == "GPU":
            gpu_times.append(elapsed)
        else:
            cpu_times.append(elapsed)

        # 결과 표시
        h, w = frame.shape[:2]

        # 원본 + 엣지 나란히 표시
        edges_bgr = cv2.cvtColor(edges, cv2.COLOR_GRAY2BGR)
        display = np.hstack((frame, edges_bgr))

        avg = f"평균: {sum(gpu_times if use_gpu else cpu_times[-30:]) / max(len(gpu_times if use_gpu else cpu_times[-30:]), 1):.1f}ms"
        cv2.putText(display, f"Mode: {mode}  Latency: {elapsed:.1f}ms  {avg}",
                    (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)
        cv2.putText(display, f"CPU samples: {len(cpu_times)}  GPU samples: {len(gpu_times)}",
                    (10, 60), cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 0), 2)

        # 가이드 라인
        cv2.line(display, (w, 0), (w, h), (0, 255, 255), 2)

        cv2.imshow("Jetson Nano GPU Video Processing", display)

        key = cv2.waitKey(1) & 0xFF
        if key == 27:
            break
        elif key == ord('c'):
            use_gpu = False
            print("CPU 모드 전환")
        elif key == ord('g'):
            use_gpu = True
            print("GPU 모드 전환")

    cap.release()
    cv2.destroyAllWindows()

    # 최종 통계
    if cpu_times:
        print(f"\nCPU 평균: {sum(cpu_times)/len(cpu_times):.1f}ms")
    if gpu_times:
        print(f"GPU 평균: {sum(gpu_times)/len(gpu_times):.1f}ms")


if __name__ == "__main__":
    main()
```

### 실습 7-3: USB 카메라

**코드**: `usb_camera.py`
```python
#!/usr/bin/env python3
"""
USB 카메라 캡처 — CSI 카메라와 성능 비교
"""

import cv2
import time

def main():
    print("USB 카메라 테스트")
    print("  ESC: 종료")
    print("-" * 30)

    # USB 카메라 열기 (0 = 첫 번째 USB 카메라)
    cap = cv2.VideoCapture(0, cv2.CAP_V4L2)

    if not cap.isOpened():
        print("USB 카메라를 열 수 없습니다.")
        print("연결 확인: ls /dev/video*")
        return

    # 해상도 설정
    cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
    cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)
    cap.set(cv2.CAP_PROP_FPS, 30)

    print(f"실제 해상도: {int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))}x"
          f"{int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))}")
    print(f"실제 FPS: {cap.get(cv2.CAP_PROP_FPS)}")

    frame_count = 0
    fps_start = time.perf_counter()
    fps_count = 0

    while True:
        ret, frame = cap.read()
        if not ret:
            break

        frame_count += 1
        fps_count += 1

        # 1초마다 FPS 계산
        if time.perf_counter() - fps_start >= 1.0:
            actual_fps = fps_count
            fps_count = 0
            fps_start = time.perf_counter()
        else:
            actual_fps = 0

        cv2.putText(frame, f"USB Camera #{frame_count}",
                    (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)
        if actual_fps > 0:
            cv2.putText(frame, f"FPS: {actual_fps}",
                        (10, 60), cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 0), 2)

        cv2.imshow("USB Camera", frame)

        if cv2.waitKey(1) & 0xFF == 27:
            break

    cap.release()
    cv2.destroyAllWindows()

if __name__ == "__main__":
    main()
```

---

## 4. 최종 실습 — 움직임 감지 카메라

프레임 차분(Frame Differencing)으로 움직임을 감지하고 로깅합니다.

**코드**: `motion_detector.py`
```python
#!/usr/bin/env python3
"""
움직임 감지 카메라 — 프레임 차분 기반
움직임 감지 시: 사진 저장 + 타임스탬프 로깅
"""

import cv2
import numpy as np
import time
import os
from datetime import datetime

CSI_PIPELINE = (
    "nvarguscamerasrc ! "
    "video/x-raw(memory:NVMM), width=640, height=480, format=NV12, framerate=30/1 ! "
    "nvvidconv flip-method=0 ! "
    "video/x-raw, format=BGRx ! "
    "videoconvert ! "
    "video/x-raw, format=BGR ! "
    "appsink drop=1"
)


class MotionDetector:
    def __init__(self, threshold=25, min_area=500, capture_dir="motion_captures"):
        self.threshold = threshold
        self.min_area = min_area
        self.capture_dir = capture_dir
        os.makedirs(capture_dir, exist_ok=True)

        self.prev_gray = None
        self.motion_count = 0
        self.is_motion = False
        self.cooldown = 0

    def detect(self, frame):
        """
        프레임 차분으로 움직임 감지
        반환: (움직임_있음, 움직임_마스크, 변화_비율)
        """
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        gray = cv2.GaussianBlur(gray, (21, 21), 0)

        if self.prev_gray is None:
            self.prev_gray = gray
            return False, None, 0.0

        # 현재 프레임 - 이전 프레임 차분
        diff = cv2.absdiff(self.prev_gray, gray)
        _, thresh = cv2.threshold(diff, self.threshold, 255, cv2.THRESH_BINARY)

        # 노이즈 제거
        thresh = cv2.erode(thresh, None, iterations=2)
        thresh = cv2.dilate(thresh, None, iterations=2)

        # 컨투어 찾기
        contours, _ = cv2.findContours(thresh, cv2.RETR_EXTERNAL,
                                        cv2.CHAIN_APPROX_SIMPLE)

        # 움직임 영역 계산
        motion_area = 0
        for cnt in contours:
            area = cv2.contourArea(cnt)
            if area > self.min_area:
                motion_area += area
                # 바운딩 박스 그리기
                x, y, w, h = cv2.boundingRect(cnt)
                cv2.rectangle(frame, (x, y), (x + w, y + h), (0, 0, 255), 2)

        change_ratio = motion_area / (frame.shape[0] * frame.shape[1])
        has_motion = change_ratio > 0.001  # 0.1% 이상 변화

        # 쿨다운: 최소 1초 간격으로만 감지
        now = time.time()
        if has_motion and now > self.cooldown:
            self.cooldown = now + 1.0
            if not self.is_motion:
                self.motion_count += 1
                self.is_motion = True
                self._save_capture(frame)
        else:
            self.is_motion = False

        self.prev_gray = gray
        return has_motion, thresh, change_ratio

    def _save_capture(self, frame):
        """움직임 감지 시 사진 저장"""
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"{self.capture_dir}/motion_{timestamp}.jpg"
        cv2.imwrite(filename, frame)
        print(f"[움직임 #{self.motion_count}] 저장: {filename}")


def main():
    print("움직임 감지 카메라 시작")
    print("  ESC: 종료")
    print("-" * 40)

    cap = cv2.VideoCapture(CSI_PIPELINE, cv2.CAP_GSTREAMER)
    if not cap.isOpened():
        print("카메라 열기 실패")
        return

    detector = MotionDetector(threshold=25, min_area=500)
    start_time = time.time()

    try:
        while True:
            ret, frame = cap.read()
            if not ret:
                break

            # 움직임 감지
            has_motion, mask, ratio = detector.detect(frame.copy())

            # 정보 표시
            elapsed = time.time() - start_time
            status = "MOTION!" if has_motion else "Idle"
            color = (0, 0, 255) if has_motion else (0, 255, 0)

            cv2.putText(frame, f"Status: {status}", (10, 30),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.7, color, 2)
            cv2.putText(frame, f"Motion: {detector.motion_count}", (10, 55),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 0), 2)
            cv2.putText(frame, f"Change: {ratio*100:.2f}%", (10, 80),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 0), 2)
            cv2.putText(frame, f"Time: {int(elapsed)}s", (10, 105),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 0), 2)

            cv2.imshow("Motion Detector", frame)

            if cv2.waitKey(1) & 0xFF == 27:
                break

    except KeyboardInterrupt:
        pass
    finally:
        cap.release()
        cv2.destroyAllWindows()
        print(f"\n총 {detector.motion_count}회 움직임 감지")
        print(f"캡처 저장 위치: {detector.capture_dir}/")


if __name__ == "__main__":
    main()
```

---

## 5. 확인 및 테스트

```
1. CSI 카메라: csi_camera_basic.py → 실시간 영상 출력 확인
2. GPU 처리: gpu_video_processing.py → 'c'/'g'로 CPU/GPU 모드 전환, 성능 비교
3. USB 카메라: usb_camera.py → USB 카메라 영상 출력 확인
4. 움직임 감지: motion_detector.py → 손 흔들면 MOTION! 표시 + 사진 저장
```

> **다음 모듈**: Module 08 — 영상 처리 심화 및 컴퓨터 비전

# Module 09 — NVIDIA Jetson AI / 딥러닝

> **소요 시간**: 4~6시간  
> **난이도**: ★★★★☆  
> **준비물**: CSI 카메라, 인터넷 연결 (모델 다운로드)

---

## 1. 개요

Jetson Nano의 핵심 가치는 Edge AI 추론입니다. TensorRT 최적화 모델로 실시간 객체 탐지, 분류, 세그멘테이션을 수행합니다.

### 학습 목표
- TensorRT의 FP16/INT8 양자화 개념을 이해한다
- `jetson-inference` 라이브러리로 추론을 수행할 수 있다
- ImageNet 분류, DetectNet 객체 탐지, SegNet 세그멘테이션을 실행할 수 있다
- YOLO 모델을 TensorRT로 변환하여 실행할 수 있다
- PyTorch → ONNX → TensorRT 파이프라인을 이해한다

---

## 2. 이론

### 2.1 TensorRT

NVIDIA TensorRT는 추론 최적화 엔진입니다:

| 최적화 | 효과 |
|--------|------|
| FP16 (반정밀도) | 2배 속도 향상, 정밀도 손실 거의 없음 |
| INT8 (정수 양자화) | 4배 속도 향상, 약간의 정밀도 손실 |
| Layer Fusion | 커널 병합으로 메모리 대역폭 감소 |
| Kernel Auto-tuning | 최적 CUDA 커널 자동 선택 |

### 2.2 jetson-inference

NVIDIA 공식 추론 라이브러리:

| 클래스 | 용도 | 출력 |
|--------|------|------|
| `imageNet` | 이미지 분류 | Top-5 클래스 + 확률 |
| `detectNet` | 객체 탐지 | 바운딩 박스 + 클래스 + 확률 |
| `segNet` | 의미적 분할 | 픽셀 단위 클래스 맵 |

---

## 3. 실습 코드

### 실습 9-1: jetson-inference 설치

```bash
# jetson-inference 클론 및 설치
git clone --recursive https://github.com/dusty-nv/jetson-inference.git
cd jetson-inference
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
sudo ldconfig

# Python 패키지 설치 확인
python3 -c "import jetson.inference; print('OK')"
```

### 실습 9-2: 이미지 분류 (imageNet)

**코드**: `imagenet_classify.py`
```python
#!/usr/bin/env python3
"""
ImageNet 실시간 이미지 분류
jetson.inference.imageNet 사용
"""

import jetson.inference
import jetson.utils
import cv2
import numpy as np

# ImageNet 로드 (GoogLeNet 기본)
net = jetson.inference.imageNet("googlenet", threshold=0.5)

# CSI 카메라
csi = jetson.utils.gstCamera(1280, 720, "0")

# 디스플레이
display = jetson.utils.glDisplay()

font = jetson.utils.cudaFont()

while display.IsOpen():
    img, width, height = csi.CaptureRGBA()

    # 추론
    class_idx, confidence = net.Classify(img, width, height)

    if class_idx >= 0:
        class_desc = net.GetClassDesc(class_idx)
        font.OverlayText(img, width, height,
                        f"{class_desc} ({confidence:.2f})",
                        10, 10, font.White, font.Gray40)

    font.OverlayText(img, width, height,
                    f"FPS: {net.GetNetworkFPS():.1f}",
                    10, height - 30, font.White, font.Gray40)

    display.RenderOnce(img, width, height)
```

**실행**:
```bash
python3 imagenet_classify.py
```

### 실습 9-3: 객체 탐지 (detectNet + OpenCV)

**코드**: `detectnet_opencv.py`
```python
#!/usr/bin/env python3
"""
객체 탐지 — detectNet + OpenCV 디스플레이
SSD-Mobilenet-v2로 실시간 객체 탐지
"""

import jetson.inference
import jetson.utils
import cv2
import numpy as np

# detectNet 로드
net = jetson.inference.detectNet("ssd-mobilenet-v2", threshold=0.5)

CSI_PIPELINE = (
    "nvarguscamerasrc ! "
    "video/x-raw(memory:NVMM), width=1280, height=720, format=NV12, framerate=30/1 ! "
    "nvvidconv flip-method=0 ! "
    "video/x-raw, format=BGRx ! videoconvert ! "
    "video/x-raw, format=BGR ! appsink drop=1"
)

def cuda_to_numpy(cuda_img):
    """CUDA 이미지 → numpy 배열 변환"""
    cuda_img = jetsons.utils.cudaToNumpy(cuda_img, 1280, 720, 4)
    return cv2.cvtColor(cuda_img, cv2.COLOR_RGBA2BGR)

def main():
    cap = cv2.VideoCapture(CSI_PIPELINE, cv2.CAP_GSTREAMER)
    if not cap.isOpened():
        print("카메라 열기 실패")
        return

    print("실시간 객체 탐지 시작")
    print("  ESC: 종료")

    while True:
        ret, frame = cap.read()
        if not ret:
            break

        # numpy → CUDA 이미지 변환
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        cuda_img = jetson.utils.cudaFromNumpy(rgb)

        # 추론
        detections = net.Detect(cuda_img, 1280, 720)

        # 결과를 OpenCV 프레임에 직접 표시
        for det in detections:
            left, top, right, bottom = int(det.Left), int(det.Top), int(det.Right), int(det.Bottom)
            class_name = net.GetClassDesc(det.ClassID)
            confidence = det.Confidence

            cv2.rectangle(frame, (left, top), (right, bottom), (0, 255, 0), 2)
            label = f"{class_name} {confidence:.2f}"
            cv2.putText(frame, label, (left, top - 10),
                       cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 255, 0), 2)

        cv2.putText(frame, f"Detections: {len(detections)}",
                   (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (255, 255, 0), 2)
        cv2.imshow("Object Detection", frame)

        if cv2.waitKey(1) & 0xFF == 27:
            break

    cap.release()
    cv2.destroyAllWindows()

if __name__ == "__main__":
    main()
```

---

## 4. 최종 실습 — Custom Model (PyTorch → ONNX → TensorRT)

**코드**: `custom_model_pipeline.py`
```python
#!/usr/bin/env python3
"""
Custom Model 배포 파이프라인
PyTorch → ONNX → TensorRT 엔진 변환 → 추론

※ Jetson Nano에서 직접 PyTorch 실행은 느리므로,
  PC에서 ONNX까지 변환 후 Jetson으로 복사하는 것을 권장
"""

import torch
import torchvision.models as models
import numpy as np
import os

# ============================================================
# 1단계: PyTorch → ONNX 변환 (PC 또는 Jetson에서)
# ============================================================
def export_to_onnx():
    """사전학습 모델을 ONNX로 변환"""
    model = models.resnet18(pretrained=True)
    model.eval()

    dummy_input = torch.randn(1, 3, 224, 224)
    torch.onnx.export(
        model,
        dummy_input,
        "resnet18.onnx",
        input_names=["input"],
        output_names=["output"],
        opset_version=11
    )
    print("ONNX 변환 완료: resnet18.onnx")

# ============================================================
# 2단계: ONNX → TensorRT 변환 (Jetson에서)
# ============================================================
def build_trt_engine(onnx_path, engine_path, precision="fp16"):
    """ONNX → TensorRT 엔진 빌드"""
    import tensorrt as trt

    TRT_LOGGER = trt.Logger(trt.Logger.INFO)
    builder = trt.Builder(TRT_LOGGER)
    network = builder.create_network(1 << int(trt.NetworkDefinitionCreationFlag.EXPLICIT_BATCH))
    parser = trt.OnnxParser(network, TRT_LOGGER)

    # ONNX 로드
    with open(onnx_path, 'rb') as f:
        if not parser.parse(f.read()):
            for error in range(parser.num_errors):
                print(parser.get_error(error))
            return None

    config = builder.create_builder_config()
    config.set_memory_pool_limit(trt.MemoryPoolType.WORKSPACE, 1 << 30)  # 1GB

    if precision == "fp16" and builder.platform_has_fast_fp16:
        config.set_flag(trt.BuilderFlag.FP16)
        print("FP16 모드 활성화")

    # 엔진 빌드
    serialized_engine = builder.build_serialized_network(network, config)
    if serialized_engine is None:
        print("TensorRT 엔진 빌드 실패")
        return None

    with open(engine_path, 'wb') as f:
        f.write(serialized_engine)

    print(f"TensorRT 엔진 저장: {engine_path}")
    return engine_path


# ============================================================
# 3단계: TensorRT 추론 (Jetson에서)
# ============================================================
class TRTInference:
    """TensorRT 추론 래퍼"""

    def __init__(self, engine_path):
        import tensorrt as trt
        import pycuda.driver as cuda

        self.TRT_LOGGER = trt.Logger(trt.Logger.WARNING)
        self.runtime = trt.Runtime(self.TRT_LOGGER)

        with open(engine_path, 'rb') as f:
            self.engine = self.runtime.deserialize_cuda_engine(f.read())

        self.context = self.engine.create_execution_context()

        # 버퍼 할당
        self.inputs = []
        self.outputs = []
        self.allocations = []

        for i in range(self.engine.num_io_tensors):
            name = self.engine.get_tensor_name(i)
            dtype = trt.nptype(self.engine.get_tensor_dtype(name))
            shape = self.engine.get_tensor_shape(name)
            size = np.prod(shape)

            # 실제 배치 크기로 업데이트
            if self.engine.get_tensor_mode(name) == trt.TensorIOMode.INPUT:
                self.context.set_input_shape(name, shape)

            allocation = cuda.mem_alloc(size * np.dtype(dtype).itemsize)
            self.allocations.append(allocation)

            if self.engine.get_tensor_mode(name) == trt.TensorIOMode.INPUT:
                self.inputs.append({'name': name, 'dtype': dtype, 'shape': shape, 'allocation': allocation})
            else:
                self.outputs.append({'name': name, 'dtype': dtype, 'shape': shape, 'allocation': allocation})

    def infer(self, input_numpy):
        """추론 실행"""
        import pycuda.driver as cuda

        # 입력 복사
        cuda.memcpy_htod(self.inputs[0]['allocation'], input_numpy)

        for inp in self.inputs:
            self.context.set_tensor_address(inp['name'], int(inp['allocation']))
        for out in self.outputs:
            self.context.set_tensor_address(out['name'], int(out['allocation']))

        # 추론
        self.context.execute_async_v3(0)

        # 출력 복사
        output = np.zeros(self.outputs[0]['shape'], dtype=self.outputs[0]['dtype'])
        cuda.memcpy_dtoh(output, self.outputs[0]['allocation'])

        return output


def main():
    print("Custom Model Pipeline")
    print("=" * 40)
    print("1. PyTorch → ONNX (PC 권장)")
    print("2. ONNX → TensorRT (Jetson)")
    print("3. TensorRT 추론 (Jetson)")
    print("-" * 40)

    onnx_path = "resnet18.onnx"
    engine_path = "resnet18_fp16.engine"

    # ONNX가 없으면 변환
    if not os.path.exists(onnx_path):
        print("[1] PyTorch → ONNX 변환")
        export_to_onnx()
    else:
        print(f"[1] ONNX 파일 존재: {onnx_path}")

    # TensorRT 엔진이 없으면 빌드
    if not os.path.exists(engine_path):
        print("[2] ONNX → TensorRT 엔진 빌드")
        build_trt_engine(onnx_path, engine_path, precision="fp16")
    else:
        print(f"[2] TensorRT 엔진 존재: {engine_path}")

    print("\n파이프라인 준비 완료!")
    print(f"  ONNX: {onnx_path}")
    print(f"  Engine: {engine_path}")
    print("\n추론은 jetson.inference 또는 직접 TRT 런타임 사용")


if __name__ == "__main__":
    main()
```

---

## 5. 확인 및 테스트

```
1. jetson-inference 설치 확인
2. imagenet_classify.py → 카메라 앞 물체 분류 결과 실시간 출력
3. detectnet_opencv.py → 사람/자동차 등 객체 탐지 바운딩 박스 확인
4. custom_model_pipeline.py → ONNX/TensorRT 엔진 변환 파이프라인 실행
```

> **다음 모듈**: Module 10 — 종합 프로젝트

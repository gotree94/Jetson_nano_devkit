# Module 08 — 영상 처리 심화 및 컴퓨터 비전

> **소요 시간**: 4~5시간  
> **난이도**: ★★★★☆  
> **준비물**: CSI 카메라, 체커보드(캘리브레이션), ArUco 마커(출력물)

---

## 1. 개요

기본 영상 처리에서 나아가 객체 추적, 특징점 매칭, Optical Flow, ArUco 마커, 카메라 캘리브레이션, 스테레오 비전까지 다룹니다.

### 학습 목표
- HSV 색공간으로 객체를 추적할 수 있다
- 특징점(SIFT/ORB)을 검출하고 매칭할 수 있다
- Optical Flow로 움직임 벡터를 계산할 수 있다
- ArUco 마커를 생성하고 포즈를 추정할 수 있다
- 카메라 캘리브레이션을 수행하고 왜곡을 보정할 수 있다

---

## 2. 이론

### 2.1 HSV 색공간

HSV는 색상(Hue), 채도(Saturation), 명도(Value)로 구성된 색공간입니다.  
BGR보다 조명 변화에 강건하여 객체 추적에 유리합니다.

```python
# BGR → HSV 변환
hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)

# 특정 색상 범위 마스킹 (빨간색 예)
lower_red = np.array([0, 100, 100])
upper_red = np.array([10, 255, 255])
mask = cv2.inRange(hsv, lower_red, upper_red)
```

### 2.2 카메라 캘리브레이션

렌즈 왜곡(특히 광각)을 보정하기 위해 카메라의 내부 파라미터를 구합니다.

```
왜곡된 이미지 → 캘리브레이션 → 보정된 이미지

필요: 체커보드 패턴 (9×6 내부 코너)
최소 10~20장 다양한 각도에서 촬영
```

---

## 3. 실습 코드

### 실습 8-1: 색상 기반 객체 추적

**코드**: `color_tracking.py`
```python
#!/usr/bin/env python3
"""
색상 기반 객체 추적 — HSV 마스킹 + 컨투어 + 중심점 추적
"""

import cv2
import numpy as np

CSI_PIPELINE = (
    "nvarguscamerasrc ! "
    "video/x-raw(memory:NVMM), width=640, height=480, format=NV12, framerate=30/1 ! "
    "nvvidconv flip-method=0 ! "
    "video/x-raw, format=BGRx ! videoconvert ! video/x-raw, format=BGR ! appsink drop=1"
)

# 트랙바 콜백
def nothing(x):
    pass

def main():
    cap = cv2.VideoCapture(CSI_PIPELINE, cv2.CAP_GSTREAMER)

    cv2.namedWindow("Tracking")
    cv2.createTrackbar("LH", "Tracking", 0, 179, nothing)
    cv2.createTrackbar("LS", "Tracking", 100, 255, nothing)
    cv2.createTrackbar("LV", "Tracking", 100, 255, nothing)
    cv2.createTrackbar("UH", "Tracking", 10, 179, nothing)
    cv2.createTrackbar("US", "Tracking", 255, 255, nothing)
    cv2.createTrackbar("UV", "Tracking", 255, 255, nothing)

    while True:
        ret, frame = cap.read()
        if not ret:
            break

        hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)

        lh = cv2.getTrackbarPos("LH", "Tracking")
        ls = cv2.getTrackbarPos("LS", "Tracking")
        lv = cv2.getTrackbarPos("LV", "Tracking")
        uh = cv2.getTrackbarPos("UH", "Tracking")
        us = cv2.getTrackbarPos("US", "Tracking")
        uv = cv2.getTrackbarPos("UV", "Tracking")

        lower = np.array([lh, ls, lv])
        upper = np.array([uh, us, uv])

        mask = cv2.inRange(hsv, lower, upper)
        mask = cv2.erode(mask, None, iterations=2)
        mask = cv2.dilate(mask, None, iterations=2)

        result = cv2.bitwise_and(frame, frame, mask=mask)

        # 컨투어
        contours, _ = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
        if contours:
            largest = max(contours, key=cv2.contourArea)
            if cv2.contourArea(largest) > 300:
                x, y, w, h = cv2.boundingRect(largest)
                cv2.rectangle(frame, (x, y), (x+w, y+h), (0, 255, 0), 2)
                cv2.circle(frame, (x+w//2, y+h//2), 5, (0, 0, 255), -1)
                cv2.putText(frame, f"({x+w//2}, {y+h//2})",
                            (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)

        cv2.imshow("Original", frame)
        cv2.imshow("Mask", mask)
        cv2.imshow("Result", result)

        if cv2.waitKey(1) & 0xFF == 27:
            break

    cap.release()
    cv2.destroyAllWindows()

if __name__ == "__main__":
    main()
```

### 실습 8-2: 특징점 검출 및 매칭

**코드**: `feature_matching.py`
```python
#!/usr/bin/env python3
"""
ORB 특징점 검출 및 매칭 — 두 영상 간 특징점 매칭
"""

import cv2
import numpy as np

def main():
    # 참조 이미지 (스마트폰으로 미리 촬영)
    ref_img = cv2.imread("reference.jpg")
    if ref_img is None:
        print("reference.jpg 파일이 필요합니다.")
        print("스마트폰으로 물체를 촬영하여 reference.jpg로 저장하세요.")
        return

    # ORB 검출기
    orb = cv2.ORB_create(nfeatures=1000)

    # 참조 이미지 특징점
    kp1, des1 = orb.detectAndCompute(ref_img, None)
    ref_kp_img = cv2.drawKeypoints(ref_img, kp1, None, color=(0, 255, 0))
    cv2.imshow("Reference Keypoints", ref_kp_img)

    # BF 매처
    bf = cv2.BFMatcher(cv2.NORM_HAMMING, crossCheck=True)

    CSI_PIPELINE = (
        "nvarguscamerasrc ! "
        "video/x-raw(memory:NVMM), width=640, height=480, format=NV12, framerate=30/1 ! "
        "nvvidconv flip-method=0 ! "
        "video/x-raw, format=BGRx ! videoconvert ! "
        "video/x-raw, format=BGR ! appsink drop=1"
    )

    cap = cv2.VideoCapture(CSI_PIPELINE, cv2.CAP_GSTREAMER)

    while True:
        ret, frame = cap.read()
        if not ret:
            break

        kp2, des2 = orb.detectAndCompute(frame, None)

        if des2 is not None and len(kp2) > 5:
            matches = bf.match(des1, des2)
            matches = sorted(matches, key=lambda x: x.distance)[:50]

            match_img = cv2.drawMatches(
                ref_img, kp1, frame, kp2, matches, None,
                flags=cv2.DrawMatchesFlags_NOT_DRAW_SINGLE_POINTS
            )

            cv2.putText(match_img, f"Matches: {len(matches)}",
                        (10, 30), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 255, 0), 2)
            cv2.imshow("Feature Matching", match_img)
        else:
            cv2.imshow("Feature Matching", frame)

        if cv2.waitKey(1) & 0xFF == 27:
            break

    cap.release()
    cv2.destroyAllWindows()

if __name__ == "__main__":
    main()
```

---

## 4. 최종 실습 — ArUco 마커 포즈 추정

**코드**: `aruco_pose.py`
```python
#!/usr/bin/env python3
"""
ArUco 마커 생성 + 실시간 포즈 추정
마커 위에 3D 축을 오버레이
"""

import cv2
import cv2.aruco as aruco
import numpy as np

# 마커 생성
def generate_marker(marker_id=0, size=200):
    """ArUco 마커 이미지 생성 및 저장"""
    dict = aruco.getPredefinedDictionary(aruco.DICT_6X6_250)
    img = aruco.drawMarker(dict, marker_id, size)
    cv2.imwrite(f"aruco_marker_{marker_id}.png", img)
    print(f"마커 저장: aruco_marker_{marker_id}.png")
    cv2.imshow(f"ArUco Marker {marker_id}", img)
    cv2.waitKey(0)
    cv2.destroyWindow(f"ArUco Marker {marker_id}")

# 마커 생성 (최초 1회 실행)
generate_marker(0)

# 카메라 캘리브레이션 파라미터 (Jetson Nano CSI IMX219)
# 실제로는 캘리브레이션 필요, 여기서는 IMX219 기본값 사용
CAM_MATRIX = np.array([
    [921.0, 0, 640.0],
    [0, 921.0, 360.0],
    [0, 0, 1.0]
], dtype=np.float32)

DIST_COEFFS = np.zeros((5, 1), dtype=np.float32)

CSI_PIPELINE = (
    "nvarguscamerasrc ! "
    "video/x-raw(memory:NVMM), width=1280, height=720, format=NV12, framerate=30/1 ! "
    "nvvidconv flip-method=0 ! "
    "video/x-raw, format=BGRx ! videoconvert ! "
    "video/x-raw, format=BGR ! appsink drop=1"
)

def draw_axes(img, rvec, tvec, cam_matrix, dist_coeffs, length=0.05):
    """3D 축 그리기 (X=빨강, Y=초록, Z=파랑)"""
    axis_pts = np.float32([
        [0, 0, 0],
        [length, 0, 0],
        [0, length, 0],
        [0, 0, -length]
    ])
    img_pts, _ = cv2.projectPoints(axis_pts, rvec, tvec, cam_matrix, dist_coeffs)
    origin = tuple(img_pts[0].ravel().astype(int))

    colors = [(0, 0, 255), (0, 255, 0), (255, 0, 0)]  # X, Y, Z
    for i in range(3):
        pt = tuple(img_pts[i + 1].ravel().astype(int))
        cv2.line(img, origin, pt, colors[i], 3)
        cv2.circle(img, pt, 5, colors[i], -1)

def main():
    cap = cv2.VideoCapture(CSI_PIPELINE, cv2.CAP_GSTREAMER)
    dict = aruco.getPredefinedDictionary(aruco.DICT_6X6_250)
    params = aruco.DetectorParameters()

    print("ArUco 마커 포즈 추정")
    print("  마커 ID 0 을 카메라 앞에 보여주세요")
    print("  ESC: 종료")

    while True:
        ret, frame = cap.read()
        if not ret:
            break

        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        corners, ids, _ = aruco.detectMarkers(gray, dict, parameters=params)

        if ids is not None:
            aruco.drawDetectedMarkers(frame, corners, ids)

            for i in range(len(ids)):
                rvec, tvec, _ = aruco.estimatePoseSingleMarkers(
                    corners[i], 0.05, CAM_MATRIX, DIST_COEFFS
                )
                draw_axes(frame, rvec[0], tvec[0], CAM_MATRIX, DIST_COEFFS, 0.03)

                # 거리 정보
                dist = np.linalg.norm(tvec[0])
                cv2.putText(frame, f"ID:{ids[i][0]} Dist:{dist:.2f}m",
                            (10, 30 + i*25), cv2.FONT_HERSHEY_SIMPLEX,
                            0.7, (0, 255, 0), 2)

        cv2.imshow("ArUco Pose Estimation", frame)
        if cv2.waitKey(1) & 0xFF == 27:
            break

    cap.release()
    cv2.destroyAllWindows()

if __name__ == "__main__":
    main()
```

**실행 전 마커 출력**: 생성된 `aruco_marker_0.png`를 A4 용지에 인쇄하여 사용합니다.

---

## 5. 확인 및 테스트

```
1. color_tracking.py → 빨간색 물체 추적 (트랙바로 HSV 범위 조정)
2. feature_matching.py → reference.jpg와 실시간 영상 특징점 매칭
3. aruco_pose.py → ArUco 마커 출력물의 3D 포즈 추정 확인
```

> **다음 모듈**: Module 09 — NVIDIA Jetson AI / 딥러닝

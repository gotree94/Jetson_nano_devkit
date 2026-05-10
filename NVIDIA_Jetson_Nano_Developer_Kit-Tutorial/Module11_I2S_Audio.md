# Module 11 — I2S 디지털 오디오

> **소요 시간**: 3~4시간  
> **난이도**: ★★★☆☆  
> **준비물**: INMP441 I2S MEMS 마이크, MAX98357 I2S 앰프 + 3W 스피커

---

## 1. 개요

I2S(Inter-IC Sound)는 디지털 오디오 데이터 전송을 위한 직렬 버스 인터페이스입니다.  
Jetson Nano의 40-pin 헤더에는 I2S_4 인터페이스가 있습니다.

### 학습 목표
- I2S 프로토콜을 이해한다 (SCLK/LRCK/SDIN/SDOUT)
- INMP441 MEMS 마이크로 음성을 캡처할 수 있다
- MAX98357 I2S 앰프로 오디오를 출력할 수 있다
- FFT로 실시간 스펙트럼 분석을 수행할 수 있다

**핀 연결**:
| I2S 신호 | Jetson Pin | INMP441 | MAX98357 |
|----------|-----------|---------|----------|
| SCLK | 12 | SCK | BCLK |
| LRCK | 35 | WS | LRC |
| SDIN | 38 | DOUT | — |
| SDOUT | 40 | — | DIN |
| MCLK | 7 | — | — (일부 모델) |

---

## 2. 실습 코드

### 실습 11-1: INMP441 음성 녹음

**코드**: `i2s_record.py`
```python
#!/usr/bin/env python3
"""
I2S MEMS 마이크(INMP441) 음성 녹음
pydub + numpy로 WAV 저장
"""

import pyaudio
import wave
import numpy as np
import time

# I2S 설정
FORMAT = pyaudio.paInt16
CHANNELS = 1
RATE = 16000  # 16kHz
CHUNK = 1024
RECORD_SECONDS = 5

def record_audio(filename="recording.wav"):
    """I2S 마이크로 녹음"""
    p = pyaudio.PyAudio()

    # I2S 장치 찾기
    i2s_device = None
    for i in range(p.get_device_count()):
        info = p.get_device_info_by_index(i)
        if 'I2S' in info['name'] or 'tegra' in info['name'].lower():
            i2s_device = i
            break

    if i2s_device is None:
        print("I2S 장치를 찾을 수 없습니다.")
        print("사용 가능한 장치:")
        for i in range(p.get_device_count()):
            print(f"  {i}: {p.get_device_info_by_index(i)['name']}")
        return

    stream = p.open(format=FORMAT, channels=CHANNELS, rate=RATE,
                    input=True, input_device_index=i2s_device,
                    frames_per_buffer=CHUNK)

    print(f"녹음 시작 ({RECORD_SECONDS}초)...")
    frames = []

    for _ in range(0, int(RATE / CHUNK * RECORD_SECONDS)):
        data = stream.read(CHUNK)
        frames.append(data)

    print("녹음 완료")

    stream.stop_stream()
    stream.close()
    p.terminate()

    # WAV 저장
    wf = wave.open(filename, 'wb')
    wf.setnchannels(CHANNELS)
    wf.setsampwidth(p.get_sample_size(FORMAT))
    wf.setframerate(RATE)
    wf.writeframes(b''.join(frames))
    wf.close()
    print(f"저장: {filename}")

if __name__ == "__main__":
    record_audio()
```

### 실습 11-2: FFT 스펙트럼 분석

**코드**: `fft_analyzer.py`
```python
#!/usr/bin/env python3
"""
실시간 FFT 스펙트럼 분석기
"""

import pyaudio
import numpy as np
import time

FORMAT = pyaudio.paInt16
CHANNELS = 1
RATE = 44100
CHUNK = 4096

def main():
    p = pyaudio.PyAudio()

    # I2S 입력 장치 찾기
    dev_idx = None
    for i in range(p.get_device_count()):
        info = p.get_device_info_by_index(i)
        if info['maxInputChannels'] > 0:
            print(f"  {i}: {info['name']}")
            if 'I2S' in info['name']:
                dev_idx = i

    if dev_idx is None:
        print("I2S 입력 장치를 찾을 수 없습니다")
        return

    stream = p.open(format=FORMAT, channels=CHANNELS, rate=RATE,
                    input=True, input_device_index=dev_idx,
                    frames_per_buffer=CHUNK)

    print("FFT 스펙트럼 분석 시작 (Ctrl+C 종료)")
    print(f"{'주파수(Hz)':>12} {'진폭':>10} {'막대':<40}")

    try:
        while True:
            data = stream.read(CHUNK, exception_on_overflow=False)
            audio = np.frombuffer(data, dtype=np.int16).astype(np.float32)

            # FFT
            fft = np.fft.rfft(audio)
            magnitude = np.abs(fft) / CHUNK
            freqs = np.fft.rfftfreq(CHUNK, 1/RATE)

            # 주요 주파수 찾기
            peak_idx = np.argmax(magnitude[1:]) + 1
            peak_freq = freqs[peak_idx]
            peak_mag = magnitude[peak_idx]

            if peak_mag > 1.0:
                bar = '█' * int(peak_mag / 50) + '░' * (40 - int(peak_mag / 50))
                print(f"\r{peak_freq:8.1f}Hz  {peak_mag:8.1f}  {bar}", end='', flush=True)

            time.sleep(0.05)
    except KeyboardInterrupt:
        print("\n종료")
    finally:
        stream.close()
        p.terminate()

if __name__ == "__main__":
    main()
```

---

> **다음 모듈**: Module 12 — 네트워크 프로그래밍

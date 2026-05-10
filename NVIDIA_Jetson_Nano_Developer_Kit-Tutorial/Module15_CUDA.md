# Module 15 — CUDA 기초

> **소요 시간**: 4~5시간  
> **난이도**: ★★★★☆  
> **준비물**: Jetson Nano (JetPack CUDA 포함)

---

## 1. 개요

Jetson Nano의 128-core Maxwell GPU를 활용한 CUDA 병렬 프로그래밍 기초를 배웁니다.

### 학습 목표
- CUDA 아키텍처(Grid/Block/Thread/Warp)를 이해한다
- `nvcc`로 CUDA 커널을 컴파일할 수 있다
- Thread/Block 인덱싱을 이해하고 활용할 수 있다
- Shared Memory를 사용한 최적화를 이해한다
- OpenCV CUDA Interop을 사용할 수 있다

---

## 2. 이론

### 2.1 CUDA 메모리 계층

| 메모리 | 위치 | 접근 속도 | 범위 | 용도 |
|--------|------|----------|------|------|
| Global | DRAM | 느림 (400클럭) | 모든 스레드 | 대용량 데이터 |
| Shared | SRAM | 빠름 (1클럭) | 블록 내 | 데이터 재사용 |
| Local | DRAM (레지스터 스필) | 느림 | 스레드 | 배열 인덱싱 |
| Register | 레지스터 파일 | 즉시 | 스레드 | 로컬 변수 |

### 2.2 실행 모델

```
Grid
└── Block(0,0)      Block(1,0)      Block(2,0)
    ├── Thread(0,0)  ├── Thread(0,0)  ├── Thread(0,0)
    ├── Thread(1,0)  ├── Thread(1,0)  ├── Thread(1,0)
    ├── Thread(2,0)  ├── Thread(2,0)  ├── Thread(2,0)
    └── ...           └── ...           └── ...
```

---

## 3. 실습 코드

### 실습 15-1: 벡터 덧셈 (CUDA C)

**파일**: `vector_add.cu`
```cuda
#include <stdio.h>
#include <cuda_runtime.h>
#include <time.h>

// CUDA 커널
__global__ void vectorAdd(const float *A, const float *B, float *C, int N) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < N) {
        C[idx] = A[idx] + B[idx];
    }
}

// CPU 구현 (비교용)
void vectorAddCPU(const float *A, const float *B, float *C, int N) {
    for (int i = 0; i < N; i++) {
        C[i] = A[i] + B[i];
    }
}

int main() {
    int N = 1 << 20;  // 1M 요소
    size_t size = N * sizeof(float);

    // 호스트 메모리 할당
    float *h_A = (float*)malloc(size);
    float *h_B = (float*)malloc(size);
    float *h_C_cpu = (float*)malloc(size);
    float *h_C_gpu = (float*)malloc(size);

    // 초기화
    for (int i = 0; i < N; i++) {
        h_A[i] = (float)i;
        h_B[i] = (float)(N - i);
    }

    // GPU 메모리 할당
    float *d_A, *d_B, *d_C;
    cudaMalloc(&d_A, size);
    cudaMalloc(&d_B, size);
    cudaMalloc(&d_C, size);

    // CPU → GPU 복사
    cudaMemcpy(d_A, h_A, size, cudaMemcpyHostToDevice);
    cudaMemcpy(d_B, h_B, size, cudaMemcpyHostToDevice);

    // 커널 실행
    int threadsPerBlock = 256;
    int blocksPerGrid = (N + threadsPerBlock - 1) / threadsPerBlock;

    clock_t cpu_start = clock();
    vectorAddCPU(h_A, h_B, h_C_cpu, N);
    clock_t cpu_end = clock();
    double cpu_time = (double)(cpu_end - cpu_start) / CLOCKS_PER_SEC * 1000;

    cudaEvent_t start, stop;
    cudaEventCreate(&start);
    cudaEventCreate(&stop);

    cudaEventRecord(start);
    vectorAdd<<<blocksPerGrid, threadsPerBlock>>>(d_A, d_B, d_C, N);
    cudaEventRecord(stop);
    cudaEventSynchronize(stop);

    float gpu_time = 0;
    cudaEventElapsedTime(&gpu_time, start, stop);

    // GPU → CPU 복사
    cudaMemcpy(h_C_gpu, d_C, size, cudaMemcpyDeviceToHost);

    // 검증
    bool ok = true;
    for (int i = 0; i < N; i++) {
        if (h_C_cpu[i] != h_C_gpu[i]) {
            ok = false;
            break;
        }
    }

    printf("Vector Add (N=%d)\n", N);
    printf("  CPU 시간: %.2f ms\n", cpu_time);
    printf("  GPU 시간: %.2f ms\n", gpu_time);
    printf("  속도 향상: %.1fx\n", cpu_time / gpu_time);
    printf("  결과: %s\n", ok ? "일치 ✓" : "불일치 ✗");

    // 정리
    cudaFree(d_A); cudaFree(d_B); cudaFree(d_C);
    free(h_A); free(h_B); free(h_C_cpu); free(h_C_gpu);
    cudaEventDestroy(start); cudaEventDestroy(stop);

    return 0;
}
```

**컴파일 및 실행**:
```bash
nvcc -o vector_add vector_add.cu
./vector_add
```

### 실습 15-2: Sobel Edge Detection CUDA

**파일**: `sobel_cuda.cu`
```cuda
#include <stdio.h>
#include <cuda_runtime.h>
#include <math.h>

// Sobel X 커널
__global__ void sobelX(const unsigned char *input, unsigned char *output,
                       int width, int height) {
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    int y = blockIdx.y * blockDim.y + threadIdx.y;

    if (x < 1 || x >= width-1 || y < 1 || y >= height-1) {
        if (x < width && y < height) output[y * width + x] = 0;
        return;
    }

    float gx = 0;
    int idx = y * width + x;

    // Sobel X 커널
    gx = -1 * input[(y-1)*width + (x-1)] + 1 * input[(y-1)*width + (x+1)]
         -2 * input[y*width + (x-1)]      + 2 * input[y*width + (x+1)]
         -1 * input[(y+1)*width + (x-1)] + 1 * input[(y+1)*width + (x+1)];

    output[idx] = (unsigned char)min(max(abs(gx), 0.0f), 255.0f);
}

int main() {
    // 640x480 그레이스케일 이미지 가정
    int width = 640, height = 480;
    size_t size = width * height;

    // 호스트 메모리
    unsigned char *h_in = (unsigned char*)malloc(size);
    unsigned char *h_out = (unsigned char*)malloc(size);
    for (int i = 0; i < size; i++) h_in[i] = rand() % 256;

    // GPU 메모리
    unsigned char *d_in, *d_out;
    cudaMalloc(&d_in, size);
    cudaMalloc(&d_out, size);
    cudaMemcpy(d_in, h_in, size, cudaMemcpyHostToDevice);

    // 2D Grid/Block
    dim3 block(16, 16);
    dim3 grid((width + block.x - 1) / block.x,
              (height + block.y - 1) / block.y);

    sobelX<<<grid, block>>>(d_in, d_out, width, height);
    cudaMemcpy(h_out, d_out, size, cudaMemcpyDeviceToHost);

    printf("Sobel Edge Detection CUDA\n");
    printf("  해상도: %dx%d\n", width, height);
    printf("  Grid: %dx%d, Block: %dx%d\n", grid.x, grid.y, block.x, block.y);

    cudaFree(d_in); cudaFree(d_out);
    free(h_in); free(h_out);
    return 0;
}
```

---

> **다음 모듈**: Module 16 — 시스템 프로그래밍

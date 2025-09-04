---
layout: post
title: "[OpenGL] 01 - 개념 정리와 첫 번째 윈도우"
date: 2025-09-05
categories: [Graphics, OpenGL, Tutorial]
tags: [cpp, opengl, graphics, rendering]
---

## 📌 시리즈 정보

<div style="display: flex; gap: 10px; margin-bottom: 20px;">
<img src="https://img.shields.io/badge/플랫폼-macOS-black?style=flat-square" alt="macOS" />
<img src="https://img.shields.io/badge/IDE-CLion-00ACC1?style=flat-square" alt="CLion" />
<img src="https://img.shields.io/badge/OpenGL-3.3+-5586A4?style=flat-square" alt="OpenGL" />
<img src="https://img.shields.io/badge/C++-23-00599C?style=flat-square" alt="C++23" />
</div>

**시리즈 목표:** 볼류메트릭 클라우드 + 대기 산란 렌더링  
**현재 단계:** OpenGL 기초 - 윈도우 생성

## 📖 OpenGL이란?

### 1. 정의와 역할

**OpenGL (Open Graphics Library)**는 크로스 플랫폼 그래픽스 API입니다. CPU에서 "이런 도형을 그려줘"라고 명령하면, GPU가 이를 픽셀로 변환해 화면에 표시합니다.

### 2. CPU vs GPU

| 구분 | CPU | GPU |
|------|-----|-----|
| 처리 방식 | 순차적 처리 | 병렬 처리 |
| 코어 수 | 4-16개 | 수천 개 |
| 최적화 대상 | 복잡한 로직 | 단순 연산의 대량 처리 |
| 예시 | 게임 로직, AI | 픽셀 색상 계산, 행렬 연산 |

### 3. 그래픽스 파이프라인 개요

```
[3D 정점 데이터] 
    ↓ (Vertex Shader)
[변환된 정점] 
    ↓ (Primitive Assembly)
[도형(삼각형)] 
    ↓ (Rasterization)
[픽셀 후보들] 
    ↓ (Fragment Shader)
[최종 픽셀 색상]
    ↓ (Frame Buffer)
[화면 출력]
```

## 🎯 핵심 개념

### State Machine (상태 머신)
OpenGL은 거대한 상태 머신입니다. 한 번 설정한 상태는 명시적으로 변경하기 전까지 유지됩니다.

```cpp
glEnable(GL_DEPTH_TEST);  // 깊이 테스트 활성화
// ... 이후 모든 렌더링에 깊이 테스트 적용
glDisable(GL_DEPTH_TEST); // 명시적으로 비활성화
```

### Context (컨텍스트)
OpenGL의 모든 상태 정보를 담고 있는 객체입니다. 각 윈도우는 하나의 컨텍스트를 가집니다.

### Double Buffering (더블 버퍼링)
화면 깜빡임 없이 부드러운 애니메이션을 위한 기법:
- **Front Buffer**: 현재 화면에 표시 중인 버퍼
- **Back Buffer**: 다음 프레임을 그리는 버퍼

## 💡 필요한 라이브러리

### GLFW (Graphics Library Framework)
- 윈도우 생성 및 관리
- OpenGL 컨텍스트 생성
- 입력 이벤트 처리
- OS 독립적인 인터페이스 제공

### GLEW (OpenGL Extension Wrangler)
- OpenGL 확장 기능 로드
- 함수 포인터 관리
- OpenGL 1.1 이후 기능 사용 가능

### GLM (OpenGL Mathematics)
- 벡터, 행렬 연산
- OpenGL과 호환되는 수학 라이브러리
- GLSL과 유사한 문법

## 🛠 환경 설정

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.31)
project(Graphics_Study)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)   # 표준 C++만 사용'

if (NOT MSVC)
    # Wall: 대부분의 일반적인 경고 활성화(사용하지 않는 변수, 초기화 되지 않은 변수)
    # Wextra: Wall에 포함되지 않은 추가 경고(사용하지 않는 매개변수, 부호 비교)
    # Wpedantic: ISO C++표준 엄격 준수(비표준 확장 기능 사용 시 경고)
    add_compile_options(-Wall -Wextra -Wpedantic)
endif ()

if (APPLE)
    # libstdc++: GNU의 구현(구버전) C++11까지
    # libc++: LLVM/Clang의 구현 C++20/23 완벽 지원
    add_compile_options(-stdlib=libc++)
    add_link_options(-stdlib=libc++)
    add_compile_definitions(GL_SILENCE_DEPRECATION)
endif ()

find_package(OpenGL REQUIRED)
find_package(glfw3 REQUIRED)
find_package(GLEW REQUIRED)
find_package(glm REQUIRED)

# 실행 파일
add_executable(${PROJECT_NAME} main.cpp
        window.cpp
        window.hpp)

# 라이브러리 링크
target_link_libraries(${PROJECT_NAME}
        OpenGL::GL
        glfw
        GLEW::GLEW
        glm::glm
)
```

## ✅ 첫 번째 윈도우 생성

### 최소 코드 버전

```cpp
#include <iostream>
#include <GL/glew.h>    // OpenGL 확장 기능
#include <GLFW/glfw3.h> // 윈도우 관리

int main() {
    // 1. GLFW 초기화
    if (!glfwInit()) {
        std::cerr << "Failed to initialize GLFW" << std::endl;
        return -1;
    }

    // 2. 윈도우 생성
    GLFWwindow* window = glfwCreateWindow(
        800, 600,                    // 크기
        "My First OpenGL Window",    // 제목
        nullptr,                     // 모니터 (nullptr = 창모드)
        nullptr                      // 공유 컨텍스트
    );

    if (!window) {
        std::cerr << "Failed to create window" << std::endl;
        glfwTerminate();
        return -1;
    }

    // 3. OpenGL 컨텍스트 설정
    glfwMakeContextCurrent(window);

    // 4. GLEW 초기화
    if (glewInit() != GLEW_OK) {
        std::cerr << "Failed to initialize GLEW" << std::endl;
        return -1;
    }

    // 5. 메인 루프
    while (!glfwWindowShouldClose(window)) {
        // 이벤트 처리
        glfwPollEvents();
    }

    // 6. 정리
    glfwTerminate();
    return 0;
}
```

### 🔬 실행 및 테스트

실행 결과

- 800x600 윈도우 생성
- GLFW 초기화
- GLEW 초기화
- 메인 루프

![result](../assets/images/OpenGL/Empty%20Window.png)

### 💬 댓글

문제에 대한 질문이나 다른 풀이 방법이 있다면 아래 댓글로 공유해주세요!
---
layout: post
title: "[OpenGL] 03 - 하드코딩으로 첫 삼각형"
date: 2025-09-05
categories: [Graphics, OpenGL, Tutorial]
tags: [cpp, opengl, graphics, rendering, vbo, vao]
---

## 📌 시리즈 정보

<div style="display: flex; gap: 10px; margin-bottom: 20px;">

<img src="https://img.shields.io/badge/플랫폼-macOS-black?style=flat-square" alt="macOS" />

<img src="https://img.shields.io/badge/IDE-CLion-00ACC1?style=flat-square" alt="CLion" />

<img src="https://img.shields.io/badge/OpenGL-3.3+-5586A4?style=flat-square" alt="OpenGL" />

<img src="https://img.shields.io/badge/C++-23-00599C?style=flat-square" alt="C++23" />

</div>

이전 포스트: [02 Window 클래스 설계와 RAII 패턴](/blog/[OpenGL]%2002%20-%20Window%20클래스%20설계와%20RAII%20패턴.html)
현재 단계: 하드코딩으로 첫 삼각형 그리기
다음 포스트: Shader 클래스 설계

## 📖 이번 챕터의 목표

> "Make it work, make it right, make it fast" - Kent Beck

코드 구조는 잠시 제쳐두고, 주황색 삼각형을 화면에 띄우는 것이 목표입니다. 모든 코드를 `main.cpp`에 하드 코딩하여 OpenGL 렌더링 파이프라인의 핵심 흐름을 이해합니다.

## 🎯 시작하기 전에

현재 우리가 가진 것:

- ✅ Window 클래스 (GLFW/GLEW 초기화 완료)
- ✅ OpenGL 컨텍스트 생성
- ✅ 렌더링 루프 동작 중

이제 추가할 것:

- 🔺 셰이더 프로그램(GPU에서 실행될 코드)
- 🔺 정점 데이터 (삼각형 3개의 점)
- 🔺 VAO/VBO (GPU로 데이터 전송)

## 💡 핵심 개념 이해

**OpenGL** 렌더링 파이프라인 (간단 버전)

```
정점 데이터 (3개의 점)
    ↓ VBO에 업로드
Vertex Shader (위치 결정)
    ↓
Fragment Shader (색상 결정)
    ↓ 
화면에 삼각형!
```

**NDC** (Normalized Device Coordinates)

```
     (0, 1)
        ^
        |
(-1,0) --+--> (1, 0)
        |
        v
     (0, -1)
```

모든 좌표는 -1.0-1.0 범위 안에 있어야 화면에 표시됩니다.

## 📝 코드 구현

### **Step 1:** 셰이더 소스 코드 추가

`main.cpp` 상단, 익명 네임스페이스에 셰이더 코드를 하드코딩합니다:

```cpp

namespace {
    // Vertex Shader: 각 정점의 위치를 결정
    constexpr const char *VERTEX_SHADER_SOURCE = R"(
// OpenGL 3.3 Core Profile 사용
#version 330 core

// layout (location = 0): VAO에서 설정할 속성 위치
layout (location = 0) in vec3 aPos;     // 정점 위치 입력

void main() {
    // 입력받은 위치를 그대로 출력
    gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);
}
)";

    // FragmentShader: 각 픽셀의 색상을 결정
    constexpr const char *FRAGMENT_SHADER_SOURCE = R"(
#version 330 core
out vec4 FragColor;     // 출력 색상

void main() {
    // 모든 픽셀을 주황색으로
    FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);   // R, G, B, A
}
)";
}
```

### **Step 2: Application** 클래스 수정

OpenGL 객체들을 멤버 변수로 추가:

```cpp
class Application final {
private:
    std::unique_ptr<gl::Window> m_window;
    bool m_useAlternateColor = false;

    GLuint m_shaderProgram = 0; // 셰이더 프로그램 ID
    GLuint m_vao = 0; // Vertex Array Object
    GLuint m_vbo = 0; // Vertex Buffer Object
    bool m_wireframeMode = false; // 와이어프레임 모드 플래그
}
```

### **Step 3:** 셰이더 컴파일 함수

```cpp
class Application final {
private:
    GLuint compileShader(const GLenum shaderType, const char *source) {
        // 1. 셰이더 객체 생성
        const GLuint shader = glCreateShader(shaderType);

        // 2. 소스 코드 연결
        glShaderSource(shader, 1, &source, nullptr);

        // 3. 컴파일
        glCompileShader(shader);

        // 4. 컴파일 성공 여부 확인
        GLint success;
        glGetShaderiv(shader, GL_COMPILE_STATUS, &success);

        if (!success) {
            char infoLog[512];
            glGetShaderInfoLog(shader, 512, nullptr, infoLog);

            const char *shaderName = (shaderType == GL_VERTEX_SHADER)
                                         ? "Vertex"
                                         : "Fragment";
            std::println(stderr, "{} shader complication failed:\n{}", shaderName, infoLog);

            glDeleteShader(shader);
            return 0;
        }

        return shader;
    }

    bool setupShaders() {
        std::println("Setting up shaders...");

        // 1. 버텍스 셰이더 컴파일
        const GLuint vertexShader = compileShader(GL_VERTEX_SHADER, VERTEX_SHADER_SOURCE);
        if (!vertexShader) {
            return false;
        }

        // 2. 프래그먼트 셰이더 컴파일
        const GLuint fragmentShader = compileShader(GL_FRAGMENT_SHADER, FRAGMENT_SHADER_SOURCE);
        if (!fragmentShader) {
            glDeleteShader(vertexShader);
            return false;
        }

        // 3. 셰이더 프로그램 생성
        m_shaderProgram = glCreateProgram();

        // 4. 셰이더 연결
        glAttachShader(m_shaderProgram, vertexShader);
        glAttachShader(m_shaderProgram, fragmentShader);

        // 5. 프로그램 링킹
        glLinkProgram(m_shaderProgram);

        // 6. 링킹 성공 여부 확인
        GLint success;
        glGetProgramiv(m_shaderProgram, GL_LINK_STATUS, &success);

        if (!success) {
            char infoLog[512];
            glGetProgramInfoLog(m_shaderProgram, 512, nullptr, infoLog);
            std::println(stderr, "Shader program linking failed:\n{}", infoLog);

            glDeleteShader(vertexShader);
            glDeleteShader(fragmentShader);
            glDeleteProgram(m_shaderProgram);
            m_shaderProgram = 0;
            return false;
        }

        // 7. 개별 셰이더는 이제 필요 없음 (프로그램에 링크됨)
        glDeleteShader(vertexShader);
        glDeleteShader(fragmentShader);

        std::println("Shaders compiled and linked successfully!");
        return true;
    }
}
```

### **Step 4:** 삼각형 데이터 설정

```cpp
class Application final {
private:
bool setupTriangle() {
        std::println("Setting up triangle...");

        // 삼각형 3개 정점 (NDC 좌표계: -1~1)
        constexpr float vertices[] = {
            // x    y       z
            -0.5f, -0.5f, 0.0f, // 왼쪽아래
            0.5f, -0.5f, 0.0f, // 오른쪽 아래
            0.0f, 0.5f, 0.0f, // 위 중앙
        };

        // 1. VAO 생성 및 바인딩
        glGenVertexArrays(1, &m_vao);
        glBindVertexArray(m_vao);

        // 2. VBO 생성 및 바인딩
        glGenBuffers(1, &m_vbo);
        glBindBuffer(GL_ARRAY_BUFFER, m_vbo);

        // 3. 정점 데이터를 GPU 메모리로 복사
        glBufferData(GL_ARRAY_BUFFER,
                     sizeof(vertices),
                     vertices,
                     GL_STATIC_DRAW // Data가 자주 변경되지 않음
        );

        // 4. 버텍스 속성 포인터 설정
        glVertexAttribPointer(
            0, // 속성 번호 (layout location = 0)
            3, // 크기 (x, y, z = 3개)
            GL_FLOAT, // Data 타입
            GL_FALSE, // 정규화 여부
            3 * sizeof(float), // stride (다음 정점까지 간격)
            static_cast<void *>(nullptr) // 버퍼에서 시작 위치
        );

        // 5. 버텍스 속성 활성화
        glEnableVertexAttribArray(0);

        // 6. 바인딩 해제 (안전을 위해)
        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glBindVertexArray(0);

        std::println("Triangle VAO/VBO created successfully!");
        return true;
    }
}
```

### **Step 5:** Application 생성 및 정리

```cpp
class Application final {
public:
    [[nodiscard]] static std::expected<Application, gl::Error>
    create() noexcept {
       
       // ... Window 생성 코드 ...

        Application app{std::move(windowResult.value())};

        // OpenGL 객체 초기화
        if (!app.setupShaders()) {
            return std::unexpected(gl::Error{
                .code = gl::ErrorCode::ContextCreationFailed,
                .message = "Failed to compile/link shaders"
            });
        }

        if (!app.setupTriangle()) {
            return std::unexpected(gl::Error{
                .code = gl::ErrorCode::ContextCreationFailed,
                .message = "Failed to create triangle buffers"
            });
        }

        return app;
    }

    ~Application() {
        cleanup();
    }

private:
    void cleanup() const noexcept {
        // OpenGL 객체 정리 (RAII로 개선)
        if (m_shaderProgram) {
            glDeleteProgram(m_shaderProgram);
        }
        if (m_vao) {
            glDeleteVertexArrays(1, &m_vao);
        }
        if (m_vbo) {
            glDeleteBuffers(1, &m_vbo);
        }
    }
}
```

### **Step 6:** 렌더링 함수 수정

```cpp
void render() const {
        const Color &color = m_useAlternateColor
                                 ? Color{0.3f, 0.2f, 0.3f, 1.0f}
                                 : BACKGROUND_COLOR;

        glClearColor(color.r, color.g, color.b, color.a);
        glClear(GL_COLOR_BUFFER_BIT);

        // 1. 셰이더 프로그램 활성화
        glUseProgram(m_shaderProgram);

        // 2. VAO 바인딩
        glBindVertexArray(m_vao);

        // 3. 그리기 명령
        glDrawArrays(
            GL_TRIANGLES, // Primitive 타입
            0, // 시작 인덱스
            3 // 정점 개수
        );

        // 4. VAO 언바인딩
        glBindVertexArray(0);
    }
```

### **Step 7:** 

```cpp
void handleKeyPress(const int key) {
        switch (key) {
            // .. 기존 케이스들 ...

            case GLFW_KEY_W: {
                m_wireframeMode = !m_wireframeMode;
                glPolygonMode(GL_FRONT_AND_BACK, m_wireframeMode ? GL_LINE : GL_FILL);
                std::println("Wireframe mode: {}", m_wireframeMode ? "ON" : "OFF");
            }
            break;
            default:
                break;
        }
    }
```


### 🔬 실행 및 테스트

실행 결과

- 800x600 윈도우 생성
- ESC: 종료
- Space: 배경색 변경
- I: 정보 출력
- W: 와이어프레임 모드 토글
- FPS 콘솔 출력

<details>
<summary>전체 데모 영상 보기</summary>
<video width="100%" controls>
  <source src="/assets/videos/OpenGL/First Triangle.mp4" type="video/mp4">
</video>
</details>

## 🔑 핵심 개념 정리

### VAO와 VBO의 관계

- VBO: 실제 정점 데이터가 저장되는 GPU 메모리
- VAO: VBO의 데이터를 어떻게 해석할지 저장하는 "설정 파일"

### 셰이더 프로그램 실행 시점

1. `glUseProgram()`: 사용할 셰이더 프로그램 지정
2. `glDrawArrays()`: GPU가 각 정점에 대해 버텍스 셰이더 실행
3. 레스터라이제이션 후 각 픽셀에 대해 프래그먼트 셰이더 실행

## 📊 현재 코드의 문제점

- ❌ 셰이더 소스가 하드코딩됨
- ❌ OpenGL 객체 수동 관리
- ❌ 재사용성 없음

## 🚀 다음 단계

다음 포스트에서 다룰 내용:

- Shader 클래스 설계
- RAII로 자동 리소스 관리
- Uniform 변수 지원

## 💬 트러블 슈팅

### 문제: Window 클래스 복사/이동 생성자 누락

```cpp
 // 복사 방지
Window(const Window &other) = delete;
Window &operator=(const Window &other) = delete;

// 이동 생성자
Window(Window &&other) noexcept;
Window &operator=(Window &&other) noexcept;
```

```cpp
Window::Window(Window &&other) noexcept
        : m_window(std::move(other.m_window))
          , m_context(std::move(other.m_context))
          , m_resizeCallback(std::move(other.m_resizeCallback))
          , m_keyCallback(std::move(other.m_keyCallback)) {
        if (m_window) {
            glfwSetWindowUserPointer(m_window.get(), this);
        }
    }

    Window &Window::operator=(Window &&other) noexcept {
        if (this != &other) {
            m_window = std::move(other.m_window);
            m_context = std::move(other.m_context);
            m_resizeCallback = std::move(other.m_resizeCallback);
            m_keyCallback = std::move(other.m_keyCallback);

            if (m_window) {
                glfwSetWindowUserPointer(m_window.get(), this);
            }
        }
        return *this;
    }
```

### 문제: Application 클래스 복사/이동 생성자 누락

```cpp
// 복사 방지
    Application(const Application &) = delete;
    Application& operator=(const Application &) = delete;

    // 이동 생성자
    Application(Application &&other) noexcept
        : m_window(std::move(other.m_window))
          , m_useAlternateColor(other.m_useAlternateColor)
          , m_shaderProgram(std::exchange(other.m_shaderProgram, 0))
          , m_vao(std::exchange(other.m_vao, 0))
          , m_vbo(std::exchange(other.m_vbo, 0))
          , m_wireframeMode(other.m_wireframeMode) {
    }

    Application &operator=(Application &&other) noexcept {
        if (this != &other) {
            // 기존 리소스 정리
            cleanup();

            // 리소스 이동
            m_window = std::move(other.m_window);
            m_useAlternateColor = other.m_useAlternateColor;
            m_shaderProgram = std::exchange(other.m_shaderProgram, 0);
            m_vao = std::exchange(other.m_vao, 0);
            m_vbo = std::exchange(other.m_vbo, 0);
            m_wireframeMode = other.m_wireframeMode;
        }
        return *this;
    }
```

### 💬 댓글

문제에 대한 질문이나 다른 풀이 방법이 있다면 아래 댓글로 공유해주세요!
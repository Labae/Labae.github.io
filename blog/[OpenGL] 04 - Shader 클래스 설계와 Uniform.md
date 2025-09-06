---
layout: post
title: "[OpenGL] 04 - Shader 클래스 설계와 Uniform"
date: 2025-09-06
categories: [Graphics, OpenGL, Tutorial]
tags: [cpp, opengl, graphics, shader, uniform, glsl, bit-flags]
---

## 📌 시리즈 정보

<div style="display: flex; gap: 10px; margin-bottom: 20px;">
<img src="https://img.shields.io/badge/플랫폼-macOS-black?style=flat-square" alt="macOS" />
<img src="https://img.shields.io/badge/IDE-CLion-00ACC1?style=flat-square" alt="CLion" />
<img src="https://img.shields.io/badge/OpenGL-3.3+-5586A4?style=flat-square" alt="OpenGL" />
<img src="https://img.shields.io/badge/C++-23-00599C?style=flat-square" alt="C++23" />
</div>

- 이전 포스트: [03 - 하드코딩으로 첫 삼각형](/blog/[OpenGL]%2003%20-%20하드코딩으로%20첫%20삼각형.html)
- **현재 단계: Shader 클래스 설계와 Uniform 변수 활용**
- 다음 포스트: 05 변환(Transformation)과 GLM

## 📖 이번 챕터의 목표

> "Good code is its own best documentation" - Steve McConnell

Chapter 3에서 하드코딩한 셰이더 코드를 **재사용 가능한 Shader 클래스**로 리팩토링하고, **Uniform 변수**를 활용해 동적인 렌더링을 구현합니다.

## 🎯 리팩토링 전후 비교

### Before (Chapter 3)

```cpp
// main.cpp에 모든 것이 하드코딩
constexpr const char* VERTEX_SHADER_SOURCE = R"(...)";
constexpr const char* FRAGMENT_SHADER_SOURCE = R"(...)";
GLuint compileShader(...) { /* 100줄 */ }
bool setupShaders() { /* 50줄 */ }
```

### After (Chapter 4)

```cpp
// 깔끔한 인터페이스
auto shader = gl::Shader::create("shaders/basic.vert", "shaders/basic.frag");
shader->use();
shader->setVec3("u_Color", glm::vec3(1.0f, 0.5f, 0.2f));
```

## 💡 핵심 개념: Uniform 변수

### Uniform이란?

**Uniform 변수**는 CPU에서 GPU로 데이터를 전달하는 가장 기본적인 방법입니다.

```
CPU (C++ 코드)          GPU (GLSL Shader)
     |                        |
     |  glUniform3f() -----> uniform vec3 u_Color;
     |                        |
     v                        v
Application Logic        Shader Execution
```

### Uniform vs Attribute

| 구분 | Uniform | Attribute |
|------|---------|-----------|
| **용도** | 전역 값 (시간, 색상, 변환 행렬) | 정점별 데이터 (위치, 노멀, UV) |
| **변경 빈도** | 프레임마다 | 정점마다 |
| **적용 범위** | 모든 정점/픽셀에 동일 | 각 정점마다 다름 |
| **예시** | `uniform mat4 u_MVP;` | `in vec3 aPos;` |

## 🏗️ Shader 클래스 설계

### 설계 원칙

1. **RAII 패턴**: 생성자에서 컴파일/링킹, 소멸자에서 자동 정리
2. **에러 처리**: `std::expected`로 명확한 에러 전파
3. **타입 안전성**: GLM 타입 직접 지원
4. **캐싱 최적화**: Uniform 위치 캐싱 (선택적)

### 클래스 다이어그램

```
┌─────────────────────────────────┐
│         gl::Shader              │
├─────────────────────────────────┤
│ - m_programID: GLuint           │
├─────────────────────────────────┤
│ + create(vert, frag): expected  │
│ + use(): void                   │
│ + setInt(name, value): void     │
│ + setFloat(name, value): void   │
│ + setVec3(name, value): void    │
│ + setMat4(name, value): void    │
│ - compileShader(): expected     │
│ - linkProgram(): expected       │
└─────────────────────────────────┘
```

## 📝 구현 과정

### Step 1: 파일 구조 준비

```
Graphics_Study/
├── CMakeLists.txt
├── main.cpp
├── window.hpp/cpp
├── shader.hpp      # 새 파일
├── shader.cpp      # 새 파일
└── shaders/        # 새 폴더
    ├── basic.vert
    └── basic.frag
```

### Step 2: Shader 헤더 설계

```cpp
#ifndef SHADER_HPP
#define SHADER_HPP
#include <expected>
#include <memory>
#include <string>

#include <glm/matrix.hpp>
#include <GLFW/glfw3.h>

namespace gl {
    enum class ShaderErrorCode {
        FileReadFailed,
        VertexCompileFailed,
        FragmentCompileFailed,
        ProgramLinkFailed,
        UniformNotFound,
    };

    struct ShaderError {
        ShaderErrorCode code;
        std::string message;
    };

    class Shader final {
    public:
        [[nodiscard]] static std::expected<std::unique_ptr<Shader>, ShaderError>
        create(const std::string &vertexPath, const std::string &fragmentPath) noexcept;

        ~Shader() noexcept;

        // 복사 방지
        Shader(const Shader &) = delete;
        Shader &operator=(const Shader &) = delete;

        // 이동 생성자
        Shader(Shader &&other) noexcept;
        Shader &operator=(Shader &&other) noexcept;

        void use() const noexcept {
            glUseProgram(m_programID);
        }

        // Uniform 설정 메서드들
        void setBool(const std::string &name, bool value) const noexcept;
        void setInt(const std::string &name, int value) const noexcept;
        void setFloat(const std::string &name, float value) const noexcept;
        void setVec3(const std::string &name, const glm::vec3& value) const noexcept;
        void setVec4(const std::string &name, const glm::vec4& value) const noexcept;
        void setMat4(const std::string& name, const glm::mat4& value) const noexcept;

        [[nodiscard]] GLuint getProgramID() const noexcept {
            return m_programID;
        }

    private:
        explicit Shader(GLuint programID) noexcept;

        [[nodiscard]] static std::expected<std::string, ShaderError>
        readFile(const std::string &filePath) noexcept;

        [[nodiscard]] static std::expected<GLuint, ShaderError>
        compileShader(GLenum shaderType, const std::string& source) noexcept;

        [[nodiscard]] static std::expected<GLuint, ShaderError>
        linkProgram(GLuint vertexShader, GLuint fragmentShader) noexcept;

        [[nodiscard]] GLint getUniformLocation(const std::string &name) const noexcept;

    private:
        GLuint m_programID = 0;
    };
}

#endif //SHADER_HPP
```

### Step 3: Shader 구현

```cpp
#include "shader.hpp"

#include <cassert>
#include <fstream>
#include <sstream>
#include <glm/gtc/type_ptr.hpp>

namespace gl {
    std::expected<std::unique_ptr<Shader>, ShaderError>
    Shader::create(const std::string &vertexPath, const std::string &fragmentPath) noexcept {
        // 1. 파일에서 셰이더 소스 코드 읽기
        auto vertexResult = readFile(vertexPath);
        if (!vertexResult) {
            return std::unexpected(vertexResult.error());
        }

        auto fragmentResult = readFile(fragmentPath);
        if (!fragmentResult) {
            return std::unexpected(fragmentResult.error());
        }

        // 2. 셰이더 컴파일
        auto vertexShaderResult = compileShader(GL_VERTEX_SHADER, vertexResult.value());
        if (!vertexShaderResult) {
            return std::unexpected(vertexShaderResult.error());
        }

        auto fragmentShaderResult = compileShader(GL_FRAGMENT_SHADER, fragmentResult.value());
        if (!fragmentShaderResult) {
            // 정리
            glDeleteShader(vertexShaderResult.value());
            return std::unexpected(fragmentShaderResult.error());
        }

        // 3. 프로그램 링킹
        auto programResult = linkProgram(vertexShaderResult.value(), fragmentShaderResult.value());

        // 4. 개별 셰이더는 더 이상 필요 없음
        glDeleteShader(vertexShaderResult.value());
        glDeleteShader(fragmentShaderResult.value());

        if (!programResult) {
            return std::unexpected(programResult.error());
        }

        // 5. 셰이더 객체 생성
        return std::unique_ptr<Shader>(new Shader(programResult.value()));
    }

    Shader::~Shader() noexcept {
        if (m_programID) {
            glDeleteProgram(m_programID);
        }
    }

    Shader::Shader(Shader &&other) noexcept
    : m_programID(std::exchange(other.m_programID, 0)) {
    }

    Shader & Shader::operator=(Shader &&other) noexcept {
        if (this != &other) {
            if (m_programID) {
                glDeleteProgram(m_programID);
            }

            m_programID = std::exchange(other.m_programID, 0);
        }
        return *this;
    }

    void Shader::setBool(const std::string &name, bool value) const noexcept {
        assert(m_programID != 0 && "Invalid shader program ID");
        glUniform1i(getUniformLocation(name), value);
    }

    void Shader::setInt(const std::string &name, int value) const noexcept {
        assert(m_programID != 0 && "Invalid shader program ID");
        glUniform1i(getUniformLocation(name), value);
    }

    void Shader::setFloat(const std::string &name, float value) const noexcept {
        assert(m_programID != 0 && "Invalid shader program ID");
        glUniform1f(getUniformLocation(name), value);
    }

    void Shader::setVec3(const std::string &name, const glm::vec3 &value) const noexcept {
        assert(m_programID != 0 && "Invalid shader program ID");
        glUniform3fv(getUniformLocation(name), 1, glm::value_ptr(value));
    }

    void Shader::setVec4(const std::string &name, const glm::vec4 &value) const noexcept {
        assert(m_programID != 0 && "Invalid shader program ID");
        glUniform4fv(getUniformLocation(name), 1, glm::value_ptr(value));
    }

    void Shader::setMat4(const std::string &name, const glm::mat4 &value) const noexcept {
        assert(m_programID != 0 && "Invalid shader program ID");
        glUniformMatrix4fv(getUniformLocation(name), 1, GL_FALSE, glm::value_ptr(value));
    }

    Shader::Shader(const GLuint programID) noexcept
     : m_programID(programID) {
        assert(m_programID != 0 && "Invalid shader program ID");
    }

    std::expected<std::string, ShaderError> Shader::readFile(const std::string &filePath) noexcept {
        std::ifstream file(filePath);
        if (!file.is_open()) {
            return std::unexpected(ShaderError {
                .code = ShaderErrorCode::FileReadFailed,
                .message = "Failed to open file" + filePath
            });
        }

        std::stringstream buffer;
        buffer << file.rdbuf();

        // 파일이 비어있는지 확인
        std::string content = buffer.str();
        assert(!content.empty() && "Shader file is empty");

        return content;
    }

    std::expected<GLuint, ShaderError> Shader::compileShader(const GLenum shaderType,
        const std::string &source) noexcept {

        // 1. 셰이더 객체 생성
        const GLuint shader = glCreateShader(shaderType);
        assert(shader != 0 && "Failed to create a shader object");

        // 2. 소스 코드 연결
        const char* sourceCStr = source.c_str();
        glShaderSource(shader, 1, &sourceCStr, nullptr);

        // 3. 컴파일
        glCompileShader(shader);

        // 4. 컴파일 성공 여부 확인
        GLint success;
        glGetShaderiv(shader, GL_COMPILE_STATUS, &success);

        if (!success) {
            char infoLog[512];
            glGetShaderInfoLog(shader, 512, nullptr, infoLog);

            const char *shaderTypeName = (shaderType == GL_VERTEX_SHADER)
                                         ? "Vertex"
                                         : "Fragment";

            glDeleteShader(shader);

            return std::unexpected(ShaderError {
                .code = (shaderType == GL_VERTEX_SHADER)
                ? ShaderErrorCode::VertexCompileFailed
                : ShaderErrorCode::FragmentCompileFailed,
                .message =  std::string(shaderTypeName) + " shader compilation failed:\n" + infoLog
            });
        }

        return shader;
    }

    std::expected<GLuint, ShaderError> Shader::linkProgram(GLuint vertexShader, GLuint fragmentShader) noexcept {
        assert(vertexShader != 0 && "Invalid vertex shader ID");
        assert(fragmentShader != 0 && "Invalid fragment shader ID");


        // 1. 프로그램 객체 생성
        GLuint program = glCreateProgram();
        assert(program != 0 && "Failed to create a shader program");

        // 2. 셰이더 연결
        glAttachShader(program, vertexShader);
        glAttachShader(program, fragmentShader);

        // 3. 링킹
        glLinkProgram(program);

        // 4. 링킹 성공 여부 확인
        GLint success;
        glGetProgramiv(program, GL_LINK_STATUS, &success);
        if (!success) {
            char infoLog[512];
            glGetProgramInfoLog(program, 512, nullptr, infoLog);
            glDeleteProgram(program);

            return std::unexpected(ShaderError {
                .code = ShaderErrorCode::ProgramLinkFailed,
                .message = std::string("Shader program linking failed:\n") + infoLog
            });
        }
        return program;
    }

    GLint Shader::getUniformLocation(const std::string &name) const noexcept {
        assert(m_programID != 0 && "Invalid shader program ID");

        GLint location = glGetUniformLocation(m_programID, name.c_str());

#ifdef DEBUG
        if (location == -1) {
            std::println(stderr, "Warning: uniform '{}' not found in shader!", name);
        }
#endif

        return location;
    }
}
```

## 수정된 main.cpp

```cpp
#include <chrono>
#include <print>
#include <thread>
#include <cassert>

#include "window.hpp"
#include "shader.hpp"

namespace {
    constexpr std::int32_t WINDOW_WIDTH = 800;
    constexpr std::int32_t WINDOW_HEIGHT = 600;
    constexpr std::string_view WINDOW_TITLE = "OpenGL Window";

    struct Color {
        float r, g, b, a;

        constexpr Color(
            const float r,
            const float g,
            const float b,
            const float a)
            : r(r), g(g), b(b), a(a) {
        }

        [[nodiscard]] glm::vec3 toVec3() const noexcept {
            return {r, g, b};
        }
    };

    constexpr Color BACKGROUND_COLOR{0.2f, 0.3f, 0.3f, 1.0f};

    constexpr Color ORANGE{1.0f, 0.5f, 0.2f, 1.0f};
    constexpr Color CYAN{0.2f, 0.8f, 0.8f, 1.0f};
    constexpr Color PURPLE{0.8f, 0.2f, 0.8f, 1.0f};
    constexpr Color GREEN{0.2f, 0.8f, 0.2f, 1.0f};

    enum ColorEffect : std::uint8_t {
        None = 0,
        Pulse = 1 << 0, // 0001
        Rainbow = 1 << 1, // 0010
        Gradient = 1 << 2, // 0100
    };
}

class Application final {
public:
    [[nodiscard]] static std::expected<Application, gl::WindowError>
    create() noexcept {
        const gl::WindowProperties properties{
            .width = WINDOW_WIDTH,
            .height = WINDOW_HEIGHT,
            .title = std::string(WINDOW_TITLE),
            .glMajorVersion = 3,
            .glMinorVersion = 3,
            .useCore = true,
            .resizable = true
        };

        auto windowResult = gl::Window::create(properties);
        if (!windowResult) {
            return std::unexpected(windowResult.error());
        }

        // Application 객체 생성
        Application app{std::move(windowResult.value())};

        // OpenGL 객체 초기화
        if (!app.setupShaders()) {
            return std::unexpected(gl::WindowError{
                .code = gl::WindowErrorCode::ContextCreationFailed,
                .message = "Failed to compile/link shaders"
            });
        }

        if (!app.setupTriangle()) {
            return std::unexpected(gl::WindowError{
                .code = gl::WindowErrorCode::ContextCreationFailed,
                .message = "Failed to create triangle buffers"
            });
        }

        return app;
    }

    ~Application() {
        cleanup();
    }

    // 복사 방지
    Application(const Application &) = delete;

    Application &operator=(const Application &) = delete;

    // 이동 생성자
    Application(Application &&other) noexcept
        : m_window(std::move(other.m_window))
          , m_shader(std::move(other.m_shader))
          , m_useAlternateColor(other.m_useAlternateColor)
          , m_vao(std::exchange(other.m_vao, 0))
          , m_vbo(std::exchange(other.m_vbo, 0))
          , m_wireframeMode(other.m_wireframeMode)
          , m_triangleColor(other.m_triangleColor)
          , m_elapsedTime(other.m_elapsedTime)
          , m_effects(other.m_effects) {
    }

    Application &operator=(Application &&other) noexcept {
        if (this != &other) {
            // 기존 리소스 정리
            cleanup();

            // 리소스 이동
            m_window = std::move(other.m_window);
            m_shader = std::move(other.m_shader);
            m_useAlternateColor = other.m_useAlternateColor;
            m_vao = std::exchange(other.m_vao, 0);
            m_vbo = std::exchange(other.m_vbo, 0);
            m_wireframeMode = other.m_wireframeMode;
            m_triangleColor = other.m_triangleColor;
            m_elapsedTime = other.m_elapsedTime;
            m_effects = other.m_effects;
        }
        return *this;
    }

    void run() {
        setupCallbacks();
        printInfo();

        // FPS 계산을 위한 변수
        auto lastTime = std::chrono::steady_clock::now();
        const auto startTime = std::chrono::steady_clock::now();
        std::size_t frameCount = 0;

        while (!m_window->shouldClose()) {
            // FPS 계산
            auto currentTime = std::chrono::steady_clock::now();
            const auto totalElapsedTime = std::chrono::duration<float>(currentTime - startTime);
            m_elapsedTime = totalElapsedTime.count();

            ++frameCount;

            const auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(currentTime - lastTime);
            if (elapsed.count() >= 1000) {
                const double fps = frameCount * 1000.0 / elapsed.count();
                std::println("FPS: {:1f}", fps);

                frameCount = 0;
                lastTime = currentTime;
            }

            // 렌더링
            render();

            // 이벤트 처리
            m_window->swapBuffers();
            m_window->pollEvents();

            // CPU 사용률 제한
            std::this_thread::sleep_for(std::chrono::milliseconds(1));
        }
    }

private:
    explicit Application(std::unique_ptr<gl::Window> window) noexcept
        : m_window(std::move(window)) {
    }

    void cleanup() const noexcept {
        // OpenGL 객체 정리 (RAII로 개선)
        if (m_vao) {
            glDeleteVertexArrays(1, &m_vao);
        }
        if (m_vbo) {
            glDeleteBuffers(1, &m_vbo);
        }
    }

    void setupCallbacks() {
        m_window->setKeyCallback([this](const int key, const int action, int /* mods*/) {
            if (action == GLFW_PRESS) {
                handleKeyPress(key);
            }
        });
    }

    void handleKeyPress(const int key) {
        switch (key) {
            case GLFW_KEY_ESCAPE: {
                std::println("Escape pressed - closing window");
                glfwSetWindowShouldClose(
                    glfwGetCurrentContext(), GLFW_TRUE);
            }
            break;

            case GLFW_KEY_SPACE: {
                m_useAlternateColor = !m_useAlternateColor;
                std::println("Color mode: {}", m_useAlternateColor ? "alternate" : "default");
            }
            break;

            case GLFW_KEY_1: {
                m_triangleColor = ORANGE;
                std::println("Color: Orange");
            }
            break;

            case GLFW_KEY_2: {
                m_triangleColor = CYAN;
                std::println("Color: Cyan");
            }
            break;

            case GLFW_KEY_3: {
                m_triangleColor = PURPLE;
                std::println("Color: Purple");
            }
            break;

            case GLFW_KEY_4: {
                m_triangleColor = GREEN;
                std::println("Color: Green");
            }
            break;

            case GLFW_KEY_P: {
                toggleEffect(ColorEffect::Pulse);
                std::println("Pulse: {}", hasEffect(ColorEffect::Pulse) ? "ON" : "OFF");
            }
            break;

            case GLFW_KEY_R: {
                toggleEffect(ColorEffect::Rainbow);
                std::println("Rainbow: {}", hasEffect(ColorEffect::Rainbow) ? "ON" : "OFF");
            }
            break;

            case GLFW_KEY_G: {
                toggleEffect(ColorEffect::Gradient);
                std::println("Gradient: {}", hasEffect(ColorEffect::Gradient) ? "ON" : "OFF");
            }
            break;

            case GLFW_KEY_I: {
                printInfo();
            }
            break;

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

    void render() const {
        const Color &color = m_useAlternateColor
                                 ? Color{0.3f, 0.2f, 0.3f, 1.0f}
                                 : BACKGROUND_COLOR;

        glClearColor(color.r, color.g, color.b, color.a);
        glClear(GL_COLOR_BUFFER_BIT);

        // 1. 셰이더 프로그램 활성화
        m_shader->use();

        m_shader->setVec3("u_Color", m_triangleColor.toVec3());
        m_shader->setFloat("u_Time", m_elapsedTime);
        m_shader->setInt("u_Effects", m_effects);

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

    bool setupShaders() {
        std::println("Setting up shaders...");

        auto shaderResult = gl::Shader::create(
            "shaders/basic.vert",
            "shaders/basic.frag");

        if (!shaderResult) {
            std::println(stderr, "Failed to create shader: {}", shaderResult.error().message);
            return false;
        }

        m_shader = std::move(shaderResult.value());

        std::println("Shaders compiled and linked successfully!");
        std::println("Shader Program ID: {}", m_shader->getProgramID());
        return true;
    }

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

    void toggleEffect(ColorEffect effect) noexcept {
        m_effects ^= effect;
    }

    [[nodiscard]] bool hasEffect(ColorEffect effect) const noexcept {
        return (m_effects & effect) != 0;
    }

    void printInfo() const {
        std::println("=== Start OpenGL Information ===");
        std::println("{}", m_window->getOpenGLInfo());

        const auto [width, height] = m_window->getWindowSize();
        std::println("Window Size: {}x{}", width, height);

        const auto [fbwidth, fbheight] = m_window->getFramebufferSize();
        std::println("Framebuffer Size: {}x{}", fbwidth, fbheight);

        if (m_shader) {
            std::println("Shader Program ID: {}", m_shader->getProgramID());
        }
        std::println("=== End OpenGL Information ===\n");
    }

private:
    std::unique_ptr<gl::Window> m_window;
    std::unique_ptr<gl::Shader> m_shader;
    bool m_useAlternateColor = false;

    GLuint m_vao = 0; // Vertex Array Object
    GLuint m_vbo = 0; // Vertex Buffer Object
    bool m_wireframeMode = false; // 와이어프레임 모드 플래그

    Color m_triangleColor = ORANGE;
    float m_elapsedTime = 0.0f;
    std::uint8_t m_effects = ColorEffect::None;
};

int main() {
    auto appResult = Application::create();
    if (!appResult) {
        std::println(stderr, "Error: {}", appResult.error().message);
        std::abort();
    }
    appResult.value().run();
    return EXIT_SUCCESS;
}
```

## 🎨 Uniform 활용: 동적 색상 효과

### Basic Fragment Shader

```glsl
#version 330 core
out vec4 FragColor;

uniform vec3 u_Color;    // CPU에서 전달받는 색상
uniform float u_Time;    // 애니메이션용 시간

void main() {
    FragColor = vec4(u_Color, 1.0);
}
```

### 고급: Bit Flag 효과 시스템

셰이더에서 여러 효과를 조합할 수 있는 시스템을 구현했습니다:

```glsl
#version 330 core
out vec4 FragColor;

uniform vec3 u_Color;
uniform float u_Time;
uniform int u_Effects;  // Bit flags: 1=Pulse, 2=Rainbow, 4=Gradient

void main() {
    vec3 color = u_Color;
    
    // Rainbow 효과 (bit 1)
    if ((u_Effects & 2) != 0) {
        float r = sin(u_Time * 2.0) * 0.5 + 0.5;
        float g = sin(u_Time * 2.0 + 2.094395) * 0.5 + 0.5;  // 2π/3
        float b = sin(u_Time * 2.0 + 4.188790) * 0.5 + 0.5;  // 4π/3
        color = vec3(r, g, b);
    }
    
    // Pulse 효과 (bit 0)
    if ((u_Effects & 1) != 0) {
        float factor = sin(u_Time * 3.0) * 0.3 + 0.7;
        color = color * factor;
    }
    
    // Gradient 효과 (bit 2)
    if ((u_Effects & 4) != 0) {
        float gradient = gl_FragCoord.x / 800.0;
        color = color * (gradient * 0.7 + 0.3);
    }
    
    FragColor = vec4(color, 1.0);
}
```

### 효과 관리

```cpp
enum ColorEffect : std::uint8_t {
    None     = 0,
    Pulse    = 1 << 0,  // 0001
    Rainbow  = 1 << 1,  // 0010
    Gradient = 1 << 2   // 0100
};

class Application {
private:
    std::uint8_t m_effects = ColorEffect::None;
    
    void toggleEffect(ColorEffect effect) noexcept {
        m_effects ^= effect;  // XOR로 토글
    }
    
    void render() const {
        m_shader->use();
        m_shader->setVec3("u_Color", m_triangleColor.toVec3());
        m_shader->setFloat("u_Time", m_elapsedTime);
        m_shader->setInt("u_Effects", m_effects);  // 비트 플래그 전달
        
        glBindVertexArray(m_vao);
        glDrawArrays(GL_TRIANGLES, 0, 3);
    }
};
```

## 🔬 실행 결과

<video width="100%" controls>
  <source src="/assets/videos/OpenGL/shader-uniform.mp4" type="video/mp4">
  Shader 효과 조합 데모
</video>

### 조작법

- **1-4**: 색상 변경 (Orange/Cyan/Purple/Green)
- **P**: Pulse 효과 토글
- **R**: Rainbow 효과 토글
- **G**: Gradient 효과 토글
- **W**: 와이어프레임 모드

## 📊 성능 고려사항

### Uniform 위치 캐싱

```cpp
// 매번 조회 (느림)
for (int i = 0; i < 1000; ++i) {
    GLint location = glGetUniformLocation(program, "u_Color");  // 매번 문자열 검색
    glUniform3f(location, r, g, b);
}

// 캐싱 (빠름)
GLint colorLocation = glGetUniformLocation(program, "u_Color");  // 한 번만
for (int i = 0; i < 1000; ++i) {
    glUniform3f(colorLocation, r, g, b);
}
```

## 🎓 핵심 정리

### 이번 챕터에서 배운 것

1. **Shader 클래스 설계**
   - RAII 패턴으로 자동 리소스 관리
   - 파일 기반 셰이더 로딩
   - 타입 안전한 Uniform 인터페이스

2. **Uniform 변수 활용**
   - CPU → GPU 데이터 전달
   - 동적 색상 및 애니메이션
   - Bit flag를 활용한 효과 조합

3. **Assert vs Exception**
   - 프로그래머 실수: assert (즉시 크래시)
   - 런타임 에러: std::expected (복구 가능)

### 설계 패턴

```cpp
// Factory Pattern + RAII
auto shader = Shader::create("vertex.glsl", "fragment.glsl");

// Bit Flag Pattern
effects |= Effect::Rainbow;   // 효과 추가
effects &= ~Effect::Rainbow;  // 효과 제거
effects ^= Effect::Rainbow;   // 효과 토글
```

## 💬 트러블슈팅

### 문제 1: Uniform이 적용되지 않음

**체크리스트:**

1. 셰이더 사용 전 `shader->use()` 호출 확인
2. Uniform 이름 오타 확인
3. GLSL에서 uniform이 실제로 사용되는지 확인 (미사용 시 최적화로 제거됨)

### 문제 2: 셰이더 파일을 못 찾음

CMakeLists.txt에 복사 명령 추가:

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

# 셰이더 파일들을 변수에 저장
file(GLOB SHADER_FILES "${CMAKE_SOURCE_DIR}/shaders/*.vert" "${CMAKE_SOURCE_DIR}/shaders/*.frag")

# 실행 파일
add_executable(${PROJECT_NAME} main.cpp
        window.cpp
        window.hpp
        ${SHADER_FILES}
        shader.cpp
        shader.hpp
)

# 셰이더 파일을 실행 파일 위치로 복사
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${CMAKE_SOURCE_DIR}/shaders
    $<TARGET_FILE_DIR:${PROJECT_NAME}>/shaders
)

# 라이브러리 링크
target_link_libraries(${PROJECT_NAME}
        OpenGL::GL
        glfw
        GLEW::GLEW
        glm::glm
)
```

## 🚀 다음 단계

다음 포스트에서 다룰 내용:
- **변환(Transformation)**: 이동, 회전, 크기 조절
- **GLM 라이브러리**: 행렬 연산
- **3D 공간**: Model, View, Projection 행렬
- **첫 3D 큐브**: 정점 인덱스와 깊이 테스트

## 📚 참고 자료

- [OpenGL Shading Language Specification](https://www.khronos.org/opengl/wiki/OpenGL_Shading_Language)
- [Learn OpenGL - Shaders](https://learnopengl.com/Getting-started/Shaders)
- [GLM Documentation](https://glm.g-truc.net/0.9.9/index.html)

---


### 💬 댓글

문제에 대한 질문이나 다른 풀이 방법이 있다면 아래 댓글로 공유해주세요!
---
layout: post
title: "[OpenGL] 02 - Window 클래스 설계와 RAII 패턴"
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

이전 포스트: [01-개념 정리와 첫 번째 윈도우]([OpenGL]%2001%20-%20개념%20정리와%20첫%20번째%20윈도우.md)
현재 단계: Window 클래스 설계 - RAII 패턴 적용
다음 포스트: 첫 번째 삼각형 그리기

## 📖 왜 클래스로 만들어야 할까?

이전 포스트에서 작성한 최소 코드는 잘 동작하지만 몇 가지 문제가 있습니다.

### 문제점들

1. 리소스 관리: 예외 발생 시 [glfwTerminate()](https://www.glfw.org/docs/latest/group__init.html#gaaae48c0a18607ea4a4ba951d939f0901) 호출 보장 못함.
2. 코드 재사용성: 윈도우 생성 로직을 다시 작성해야함.
3. 확장성: 여러 윈도우 관리가 어려움
4. 가독성: main 함수가 너무 복잡해짐

### 해결책: RAII + 클래스 설계

[RAII(Resource Acquisition Is Initialization)](https://en.cppreference.com/w/cpp/language/raii.html)

- 생성자에서 리소스 획득
- 소멸자에서 리소스 해제
- 예외 안전성 보장

## 🎯 설계 목표

- ✅ **Modern C++23** 활용 (std::format, std::expected)
- ✅ **RAII** 패턴으로 자동 리소스 관리
- ✅ 예외 처리를 예외가 아닌 값으로

## 📁 파일 구조 변경

```
Graphics_Study/
|- CMakeList.txt
|- main.cpp         # Application Entrypoint
|- window.hpp       # Window 클래스 선언 (새로 추가)
|- window.cpp       # Window 클래스 구현 (새로 추가)
```

## 💡 윈도우 클래스 설계

**window.hpp** - 인터페이스

```cpp
#ifndef WINDOW_HPP
#define WINDOW_HPP

// OpenGL 헤더는 순서가 중요! GLEW가 먼저
#include <GL/glew.h>
#include <GLFW/glfw3.h>

#include <expected>
#include <functional>
#include <memory>
#include <string>

namespace gl {
    // 에러 타입 정의
    enum class ErrorCode {
        GLFWInitFailed,
        WindowCreationFailed,
        ContextCreationFailed,
        GLEWInitFailed,
    };

    struct Error {
        ErrorCode code;
        std::string message;
    };

    using WindowHandle = GLFWwindow *;
    using WindowDeleter = decltype([](WindowHandle window) {
        if (window) {
            glfwDestroyWindow(window);
        }
    });
    using WindowPtr = std::unique_ptr<GLFWwindow, WindowDeleter>;
    using ResizeCallback = std::function<void(int, int)>;
    using KeyCallback = std::function<void(int, int, int)>;

    // 윈도우 속성
    namespace defaults {
        constexpr std::int32_t WIDTH = 800;
        constexpr std::int32_t HEIGHT = 600;
        constexpr const char *TITLE = "OpenGL Window";
        constexpr std::int32_t MAJOR_VERSION = 3;
        constexpr std::int32_t MINOR_VERSION = 3;
        constexpr bool USE_CORE_PROFILE = true;
        constexpr bool RESIZABLE = true;
    }

    struct WindowProperties {
        std::int32_t width{defaults::WIDTH};
        std::int32_t height{defaults::HEIGHT};
        std::string title{defaults::TITLE};
        std::int32_t glMajorVersion{defaults::MAJOR_VERSION};
        std::int32_t glMinorVersion{defaults::MINOR_VERSION};
        bool useCore{defaults::USE_CORE_PROFILE};
        bool resizable{defaults::RESIZABLE};
    };

    // GLFW 초기화 RAII 래퍼
    class GLFWContext final {
    public:
        [[nodiscard]] static std::expected<std::unique_ptr<GLFWContext>, Error>
        create() noexcept;

        ~GLFWContext() noexcept;

        // 복사 방지
        GLFWContext(const GLFWContext &) = delete;

        GLFWContext &operator=(const GLFWContext &) = delete;


        // 이동 생성자/대입
        GLFWContext(GLFWContext &&) noexcept = default;

        GLFWContext &operator=(GLFWContext &&) noexcept = default;

    private:
        GLFWContext() = default;
    };

    // Window 클래스
    class Window final {
    public:
        [[nodiscard]] static std::expected<std::unique_ptr<Window>, Error>
        create(const WindowProperties &properties = {}) noexcept;

        ~Window() = default;

        // 기본 작업
        void pollEvents() const noexcept {
            glfwPollEvents();
        }

        void swapBuffers() const noexcept {
            glfwSwapBuffers(m_window.get());
        }

        [[nodiscard]] bool shouldClose() const noexcept {
            return glfwWindowShouldClose(m_window.get());
        }

        void setResizeCallback(const ResizeCallback &callback) noexcept {
            m_resizeCallback = std::move(callback);
        }

        void setKeyCallback(const KeyCallback &callback) noexcept {
            m_keyCallback = std::move(callback);
        }

        // 속성 접근
        [[nodiscard]] std::pair<std::int32_t, std::int32_t> getWindowSize() const noexcept;

        [[nodiscard]] std::pair<std::int32_t, std::int32_t> getFramebufferSize() const noexcept;

        // 디버그
        [[nodiscard]] std::string getOpenGLInfo() noexcept;

    private:
        Window(WindowPtr window, std::unique_ptr<GLFWContext> context) noexcept;

        static void framebufferSizeCallbackDispatcher(GLFWwindow *window, const int width, const int height) noexcept;

        static void keyCallbackDispatcher(GLFWwindow *window, const int key, const int scancode, const int action,
                                          const int mods) noexcept;

        WindowPtr m_window;
        std::unique_ptr<GLFWContext> m_context;
        ResizeCallback m_resizeCallback;
        KeyCallback m_keyCallback;
    };
}

#endif //WINDOW_HPP
```

**window.cpp** - 구현

```cpp
#include "window.hpp"
#include <format>

namespace gl {
    std::expected<std::unique_ptr<GLFWContext>, Error>
    GLFWContext::create() noexcept {
        auto context = std::unique_ptr<GLFWContext>(new GLFWContext());
        if (glfwInit() != GLFW_TRUE) {
            return std::unexpected(Error{
                .code = ErrorCode::GLFWInitFailed,
                .message = "Failed to initialize GLFW"
            });
        }

        return context;
    }

    GLFWContext::~GLFWContext() noexcept {
        glfwTerminate();
    }

    std::expected<std::unique_ptr<Window>, Error>
    Window::create(const WindowProperties &properties) noexcept {
        // GLFW Context 생성
        auto contextResult = GLFWContext::create();
        if (!contextResult) {
            return std::unexpected(contextResult.error());
        }

        // OpenGL 버전 힌트 설정
        glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, properties.glMajorVersion);
        glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, properties.glMinorVersion);

        if (properties.useCore) {
            // Core Profile: Deprecated기능 제거, 더 빠르고 효율적
            glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
        }

#ifdef __APPLE__
        // Forward Compatibility: 미래 버전과의 호환성, mac에서 CoreProfile 사용시 필수
        glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
#endif

        glfwWindowHint(GLFW_RESIZABLE, properties.resizable ? GLFW_TRUE : GLFW_FALSE);

        const WindowHandle rawWindow = glfwCreateWindow(properties.width,
                                                  properties.height, properties.title.c_str(),
                                                  nullptr, nullptr);
        if (!rawWindow) {
            return std::unexpected(Error{
                .code = ErrorCode::WindowCreationFailed,
                .message = std::format("Failed to create {}x{} window",
                    properties.width, properties.height),
            });
        }

        // Context 설정
        glfwMakeContextCurrent(rawWindow);

        // GLEW 초기화
        if (const GLenum err = glewInit(); err != GLEW_OK) {
            glfwDestroyWindow(rawWindow);
            return std::unexpected(Error{
                .code = ErrorCode::GLEWInitFailed,
                .message = std::format("Failed to initialize GLEW: {}",
                    reinterpret_cast<const char*>(glewGetErrorString(err)))
            });
        }

        // 윈도우 객체 생성
        auto window = std::unique_ptr<Window>(new Window(WindowPtr(rawWindow),
                                               std::move(contextResult.value())));

        // 콜백 설정
        glfwSetWindowUserPointer(rawWindow, window.get());
        glfwSetFramebufferSizeCallback(rawWindow, framebufferSizeCallbackDispatcher);
        glfwSetKeyCallback(rawWindow, keyCallbackDispatcher);

        // 초기 뷰포트 설정
        int fbWidth, fbHeight;
        glfwGetWindowSize(rawWindow, &fbWidth, &fbHeight);
        glViewport(0, 0, fbWidth, fbHeight);

        return window;
    }

    std::pair<std::int32_t, std::int32_t> Window::getWindowSize() const noexcept {
        std::int32_t width, height;
        glfwGetWindowSize(m_window.get(), &width, &height);
        return {width, height};
    }

    std::pair<std::int32_t, std::int32_t> Window::getFramebufferSize() const noexcept {
        std::int32_t width, height;
        glfwGetFramebufferSize(m_window.get(), &width, &height);
        return {width, height};
    }

    std::string Window::getOpenGLInfo() noexcept {
        auto safeGetString = [](const GLenum name) -> const char* {
          const auto* str = glGetString(name);
            return str ? reinterpret_cast<const char*>(str) : "Unknown";
        };

        return std::format(
            "OpenGL Version: {}\n"
            "GLSL Version: {}\n"
            "Vendor: {}\n"
            "Renderer: {}\n",
            safeGetString(GL_VERSION),
            safeGetString(GL_SHADING_LANGUAGE_VERSION),
            safeGetString(GL_VENDOR),
            safeGetString(GL_RENDERER)
        );
    }

    Window::Window(WindowPtr window, std::unique_ptr<GLFWContext> context) noexcept
        : m_window(std::move(window)), m_context(std::move(context)) {
    }

    void Window::framebufferSizeCallbackDispatcher(GLFWwindow *window, const int width, const int height) noexcept {
        if (const auto *self = static_cast<Window *>(glfwGetWindowUserPointer(window))) {
            glViewport(0, 0, width, height);
            if (self->m_resizeCallback) {
                self->m_resizeCallback(width, height);
            }
        }
    }

    void Window::keyCallbackDispatcher(GLFWwindow *window,
                                       const int key, const int scancode, const int action, const int mods) noexcept {
        if (const auto *self = static_cast<Window *>(glfwGetWindowUserPointer(window))) {
            if (self->m_keyCallback) {
                self->m_keyCallback(key, action, mods);
            }
        }
    }
}
```

**main.cpp** - 사용 예제

```cpp
#include <chrono>
#include <print>
#include <thread>

#include "window.hpp"

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
    };

    constexpr Color BACKGROUND_COLOR{0.2f, 0.3f, 0.3f, 1.0f};
}

class Application final {
public:
    [[nodiscard]] static std::expected<Application, gl::Error>
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

        return Application{std::move(windowResult.value())};
    }

    void run() {
        setupCallbacks();
        printInfo();

        // FPS 계산을 위한 변수
        auto lastTime = std::chrono::steady_clock::now();
        std::size_t frameCount = 0;

        while (!m_window->shouldClose()) {

            // FPS 계산
            auto currentTime = std::chrono::steady_clock::now();
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

            case GLFW_KEY_I: {
                printInfo();
            }
            break;
            default:
                break;
        }
    }

    void render() const {
        const Color& color = m_useAlternateColor
        ? Color{0.3f, 0.2f, 0.3f, 1.0f}
        : BACKGROUND_COLOR;

        glClearColor(color.r, color.g, color.b, color.a);
        glClear(GL_COLOR_BUFFER_BIT);
    }

    void printInfo() const {
        std::println("=== Start OpenGL Information ===");
        std::println("{}", m_window->getOpenGLInfo());

        const auto [width, height] = m_window->getWindowSize();
        std::println("Window Size: {}x{}", width, height);

        const auto [fbwidth, fbheight] = m_window->getFramebufferSize();
        std::println("Framebuffer Size: {}x{}", fbwidth, fbheight);
        std::println("=== End OpenGL Information ===\n");
    }

    std::unique_ptr<gl::Window> m_window;
    bool m_useAlternateColor = false;
};

int main() {
    try {
        auto appResult = Application::create();
        if (!appResult) {
            std::println(stderr, "Error: {}", appResult.error().message);
            return EXIT_FAILURE;
        }

        appResult.value().run();
    }
    catch (const std::exception &e) {
        std::println(stderr, "Unhandled exception: {}", e.what());
        return EXIT_FAILURE;
    }
    catch (...) {
        std::println(stderr, "Unknown exception occurred");
        return EXIT_FAILURE;
    }
    return EXIT_SUCCESS;
}
```

### 🔬 실행 및 테스트

실행 결과

- 800x600 윈도우 생성
- ESC: 종료
- Space: 배경색 변경
- I: 정보 출력
- FPS 콘솔 출력

<details>
<summary>전체 데모 영상 보기</summary>
<video width="100%" controls>
  <source src="/assets/videos/OpenGL/Window RAII.mp4" type="video/mp4">
</video>
</details>

## 🔑 핵심 개념 설명

### 1. RAII 패턴

```cpp
{
    auto windowResult = gl::Window::create(properties); // 리소스 획득
    // ... 사용
} // 스코프 종료 시 자동으로 리소스 해제
```

### 2. std::expected (에러 처리)

```cpp
// 예외 대신 값으로 에러 반환
[[nodiscard]] static std::expected<std::unique_ptr<Window>, Error>
    create(const WindowProperties &properties = {}) noexcept;

// 사용
if (!windowResult) {
    // 에러 처리
    return std::unexpected(windowResult.error());
}
```

### 3. Custom Deleter

```cpp
using WindowDeleter = decltype([](WindowHandle window) {
    if (window) {
        glfwDestroyWindow(window);
    }
});
using WindowPtr = std::unique_ptr<GLFWwindow, WindowDeleter>;
```

## 📊 이전 코드와 비교

| 측면 | 이전(절차적) | 현재(OOP + RAII) |
|------|------|------|
| 리소스 관리 | 수정(`glfwTerminate`) | 자동(소멸자) |
| 에러 처리| if문 중첨 | `std::expected` |
| 코드 재사용 | 복사-붙여넣기 | 클래스 인스턴스 |
| 예외 안전성 | 보장 못함 | RAII로 보장 |
| 확장성 | 어려움 | 쉬움 |

## 💡 개선 포인트

현재 코드의 장점:

- ✅ 자동 리소스 관리
- ✅ 타입 안전성
- ✅ 에러 처리 명확
- ✅ 확장 가능한 구조

추가 개선 가능한 부분:

- 이벤트 시스템 추가
- 다중 윈도우 지원
- 렌더링 컨텍스트 공유

## 🚀 다음 단계

다음 포스트에서 다룰 내용:

- Shader 클래스 설계
- VAO/VBO 레퍼 클래스
- 첫 번째 삼각형 렌더링
- 셰이더 컴파일 및 링킹

## 💬 트러블 슈팅

문제: "`std::expected`를 찾을 수 없음"

```cpp
// C++23인지 확인
// C++20에서는 직접 구현하거나 라이브러리 사용
// 또는 std::optional + 에러 코드로 대체
```

문제: "private 생성자 접근 에러"

```cpp
// make_unique 대신 new 직접 사용
std::unique_ptr<T>(new T());
```

문제: "GL/glew.h not found"

```bash
#macOS
brew install glew
```

### 💬 댓글

문제에 대한 질문이나 다른 풀이 방법이 있다면 아래 댓글로 공유해주세요!
---
layout: post
title: "[OpenGL] 05 - Transform과 GLM으로 2D 변환"
date: 2025-09-07
categories: [Graphics, OpenGL, Tutorial]
tags: [cpp, opengl, graphics, transform, glm, 2d, matrix, mesh]
---

## 📌 시리즈 정보

<div style="display: flex; gap: 10px; margin-bottom: 20px;">
<img src="https://img.shields.io/badge/플랫폼-macOS-black?style=flat-square" alt="macOS" />
<img src="https://img.shields.io/badge/IDE-CLion-00ACC1?style=flat-square" alt="CLion" />
<img src="https://img.shields.io/badge/OpenGL-3.3+-5586A4?style=flat-square" alt="OpenGL" />
<img src="https://img.shields.io/badge/C++-23-00599C?style=flat-square" alt="C++23" />
</div>

- 이전 포스트: [04 - Shader 클래스 설계와 Uniform](/blog/[OpenGL]%2004%20-%20Shader%20클래스%20설계와%20Uniform.html)
- **현재 단계: Transform과 GLM으로 2D 변환**
- 다음 포스트: 05-2 - 시계와 태양계 구현하기

## 📖 이번 챕터의 목표

2D 그래픽스의 핵심인 **변환(Transformation)**을 배우고, **GLM 라이브러리**를 활용해 Transform 시스템을 구축합니다. 추가로 **Mesh 클래스**를 만들어 VAO/VBO를 RAII로 관리합니다.

## 🎯 변환의 3요소: TRS

### Translation, Rotation, Scale

모든 2D 변환은 세 가지 기본 변환의 조합입니다:
[더 자세한 내용은...](/blog/[OpenGL]%20Transform%20행렬의%20수학적%20이해.html)


```
원본 도형 → Scale(크기) → Rotation(회전) → Translation(이동) → 최종 결과
```

### 행렬 곱셈 순서의 중요성

```cpp
// 올바른 순서: T × R × S
glm::mat4 transform = translate * rotate * scale;

// 잘못된 순서: S × R × T (전혀 다른 결과!)
glm::mat4 wrong = scale * rotate * translate;
```

순서가 바뀌면:
- Scale → Translate: 크기를 키운 후 이동하면 이동 거리도 스케일됨
- Translate → Rotate: 이동 후 회전하면 원점이 아닌 곳에서 회전

<video width="100%" controls>
  <source src="/assets/videos/OpenGL/trs-matrix.mp4" type="video/mp4">
  Transform2D 데모 영상
</video>

## 💡 핵심 개념: GLM과 변환 행렬

### GLM (OpenGL Mathematics)

GLM은 OpenGL을 위한 헤더 온리 수학 라이브러리입니다:

```cpp
#include <glm/glm.hpp>                   // 벡터, 행렬
#include <glm/gtc/matrix_transform.hpp>   // 변환 함수

// 기본 사용법
glm::vec2 position(0.0f, 0.0f);
glm::mat4 transform(1.0f);  // 단위 행렬

// 변환 적용
transform = glm::translate(transform, glm::vec3(position, 0.0f));
transform = glm::rotate(transform, angle, glm::vec3(0, 0, 1));  // Z축 회전
transform = glm::scale(transform, glm::vec3(scale, 1.0f));
```

### 2D에서 4x4 행렬을 쓰는 이유

2D는 3x3 행렬이면 충분하지만, OpenGL은 **동차 좌표계**를 사용합니다:
- 2D 점 (x, y) → 4D 동차 좌표 (x, y, 0, 1)
- 이동(Translation)을 행렬 곱셈으로 표현 가능
- 3D와 일관된 파이프라인

## 🏗️ 프로젝트 구조

### 새로운 클래스들

1. **Transform2D**: 2D 변환 관리
2. **GeometryBuilder**: 기하 도형 생성 유틸리티
3. **Mesh**: VAO/VBO RAII 래퍼

### 파일 구조

```
Graphics_Study/
├── CMakeLists.txt
├── main.cpp
├── window.hpp/cpp
├── shader.hpp/cpp
├── transform2d.hpp      # 새 파일
├── geometry_builder.hpp # 새 파일
├── mesh.hpp            # 새 파일
└── shaders/
    ├── transform.vert  # 수정
    └── color.frag      # 새 파일
```

## 📝 구현

### Step 1: Transform2D 클래스

**transform2d.hpp**

```cpp
#ifndef TRANSFORM2D_HPP
#define TRANSFORM2D_HPP

#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>

namespace gl {
    /**
     * 2D 변환 컴포넌트
     * 위치, 회전, 크기를 관리하고 변환 행렬 생성
     */
    class Transform2D final {
    public:
        // 기본 생성자
        Transform2D() = default;

        // 위치만 지정하는 생성자
        explicit Transform2D(const glm::vec2& pos) noexcept
            : position(pos) {}
        
        // 위치와 회전을 지정하는 생성자
        Transform2D(const glm::vec2& pos, const float rot) noexcept
            : position(pos), rotation(rot) {}
        
        // 전체 지정 생성자
        Transform2D(const glm::vec2& pos, const float rot, const glm::vec2& scl) noexcept
            : position(pos), rotation(rot), scale(scl) {}

        // 복사 생성자/대입 연산자
        Transform2D(const Transform2D&) = default;
        Transform2D& operator=(const Transform2D&) = default;

        // 이동 생성자/대입 연산자
        Transform2D(Transform2D&&) noexcept = default;
        Transform2D& operator=(Transform2D&&) noexcept = default;

        // 소멸자
        ~Transform2D() = default;

        /**
         * 최종 변환 행렬 계산
         * 순서: Translation × Rotation × Scale (TRS)
         */
        [[nodiscard]] glm::mat4 getMatrix() const noexcept {
            const glm::vec3 position3D(position.x, position.y, 0.0f);
            const glm::vec3 scale3D(scale.x, scale.y, 1.0f);

            glm::mat4 transform(1.0f);
            transform = glm::translate(transform, position3D);
            transform = glm::rotate(transform, rotation, glm::vec3(0.0f, 0.0f, 1.0f)); // Z축 회전
            transform = glm::scale(transform, scale3D);

            return transform;
        }

        /**
         * 부모 Transform과 결합 (계층 구조 지원)
         */
        [[nodiscard]] glm::mat4 getMatrix(const glm::mat4& parentMatrix) const noexcept {
            return parentMatrix * getMatrix();
        }

        /**
         * 초기화
         */
        Transform2D& reset() noexcept {
            position = glm::vec2(0.0f, 0.0f);
            rotation = 0.0f;
            scale = glm::vec2(1.0f, 1.0f);
            return *this;
        }

    public:
        // Transform 데이터 (의도적으로 public - Unity 스타일)
        glm::vec2 position = glm::vec2{0.0f, 0.0f};  // 2D 위치
        float rotation = 0.0f;                        // Z축 회전 (라디안)
        glm::vec2 scale = glm::vec2{1.0f, 1.0f};     // 2D 스케일
    };

    // 수학 유틸리티
    namespace math {
        inline constexpr float PI = glm::pi<float>();
        inline constexpr float TWO_PI = 2.0f * glm::pi<float>();
        inline constexpr float HALF_PI = glm::half_pi<float>();
        
        [[nodiscard]] inline float radians(float degrees) noexcept {
            return glm::radians(degrees);
        }
        
        [[nodiscard]] inline float degrees(float radians) noexcept {
            return glm::degrees(radians);
        }
    }
}

#endif //TRANSFORM2D_HPP
```

### Step 2: GeometryBuilder 클래스

**geometry_builder.hpp**

```cpp
#ifndef GEOMETRY_BUILDER_HPP
#define GEOMETRY_BUILDER_HPP

#include <vector>
#include <glm/glm.hpp>
#include <glm/gtc/constants.hpp>

namespace gl {
    /**
     * 2D 기하 도형 생성 유틸리티
     * Static 메서드로 정점 데이터 생성
     */
    class GeometryBuilder final {
    public:
        // Static 클래스 (인스턴스 생성 방지)
        GeometryBuilder() = delete;

        /**
         * 원 테두리 생성 (GL_LINE_LOOP용)
         */
        [[nodiscard]] static std::vector<float> createCircle(
            float radius = 1.0f, 
            int segments = 32) noexcept {
            
            std::vector<float> vertices;
            vertices.reserve((segments + 1) * 3);

            const float angleStep = 2.0f * glm::pi<float>() / segments;
            for (int i = 0; i <= segments; ++i) {
                const float angle = i * angleStep;
                const float x = radius * std::cos(angle);
                const float y = radius * std::sin(angle);

                vertices.emplace_back(x);
                vertices.emplace_back(y);
                vertices.emplace_back(0.0f);
            }

            return vertices;
        }

        /**
         * 채워진 원 생성 (GL_TRIANGLE_FAN용)
         */
        [[nodiscard]] static std::vector<float> createFilledCircle(
            float radius = 1.0f, 
            int segments = 32) noexcept {
            
            std::vector<float> vertices;
            vertices.reserve((segments + 2) * 3);

            // 중심점
            vertices.emplace_back(0.0f);
            vertices.emplace_back(0.0f);
            vertices.emplace_back(0.0f);

            const float angleStep = 2.0f * glm::pi<float>() / segments;
            for (int i = 0; i <= segments; ++i) {
                const float angle = i * angleStep;
                const float x = radius * std::cos(angle);
                const float y = radius * std::sin(angle);

                vertices.emplace_back(x);
                vertices.emplace_back(y);
                vertices.emplace_back(0.0f);
            }

            return vertices;
        }

        /**
         * 사각형 생성 (GL_TRIANGLES용)
         */
        [[nodiscard]] static std::vector<float> createRectangle(
            float width = 1.0f,
            float height = 1.0f) noexcept {
            
            const float halfWidth = width / 2.0f;
            const float halfHeight = height / 2.0f;
            
            return {
                // 첫 번째 삼각형
                -halfWidth, -halfHeight, 0.0f,
                 halfWidth, -halfHeight, 0.0f,
                 halfWidth,  halfHeight, 0.0f,
                
                // 두 번째 삼각형
                -halfWidth, -halfHeight, 0.0f,
                 halfWidth,  halfHeight, 0.0f,
                -halfWidth,  halfHeight, 0.0f,
            };
        }

        /**
         * 정삼각형 생성
         */
        [[nodiscard]] static std::vector<float> createTriangle(
            float size = 1.0f) noexcept {
            
            const float height = size * std::sqrt(3.0f) / 2.0f;
            const float halfSize = size / 2.0f;
            
            return {
                -halfSize, -height/3.0f, 0.0f,  // 왼쪽 아래
                 halfSize, -height/3.0f, 0.0f,  // 오른쪽 아래
                 0.0f, height*2.0f/3.0f, 0.0f,  // 위
            };
        }
    };
}

#endif //GEOMETRY_BUILDER_HPP
```

### Step 3: Mesh 클래스

**mesh.hpp**

```cpp
#ifndef MESH_HPP
#define MESH_HPP

#include <vector>
#include <GL/glew.h>

namespace gl {
    /**
     * VAO/VBO를 RAII로 관리하는 Mesh 클래스
     */
    class Mesh final {
    public:
        /**
         * 정점 데이터로 Mesh 생성
         * @param vertices 정점 데이터 (x, y, z 순서)
         * @param drawMode 그리기 모드 (GL_TRIANGLES, GL_LINE_LOOP 등)
         */
        Mesh(const std::vector<float>& vertices, GLenum drawMode) noexcept
            : m_drawMode(drawMode) {
            
            m_vertexCount = static_cast<GLsizei>(vertices.size() / 3);

            // VAO 생성 및 바인딩
            glGenVertexArrays(1, &m_vao);
            glGenBuffers(1, &m_vbo);

            glBindVertexArray(m_vao);
            glBindBuffer(GL_ARRAY_BUFFER, m_vbo);

            // 데이터 업로드
            glBufferData(GL_ARRAY_BUFFER,
                        static_cast<GLsizeiptr>(vertices.size() * sizeof(float)),
                        vertices.data(),
                        GL_STATIC_DRAW);

            // 정점 속성 설정
            glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), nullptr);
            glEnableVertexAttribArray(0);

            // 바인딩 해제
            glBindBuffer(GL_ARRAY_BUFFER, 0);
            glBindVertexArray(0);
        }

        ~Mesh() noexcept {
            if (m_vao) {
                glDeleteVertexArrays(1, &m_vao);
            }
            if (m_vbo) {
                glDeleteBuffers(1, &m_vbo);
            }
        }

        // 복사 방지
        Mesh(const Mesh&) = delete;
        Mesh& operator=(const Mesh&) = delete;

        // 이동 생성자
        Mesh(Mesh&& other) noexcept
            : m_vao(std::exchange(other.m_vao, 0))
            , m_vbo(std::exchange(other.m_vbo, 0))
            , m_vertexCount(other.m_vertexCount)
            , m_drawMode(other.m_drawMode) {}

        Mesh& operator=(Mesh&& other) noexcept {
            if (this != &other) {
                // 기존 리소스 정리
                if (m_vao) glDeleteVertexArrays(1, &m_vao);
                if (m_vbo) glDeleteBuffers(1, &m_vbo);
                
                // 이동
                m_vao = std::exchange(other.m_vao, 0);
                m_vbo = std::exchange(other.m_vbo, 0);
                m_vertexCount = other.m_vertexCount;
                m_drawMode = other.m_drawMode;
            }
            return *this;
        }

        /**
         * Mesh 그리기
         */
        void draw() const noexcept {
            glBindVertexArray(m_vao);
            glDrawArrays(m_drawMode, 0, m_vertexCount);
            glBindVertexArray(0);
        }

        [[nodiscard]] GLsizei getVertexCount() const noexcept {
            return m_vertexCount;
        }

    private:
        GLuint m_vao = 0;
        GLuint m_vbo = 0;
        GLsizei m_vertexCount = 0;
        GLenum m_drawMode;
    };
}

#endif //MESH_HPP
```

### Step 4: Shader 수정

**shaders/transform.vert**

```glsl
#version 330 core

layout (location = 0) in vec3 aPos;

uniform mat4 u_Transform;

void main() {
    gl_Position = u_Transform * vec4(aPos, 1.0);
}
```

**shaders/color.frag**

```glsl
#version 330 core

out vec4 FragColor;

uniform vec3 u_Color;
uniform float u_Alpha;

void main() {
    FragColor = vec4(u_Color, u_Alpha);
}
```

### Step 5: 메인 애플리케이션

**main.cpp**

```cpp
#include <chrono>
#include <print>
#include <thread>
#include <cassert>
#include <memory>

#include "geometry_builder.hpp"
#include "mesh.hpp"
#include "window.hpp"
#include "shader.hpp"
#include "transform2d.hpp"

namespace {
    constexpr std::int32_t WINDOW_WIDTH = 800;
    constexpr std::int32_t WINDOW_HEIGHT = 600;
    constexpr std::string_view WINDOW_TITLE = "OpenGL Multi-Shape Demo";

    struct Color {
        float r, g, b, a;

        constexpr Color(float r, float g, float b, float a = 1.0f)
            : r(r), g(g), b(b), a(a) {}

        [[nodiscard]] glm::vec3 toVec3() const noexcept {
            return {r, g, b};
        }
    };

    constexpr Color BACKGROUND_COLOR{0.2f, 0.3f, 0.3f, 1.0f};
    constexpr Color WHITE{1.0f, 1.0f, 1.0f, 1.0f};
    constexpr Color CYAN{0.2f, 0.8f, 0.8f, 1.0f};
    constexpr Color ORANGE{1.0f, 0.5f, 0.2f, 1.0f};
    constexpr Color PURPLE{0.8f, 0.2f, 0.8f, 1.0f};

    enum class ShapeType {
        Circle,
        Triangle,
        Rectangle
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

        Application app{std::move(windowResult.value())};

        if (!app.setupShaders()) {
            return std::unexpected(gl::WindowError{
                .code = gl::WindowErrorCode::ContextCreationFailed,
                .message = "Failed to compile/link shaders"
            });
        }

        if (!app.setupGeometry()) {
            return std::unexpected(gl::WindowError{
                .code = gl::WindowErrorCode::ContextCreationFailed,
                .message = "Failed to create geometry"
            });
        }

        return app;
    }

    ~Application() = default;

    Application(const Application &) = delete;
    Application &operator=(const Application &) = delete;

    Application(Application &&) noexcept = default;
    Application &operator=(Application &&) noexcept = default;

    void run() {
        setupCallbacks();
        printInfo();

        auto lastTime = std::chrono::steady_clock::now();
        const auto startTime = std::chrono::steady_clock::now();
        std::size_t frameCount = 0;

        while (!m_window->shouldClose()) {
            auto currentTime = std::chrono::steady_clock::now();
            const auto totalElapsedTime = std::chrono::duration<float>(currentTime - startTime);
            m_elapsedTime = totalElapsedTime.count();

            ++frameCount;

            const auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(currentTime - lastTime);
            if (elapsed.count() >= 1000) {
                const double fps = frameCount * 1000.0 / elapsed.count();
                std::println("FPS: {:.1f}", fps);
                frameCount = 0;
                lastTime = currentTime;
            }

            updateAnimation();
            render();

            m_window->swapBuffers();
            m_window->pollEvents();

            std::this_thread::sleep_for(std::chrono::milliseconds(1));
        }
    }

private:
    explicit Application(std::unique_ptr<gl::Window> window) noexcept
        : m_window(std::move(window)) {}

    void setupCallbacks() {
        m_window->setKeyCallback([this](const int key, const int action, int) {
            if (action == GLFW_PRESS) {
                handleKeyPress(key);
            }
        });
    }

    void handleKeyPress(const int key) {
        switch (key) {
            case GLFW_KEY_ESCAPE:
                std::println("Escape pressed - closing window");
                glfwSetWindowShouldClose(glfwGetCurrentContext(), GLFW_TRUE);
                break;

            case GLFW_KEY_1:
                m_currentShape = ShapeType::Circle;
                std::println("Shape: Circle");
                break;

            case GLFW_KEY_2:
                m_currentShape = ShapeType::Triangle;
                std::println("Shape: Triangle");
                break;

            case GLFW_KEY_3:
                m_currentShape = ShapeType::Rectangle;
                std::println("Shape: Rectangle");
                break;

            case GLFW_KEY_SPACE:
                m_autoRotate = !m_autoRotate;
                std::println("Auto-rotate: {}", m_autoRotate ? "ON" : "OFF");
                break;

            case GLFW_KEY_R:
                m_transform.reset();
                std::println("Transform reset");
                break;

            case GLFW_KEY_Q:
                m_transform.rotation -= gl::math::radians(15.0f);
                std::println("Rotation: {:.1f}°", gl::math::degrees(m_transform.rotation));
                break;

            case GLFW_KEY_E:
                m_transform.rotation += gl::math::radians(15.0f);
                std::println("Rotation: {:.1f}°", gl::math::degrees(m_transform.rotation));
                break;

            case GLFW_KEY_UP:
                m_transform.position.y += 0.1f;
                break;

            case GLFW_KEY_DOWN:
                m_transform.position.y -= 0.1f;
                break;

            case GLFW_KEY_RIGHT:
                m_transform.position.x += 0.1f;
                break;

            case GLFW_KEY_LEFT:
                m_transform.position.x -= 0.1f;
                break;

            case GLFW_KEY_EQUAL:  // + key
                m_transform.scale *= 1.1f;
                std::println("Scale: {:.2f}", m_transform.scale.x);
                break;

            case GLFW_KEY_MINUS:
                m_transform.scale *= 0.9f;
                std::println("Scale: {:.2f}", m_transform.scale.x);
                break;

            case GLFW_KEY_I:
                printInfo();
                break;
        }
    }

    void updateAnimation() {
        if (m_autoRotate) {
            m_transform.rotation = m_elapsedTime * 0.5f;
        }

        // 부드러운 크기 변화 애니메이션
        const float scale = 0.4f + 0.05f * std::sin(m_elapsedTime * 2.0f);
        m_transform.scale = glm::vec2(scale);
    }

    void render() const {
        glClearColor(BACKGROUND_COLOR.r, BACKGROUND_COLOR.g, BACKGROUND_COLOR.b, BACKGROUND_COLOR.a);
        glClear(GL_COLOR_BUFFER_BIT);

        m_shader->use();

        // Aspect ratio 보정
        const auto [width, height] = m_window->getWindowSize();
        const float aspectRatio = static_cast<float>(width) / static_cast<float>(height);
        
        glm::mat4 transform = m_transform.getMatrix();
        transform = glm::scale(glm::mat4(1.0f), glm::vec3(1.0f / aspectRatio, 1.0f, 1.0f)) * transform;

        m_shader->setMat4("u_Transform", transform);

        // 현재 선택된 도형 그리기
        switch (m_currentShape) {
            case ShapeType::Circle:
                m_shader->setVec3("u_Color", CYAN.toVec3());
                m_shader->setFloat("u_Alpha", 1.0f);
                m_circleMesh->draw();
                break;

            case ShapeType::Triangle:
                m_shader->setVec3("u_Color", ORANGE.toVec3());
                m_shader->setFloat("u_Alpha", 1.0f);
                m_triangleMesh->draw();
                break;

            case ShapeType::Rectangle:
                m_shader->setVec3("u_Color", PURPLE.toVec3());
                m_shader->setFloat("u_Alpha", 1.0f);
                m_rectangleMesh->draw();
                break;
        }
    }

    bool setupShaders() {
        std::println("Setting up shaders...");

        auto shaderResult = gl::Shader::create(
            "shaders/transform.vert",
            "shaders/color.frag");

        if (!shaderResult) {
            std::println(stderr, "Failed to create shader: {}", shaderResult.error().message);
            return false;
        }

        m_shader = std::move(shaderResult.value());

        std::println("Shaders compiled and linked successfully!");
        return true;
    }

    bool setupGeometry() {
        std::println("Setting up geometry...");

        // 원 (테두리)
        auto circleVertices = gl::GeometryBuilder::createCircle(0.5f, 64);
        m_circleMesh = std::make_unique<gl::Mesh>(circleVertices, GL_LINE_LOOP);

        // 삼각형
        auto triangleVertices = gl::GeometryBuilder::createTriangle(1.0f);
        m_triangleMesh = std::make_unique<gl::Mesh>(triangleVertices, GL_TRIANGLES);

        // 사각형
        auto rectangleVertices = gl::GeometryBuilder::createRectangle(0.8f, 0.6f);
        m_rectangleMesh = std::make_unique<gl::Mesh>(rectangleVertices, GL_TRIANGLES);

        std::println("All shapes created successfully!");
        std::println("Circle vertices: {}", m_circleMesh->getVertexCount());
        std::println("Triangle vertices: {}", m_triangleMesh->getVertexCount());
        std::println("Rectangle vertices: {}", m_rectangleMesh->getVertexCount());

        return true;
    }

    void printInfo() const {
        std::println("\n=== OpenGL Multi-Shape Demo ===");
        std::println("{}", m_window->getOpenGLInfo());

        const auto [width, height] = m_window->getWindowSize();
        std::println("Window Size: {}x{}", width, height);

        std::println("\n=== Controls ===");
        std::println("1/2/3: Switch shapes (Circle/Triangle/Rectangle)");
        std::println("Q/E: Rotate manually");
        std::println("Arrow Keys: Move");
        std::println("+/-: Scale");
        std::println("SPACE: Toggle auto-rotation");
        std::println("R: Reset transform");
        std::println("I: Show this info");
        std::println("ESC: Exit");
        
        const char* shapeName = "";
        switch (m_currentShape) {
            case ShapeType::Circle: shapeName = "Circle"; break;
            case ShapeType::Triangle: shapeName = "Triangle"; break;
            case ShapeType::Rectangle: shapeName = "Rectangle"; break;
        }
        std::println("\nCurrent Shape: {}", shapeName);
        std::println("Auto-rotate: {}", m_autoRotate ? "ON" : "OFF");
        std::println("================\n");
    }

private:
    std::unique_ptr<gl::Window> m_window;
    std::unique_ptr<gl::Shader> m_shader;
    
    // Mesh 객체들
    std::unique_ptr<gl::Mesh> m_circleMesh;
    std::unique_ptr<gl::Mesh> m_triangleMesh;
    std::unique_ptr<gl::Mesh> m_rectangleMesh;
    
    // 상태
    gl::Transform2D m_transform;
    ShapeType m_currentShape = ShapeType::Circle;
    float m_elapsedTime = 0.0f;
    bool m_autoRotate = true;
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

### Step 6: CMakeLists.txt 수정

```cmake
cmake_minimum_required(VERSION 3.31)
project(Graphics_Study)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if (NOT MSVC)
    add_compile_options(-Wall -Wextra -Wpedantic)
endif ()

if (APPLE)
    add_compile_options(-stdlib=libc++)
    add_link_options(-stdlib=libc++)
    add_compile_definitions(GL_SILENCE_DEPRECATION)
endif ()

find_package(OpenGL REQUIRED)
find_package(glfw3 REQUIRED)
find_package(GLEW REQUIRED)
find_package(glm REQUIRED)

file(GLOB SHADER_FILES "${CMAKE_SOURCE_DIR}/shaders/*.vert" "${CMAKE_SOURCE_DIR}/shaders/*.frag")

add_executable(${PROJECT_NAME} 
    main.cpp
    window.cpp
    window.hpp
    ${SHADER_FILES}
    shader.cpp
    shader.hpp
    transform2d.hpp
    geometry_builder.hpp
    mesh.hpp
)

add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${CMAKE_SOURCE_DIR}/shaders
    $<TARGET_FILE_DIR:${PROJECT_NAME}>/shaders
)

target_link_libraries(${PROJECT_NAME}
    OpenGL::GL
    glfw
    GLEW::GLEW
    glm::glm
)
```

## 🎨 화면비 왜곡 문제 해결

### 문제: 원이 타원으로 보임

OpenGL의 NDC는 정사각형(-1 ~ 1)이지만, 화면은 직사각형입니다. 해결책:

```cpp
// Aspect ratio 계산
const float aspectRatio = static_cast<float>(width) / static_cast<float>(height);

// X축 스케일 조정
glm::mat4 transform = m_transform.getMatrix();
transform = glm::scale(glm::mat4(1.0f), glm::vec3(1.0f / aspectRatio, 1.0f, 1.0f)) * transform;
```

## 🔬 실행 결과

<video width="100%" controls>
  <source src="/assets/videos/OpenGL/transform2d.mp4" type="video/mp4">
  Transform2D 데모 영상
</video>

### 조작법

- **1/2/3**: 도형 전환 (원/삼각형/사각형)
- **Q/E**: 15도씩 회전
- **방향키**: 위치 이동
- **+/-**: 크기 조절
- **SPACE**: 자동 회전 토글
- **R**: Transform 초기화
- **I**: 정보 표시
- **ESC**: 종료

### 특징

- 각 도형별 다른 색상 (원: Cyan, 삼각형: Orange, 사각형: Purple)
- 부드러운 자동 회전 애니메이션
- 크기 펄스 효과
- Mesh 클래스로 깔끔한 VAO/VBO 관리

## 🎓 핵심 정리

### 이번 챕터에서 배운 것

1. **Transform2D 클래스**
   - TRS 순서의 중요성
   - Public 멤버 변수 (Unity 스타일)
   - 계층 구조 지원

2. **GeometryBuilder**
   - Static 유틸리티 클래스
   - 다양한 2D 도형 생성
   - 재사용 가능한 정점 데이터

3. **Mesh 클래스**
   - VAO/VBO RAII 관리
   - 이동 의미론 지원
   - 간단한 draw() 인터페이스

4. **GLM 활용**
   - 벡터와 행렬 연산
   - 변환 함수들
   - 라디안/도 변환

### 주의사항

```cpp
// 올바른 TRS 순서
transform = translate * rotate * scale;

// GLM은 라디안 사용
rotation = glm::radians(45.0f);  // 45도를 라디안으로

// 화면비 보정 필수
transform = aspectCorrection * transform;
```

## 💬 트러블슈팅

### 문제 1: 회전이 이상함
**원인**: 도(degree)와 라디안(radian) 혼동
**해결**: `glm::radians()` 사용

### 문제 2: 원이 찌그러짐
**원인**: 화면비 미적용
**해결**: Aspect ratio로 X축 보정

### 문제 3: 도형이 안 보임
**체크리스트**:
- Shader 파일 경로 확인
- VAO/VBO 바인딩 확인
- Transform 행렬 uniform 전달 확인

## 🚀 다음 단계

다음 챕터에서는:
- 3D 공간으로 확장
- View와 Projection 행렬
- 카메라 시스템
- 깊이 테스트

## 📚 참고 자료

- [GLM Documentation](https://glm.g-truc.net/0.9.9/index.html)
- [Learn OpenGL - Transformations](https://learnopengl.com/Getting-started/Transformations)
- [OpenGL Transformation](http://www.songho.ca/opengl/gl_transform.html)

---

### 💬 댓글

Transform과 Mesh 클래스에 대한 질문이나 개선 아이디어가 있다면 아래 댓글로 공유해주세요!
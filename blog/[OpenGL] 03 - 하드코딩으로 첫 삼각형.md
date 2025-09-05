---
layout: post
title: "[OpenGL] 03 - 하드코딩으로 첫 삼각형"
date: 2025-09-05
categories: [Graphics, OpenGL, Tutorial]
tags: [cpp, opengl, graphics, rendering, vbo, vao, shader]
---

## 📌 시리즈 정보

<div style="display: flex; gap: 10px; margin-bottom: 20px;">
<img src="https://img.shields.io/badge/플랫폼-macOS-black?style=flat-square" alt="macOS" />
<img src="https://img.shields.io/badge/IDE-CLion-00ACC1?style=flat-square" alt="CLion" />
<img src="https://img.shields.io/badge/OpenGL-3.3+-5586A4?style=flat-square" alt="OpenGL" />
<img src="https://img.shields.io/badge/C++-23-00599C?style=flat-square" alt="C++23" />
</div>

- 이전 포스트: [02 Window 클래스 설계와 RAII 패턴](/blog/opengl-02-window-class-raii.html)
- **현재 단계: 하드코딩으로 첫 삼각형 그리기**
- 다음 포스트: 04 Shader 클래스 설계

## 📖 이번 챕터의 목표

> "Make it work, make it right, make it fast" - Kent Beck

코드 구조는 잠시 제쳐두고, **주황색 삼각형을 화면에 띄우는 것**이 목표입니다. 모든 코드를 `main.cpp`에 하드코딩하여 OpenGL 렌더링 파이프라인의 핵심 흐름을 이해합니다.

## 🎯 시작하기 전에

현재 우리가 가진 것:
- ✅ Window 클래스 (GLFW/GLEW 초기화 완료)
- ✅ OpenGL 컨텍스트 생성
- ✅ 렌더링 루프 동작 중

이제 추가할 것:
- 🔺 셰이더 프로그램 (GPU에서 실행될 코드)
- 🔺 정점 데이터 (삼각형의 3개 점)
- 🔺 VAO/VBO (GPU로 데이터 전송)

## 💡 핵심 개념 이해

### OpenGL 렌더링 파이프라인 (간단 버전)

```
정점 데이터 (3개의 점)
    ↓ VBO에 업로드
Vertex Shader (위치 결정)
    ↓
Primitive Assembly (삼각형 조립)
    ↓
Rasterization (픽셀로 변환)
    ↓
Fragment Shader (색상 결정)
    ↓ 
화면에 삼각형!
```

<video width="100%" controls loop muted>
  <source src="/assets/videos/OpenGL/opengl-pipeline.mp4" type="video/mp4">
  OpenGL 렌더링 파이프라인 애니메이션
</video>

### NDC (Normalized Device Coordinates)

```
     (0, 1)
        ^
        |
(-1,0) --+--> (1, 0)
        |
        v
     (0, -1)
```

모든 좌표는 -1.0 ~ 1.0 범위 안에 있어야 화면에 표시됩니다.

## 🔑 핵심 개념 상세 설명

### 1. 셰이더 프로그램이란?

셰이더는 **GPU에서 실행되는 작은 프로그램**입니다. 우리가 작성한 C++ 코드는 CPU에서 실행되지만, 셰이더는 GPU에서 병렬로 실행됩니다.

<video width="100%" controls loop muted>
  <source src="/assets/videos/OpenGL/shader-compile.mp4" type="video/mp4">
  셰이더 컴파일 및 링킹 과정
</video>

#### Vertex Shader
- **실행 시점**: 각 정점마다 한 번씩
- **입력**: 정점의 위치 (x, y, z)
- **출력**: 변환된 정점 위치 (gl_Position)
- **우리 예제**: 입력을 그대로 출력 (변환 없음)

#### Fragment Shader
- **실행 시점**: 각 픽셀마다 한 번씩
- **입력**: 래스터라이제이션된 픽셀 정보
- **출력**: 픽셀의 최종 색상 (FragColor)
- **우리 예제**: 모든 픽셀을 주황색으로

### 2. VBO와 VAO 완벽 이해

<video width="100%" controls loop muted>
  <source src="/assets/videos/OpenGL/vao-vbo.mp4" type="video/mp4">
  VBO/VAO 데이터 전송 과정
</video>

#### VBO (Vertex Buffer Object)
- **역할**: 정점 데이터를 GPU 메모리에 저장
- **내용**: 실제 float 배열 (x, y, z 좌표값들)
- **비유**: 데이터가 담긴 "창고"

```cpp
// VBO는 이런 raw 데이터를 GPU에 저장
float vertices[] = {
    -0.5f, -0.5f, 0.0f,  // 정점 0: 왼쪽 아래
     0.5f, -0.5f, 0.0f,  // 정점 1: 오른쪽 아래
     0.0f,  0.5f, 0.0f   // 정점 2: 위 중앙
};
// 총 9개의 float = 3개 정점 × 3개 좌표(x,y,z)
```

#### VAO (Vertex Array Object)
- **역할**: VBO 데이터의 해석 방법과 레이아웃 정보 저장
- **내용**: 속성 포인터 설정, VBO 바인딩 상태
- **비유**: 데이터를 읽는 "설명서" 또는 "독서 가이드"

```cpp
// VAO는 이런 정보들을 저장
glVertexAttribPointer(
    0,                   // location = 0 (셰이더의 aPos와 연결)
    3,                   // 3개씩 읽기 (x, y, z)
    GL_FLOAT,           // float 타입
    GL_FALSE,           // 정규화 안함
    3 * sizeof(float),  // stride: 다음 정점까지 12바이트
    (void*)0            // offset: 0번째 바이트부터 시작
);
```

#### 왜 VAO가 필요한가?

**VAO 없이 매번 설정한다면:**
```cpp
// 매 프레임마다 이 모든 설정을 반복...
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 12, nullptr);
glEnableVertexAttribArray(0);
glDrawArrays(GL_TRIANGLES, 0, 3);
```

**VAO를 사용하면:**
```cpp
// 초기화 시 한 번만 설정
glBindVertexArray(vao);
glBindBuffer(GL_ARRAY_BUFFER, vbo);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 12, nullptr);
glEnableVertexAttribArray(0);

// 렌더링 시 간단하게
glBindVertexArray(vao);  // 모든 설정이 자동 적용!
glDrawArrays(GL_TRIANGLES, 0, 3);
```

### 3. 데이터 흐름 다이어그램

```
CPU (main.cpp)           GPU Memory              Shader Pipeline
    |                        |                         |
    |-- vertices[] --------> VBO                      |
    |                        |                         |
    |-- VAO 설정 ----------> VAO                      |
    |                    (VBO 해석 방법)              |
    |                        |                         |
    |-- glDrawArrays() ------|----------------------> Vertex Shader
                             |                         (각 정점 처리)
                             |                         |
                             |                         v
                             |                      Primitive Assembly
                             |                         |
                             |                         v
                             |                      Rasterization
                             |                         |
                             |                         v
                             |                      Fragment Shader
                             |                      (각 픽셀 처리)
                             |                         |
                             └----------------------> Framebuffer
                                                    (화면 출력)
```

## 📝 코드 구현

### Step 1: 셰이더 소스 코드 추가

`main.cpp` 상단, 익명 네임스페이스에 셰이더 코드를 하드코딩합니다:

```cpp
namespace {
    // Vertex Shader: 각 정점의 위치를 결정
    constexpr const char *VERTEX_SHADER_SOURCE = R"(
#version 330 core
// layout (location = 0): VAO에서 설정할 속성 위치
layout (location = 0) in vec3 aPos;     // 정점 위치 입력

void main() {
    // 입력받은 위치를 그대로 출력
    // vec3를 vec4로 변환 (w = 1.0은 점을 의미)
    gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);
}
)";

    // Fragment Shader: 각 픽셀의 색상을 결정
    constexpr const char *FRAGMENT_SHADER_SOURCE = R"(
#version 330 core
out vec4 FragColor;     // 출력 색상

void main() {
    // 모든 픽셀을 주황색으로 (R=1.0, G=0.5, B=0.2, A=1.0)
    FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);
}
)";
}
```

### Step 2: Application 클래스 수정

OpenGL 객체들을 멤버 변수로 추가:

```cpp
class Application final {
private:
    std::unique_ptr<gl::Window> m_window;
    bool m_useAlternateColor = false;
    
    GLuint m_shaderProgram = 0;  // 셰이더 프로그램 ID
    GLuint m_vao = 0;            // Vertex Array Object
    GLuint m_vbo = 0;            // Vertex Buffer Object
    bool m_wireframeMode = false; // 와이어프레임 모드 플래그
};
```

### Step 3: 셰이더 컴파일 함수

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
            std::println(stderr, "{} shader compilation failed:\n{}", 
                        shaderName, infoLog);
            
            glDeleteShader(shader);
            return 0;
        }
        
        return shader;
    }
    
    bool setupShaders() {
        std::println("Setting up shaders...");
        
        // 1. 버텍스 셰이더 컴파일
        const GLuint vertexShader = compileShader(GL_VERTEX_SHADER, 
                                                  VERTEX_SHADER_SOURCE);
        if (!vertexShader) {
            return false;
        }
        
        // 2. 프래그먼트 셰이더 컴파일
        const GLuint fragmentShader = compileShader(GL_FRAGMENT_SHADER, 
                                                    FRAGMENT_SHADER_SOURCE);
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
};
```

### Step 4: 삼각형 데이터 설정

```cpp
class Application final {
private:
    bool setupTriangle() {
        std::println("Setting up triangle...");
        
        // 삼각형 3개 정점 (NDC 좌표계: -1~1)
        constexpr float vertices[] = {
            // x      y      z
            -0.5f, -0.5f, 0.0f,  // 왼쪽 아래
             0.5f, -0.5f, 0.0f,  // 오른쪽 아래  
             0.0f,  0.5f, 0.0f   // 위 중앙
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
                    GL_STATIC_DRAW  // 데이터가 자주 변경되지 않음
        );
        
        // 4. 버텍스 속성 포인터 설정
        glVertexAttribPointer(
            0,                       // 속성 번호 (layout location = 0)
            3,                       // 크기 (x, y, z = 3개)
            GL_FLOAT,               // 데이터 타입
            GL_FALSE,               // 정규화 여부
            3 * sizeof(float),      // stride (다음 정점까지 간격)
            static_cast<void*>(nullptr)  // 버퍼에서 시작 위치
        );
        
        // 5. 버텍스 속성 활성화
        glEnableVertexAttribArray(0);
        
        // 6. 바인딩 해제 (안전을 위해)
        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glBindVertexArray(0);
        
        std::println("Triangle VAO/VBO created successfully!");
        return true;
    }
};
```

### Step 5: Application 생성자 및 소멸자

```cpp
class Application final {
public:
    [[nodiscard]] static std::expected<Application, gl::Error> 
    create() noexcept {
        // Window 생성
        auto windowResult = gl::Window::create(
            {.width = 800, .height = 600, .title = "OpenGL First Triangle"}
        );
        
        if (!windowResult.has_value()) {
            return std::unexpected(windowResult.error());
        }
        
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
    
    // 복사 방지
    Application(const Application&) = delete;
    Application& operator=(const Application&) = delete;
    
    // 이동 생성자
    Application(Application&& other) noexcept
        : m_window(std::move(other.m_window))
        , m_useAlternateColor(other.m_useAlternateColor)
        , m_shaderProgram(std::exchange(other.m_shaderProgram, 0))
        , m_vao(std::exchange(other.m_vao, 0))
        , m_vbo(std::exchange(other.m_vbo, 0))
        , m_wireframeMode(other.m_wireframeMode) {
    }
    
    Application& operator=(Application&& other) noexcept {
        if (this != &other) {
            cleanup();
            
            m_window = std::move(other.m_window);
            m_useAlternateColor = other.m_useAlternateColor;
            m_shaderProgram = std::exchange(other.m_shaderProgram, 0);
            m_vao = std::exchange(other.m_vao, 0);
            m_vbo = std::exchange(other.m_vbo, 0);
            m_wireframeMode = other.m_wireframeMode;
        }
        return *this;
    }
    
private:
    void cleanup() const noexcept {
        // OpenGL 객체 정리 (RAII로 개선 예정)
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
};
```

### Step 6: 렌더링 함수 수정

```cpp
void render() const {
    const Color& color = m_useAlternateColor 
                        ? Color{0.3f, 0.2f, 0.3f, 1.0f}
                        : BACKGROUND_COLOR;
    
    glClearColor(color.r, color.g, color.b, color.a);
    glClear(GL_COLOR_BUFFER_BIT);
    
    // 1. 셰이더 프로그램 활성화
    glUseProgram(m_shaderProgram);
    
    // 2. VAO 바인딩 (모든 정점 속성 설정이 자동 적용)
    glBindVertexArray(m_vao);
    
    // 3. 그리기 명령
    glDrawArrays(
        GL_TRIANGLES,  // Primitive 타입
        0,             // 시작 인덱스
        3              // 정점 개수
    );
    
    // 4. VAO 언바인딩 (선택사항)
    glBindVertexArray(0);
}
```

### Step 7: 와이어프레임 모드 추가

```cpp
void handleKeyPress(const int key) {
    switch (key) {
        case GLFW_KEY_ESCAPE:
            m_window->requestClose();
            break;
            
        case GLFW_KEY_SPACE:
            m_useAlternateColor = !m_useAlternateColor;
            std::println("Background color mode: {}", 
                        m_useAlternateColor ? "Alternate" : "Default");
            break;
            
        case GLFW_KEY_I:
            printInfo();
            break;
            
        case GLFW_KEY_W: {
            m_wireframeMode = !m_wireframeMode;
            glPolygonMode(GL_FRONT_AND_BACK, 
                         m_wireframeMode ? GL_LINE : GL_FILL);
            std::println("Wireframe mode: {}", 
                        m_wireframeMode ? "ON" : "OFF");
        }
        break;
        
        default:
            break;
    }
}
```

## 🔬 실행 및 테스트

### 실행 결과
- 800x600 윈도우 생성
- 주황색 삼각형 렌더링
- **ESC**: 프로그램 종료
- **Space**: 배경색 변경
- **I**: 정보 출력
- **W**: 와이어프레임 모드 토글
- FPS 콘솔 출력

<video width="100%" controls>
  <source src="/assets/videos/OpenGL/First Triangle.mp4" type="video/mp4">
  첫 삼각형 렌더링 데모
</video>

## 📊 현재 코드의 문제점

- ❌ 셰이더 소스가 하드코딩됨
- ❌ OpenGL 객체 수동 관리
- ❌ 재사용성 없음
- ❌ 에러 처리가 기본적
- ❌ 모든 코드가 한 파일에 몰려있음

## 💬 트러블슈팅

### 문제 1: 삼각형이 안 보일 때

**체크리스트:**
1. 셰이더 컴파일 에러 확인
2. VAO/VBO 바인딩 순서 확인
3. NDC 좌표 범위 (-1 ~ 1) 확인
4. glDrawArrays 호출 확인

### 문제 2: Window 클래스 이동 생성자 누락

```cpp
Window::Window(Window&& other) noexcept
    : m_window(std::move(other.m_window))
    , m_context(std::move(other.m_context))
    , m_resizeCallback(std::move(other.m_resizeCallback))
    , m_keyCallback(std::move(other.m_keyCallback)) {
    if (m_window) {
        glfwSetWindowUserPointer(m_window.get(), this);
    }
}

Window& Window::operator=(Window&& other) noexcept {
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

### 문제 3: 링킹 에러

셰이더 링킹이 실패하면 보통 다음 이유들 때문입니다:
- Vertex Shader의 out 변수와 Fragment Shader의 in 변수 불일치
- GLSL 버전 불일치
- 변수명 오타

## 🎓 핵심 정리

### 이번 챕터에서 배운 것
1. **셰이더 프로그램**: GPU에서 실행되는 코드
2. **VBO**: GPU 메모리에 정점 데이터 저장
3. **VAO**: VBO 데이터의 해석 방법 저장
4. **렌더링 파이프라인**: 정점 → 래스터라이제이션 → 픽셀
5. **NDC**: -1 ~ 1 범위의 정규화된 좌표계

### 기억할 포인트
- VAO는 VBO의 "설명서" 역할
- 셰이더는 병렬로 실행됨 (정점마다, 픽셀마다)
- OpenGL은 상태 머신 (바인딩이 중요)

## 🚀 다음 단계

다음 포스트에서 다룰 내용:
- **Shader 클래스 설계**: 재사용 가능한 셰이더 관리
- **RAII로 자동 리소스 관리**: 수동 cleanup 제거
- **Uniform 변수 지원**: 동적인 값 전달
- **파일에서 셰이더 로드**: 하드코딩 제거

## 📚 참고 자료

- [OpenGL Documentation](https://www.opengl.org/documentation/)
- [Learn OpenGL - Hello Triangle](https://learnopengl.com/Getting-started/Hello-Triangle)
- [OpenGL Wiki - Vertex Specification](https://www.khronos.org/opengl/wiki/Vertex_Specification)

---

**질문이나 피드백이 있으신가요?** 댓글로 남겨주세요! 다음 포스트에서는 이 하드코딩된 코드를 깔끔한 클래스로 리팩토링하는 과정을 다룹니다.
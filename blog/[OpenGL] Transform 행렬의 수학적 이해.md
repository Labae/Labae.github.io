---
layout: post
title: "[OpenGL] Transform 행렬의 수학적 이해"
date: 2025-09-07
categories: [Graphics, OpenGL, Math]
tags: [matrix, linear-algebra, transform, mathematics]
math: true
---

## 📐 선형 변환과 행렬

### 2D 변환의 수학적 표현

2D 공간에서 점 $P(x, y)$를 변환하는 기본 연산들을 살펴보겠습니다.

#### 1. Scale (크기 변환)

$$
\begin{bmatrix}
x' \\
y'
\end{bmatrix}
=
\begin{bmatrix}
s_x & 0 \\
0 & s_y
\end{bmatrix}
\begin{bmatrix}
x \\
y
\end{bmatrix}
$$

#### 2. Rotation (회전)

각도 $\theta$만큼 반시계 방향 회전:

$$
\begin{bmatrix}
x' \\
y'
\end{bmatrix}
=
\begin{bmatrix}
\cos\theta & -\sin\theta \\
\sin\theta & \cos\theta
\end{bmatrix}
\begin{bmatrix}
x \\
y
\end{bmatrix}
$$

#### 3. Translation (이동) - 문제 발생!

이동은 선형 변환이 아닙니다:

$$
\begin{bmatrix}
x' \\
y'
\end{bmatrix}
=
\begin{bmatrix}
x \\
y
\end{bmatrix}
+
\begin{bmatrix}
t_x \\
t_y
\end{bmatrix}
$$

덧셈이 포함되어 행렬 곱셈만으로 표현할 수 없습니다.

## 🎯 동차 좌표계 (Homogeneous Coordinates)

### 차원 확장의 마법

2D 점을 3D로 확장하여 모든 변환을 행렬 곱셈으로 통일합니다:

- 2D 점: $(x, y)$
- 3D 동차 좌표: $(x, y, w)$ where $w = 1$

### 3x3 변환 행렬

#### Scale 행렬
$$
S = \begin{bmatrix}
s_x & 0 & 0 \\
0 & s_y & 0 \\
0 & 0 & 1
\end{bmatrix}
$$

#### Rotation 행렬
$$
R = \begin{bmatrix}
\cos\theta & -\sin\theta & 0 \\
\sin\theta & \cos\theta & 0 \\
0 & 0 & 1
\end{bmatrix}
$$

#### Translation 행렬
$$
T = \begin{bmatrix}
1 & 0 & t_x \\
0 & 1 & t_y \\
0 & 0 & 1
\end{bmatrix}
$$

## 🔄 행렬 곱셈 순서의 중요성

### TRS vs SRT

변환 순서에 따라 결과가 완전히 달라집니다:

<video width="100%" controls>
  <source src="/assets/videos/OpenGL/trs-srt.mp4" type="video/mp4">
  TRS SRT 비교 영상
</video>

#### TRS 순서 (올바른 순서)
$$
M = T \cdot R \cdot S
$$

```cpp
// OpenGL 구현 예제
glm::mat4 createTransformMatrix(
    const glm::vec2& position,
    float rotation,
    const glm::vec2& scale
) {
    // 각 변환 행렬 생성
    glm::mat4 T = glm::translate(glm::mat4(1.0f), 
                                  glm::vec3(position, 0.0f));
    glm::mat4 R = glm::rotate(glm::mat4(1.0f), 
                               rotation, 
                               glm::vec3(0.0f, 0.0f, 1.0f));
    glm::mat4 S = glm::scale(glm::mat4(1.0f), 
                              glm::vec3(scale, 1.0f));
    
    // TRS 순서로 곱셈
    return T * R * S;  // 주의: 오른쪽에서 왼쪽으로 적용
}
```

#### 수학적 증명

점 $P$에 TRS 변환을 적용:

$$
P' = T \cdot R \cdot S \cdot P
$$

단계별 변환:
1. $P_1 = S \cdot P$ (원점에서 크기 조절)
2. $P_2 = R \cdot P_1$ (원점에서 회전)
3. $P' = T \cdot P_2$ (최종 위치로 이동)

### 잘못된 순서의 예

SRT 순서로 적용하면:

$$
P' = S \cdot R \cdot T \cdot P
$$

1. $P_1 = T \cdot P$ (먼저 이동)
2. $P_2 = R \cdot P_1$ (이동한 위치에서 회전 - 원점 기준 회전이 아님!)
3. $P' = S \cdot P_2$ (스케일이 이동 거리도 확대)

## 📊 OpenGL의 4x4 행렬

### 왜 2D에서도 4x4를 사용할까?

1. **3D와의 일관성**: 같은 파이프라인 사용
2. **투영 변환 지원**: Perspective division 가능
3. **최적화**: GPU는 4x4 행렬 연산에 최적화

### 2D를 위한 4x4 변환 행렬

$$
M_{4x4} = \begin{bmatrix}
\cos\theta \cdot s_x & -\sin\theta \cdot s_y & 0 & t_x \\
\sin\theta \cdot s_x & \cos\theta \cdot s_y & 0 & t_y \\
0 & 0 & 1 & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

## 💻 실제 구현 예제

### 완전한 Transform2D 구현

```cpp
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
#include <cmath>

class Transform2D {
private:
    glm::vec2 m_position{0.0f, 0.0f};
    float m_rotation{0.0f};  // 라디안
    glm::vec2 m_scale{1.0f, 1.0f};
    
    mutable glm::mat4 m_cachedMatrix{1.0f};
    mutable bool m_dirty{true};
    
public:
    // 개별 변환 행렬 생성
    glm::mat4 getTranslationMatrix() const {
        return glm::mat4(
            1.0f, 0.0f, 0.0f, 0.0f,
            0.0f, 1.0f, 0.0f, 0.0f,
            0.0f, 0.0f, 1.0f, 0.0f,
            m_position.x, m_position.y, 0.0f, 1.0f
        );
    }
    
    glm::mat4 getRotationMatrix() const {
        float c = std::cos(m_rotation);
        float s = std::sin(m_rotation);
        
        return glm::mat4(
            c,    s,    0.0f, 0.0f,
            -s,   c,    0.0f, 0.0f,
            0.0f, 0.0f, 1.0f, 0.0f,
            0.0f, 0.0f, 0.0f, 1.0f
        );
    }
    
    glm::mat4 getScaleMatrix() const {
        return glm::mat4(
            m_scale.x, 0.0f,      0.0f, 0.0f,
            0.0f,      m_scale.y, 0.0f, 0.0f,
            0.0f,      0.0f,      1.0f, 0.0f,
            0.0f,      0.0f,      0.0f, 1.0f
        );
    }
    
    // TRS 순서로 결합
    glm::mat4 getMatrix() const {
        if (m_dirty) {
            m_cachedMatrix = getTranslationMatrix() * 
                           getRotationMatrix() * 
                           getScaleMatrix();
            m_dirty = false;
        }
        return m_cachedMatrix;
    }
    
    // Setter with dirty flag
    void setPosition(const glm::vec2& pos) {
        m_position = pos;
        m_dirty = true;
    }
    
    void setRotation(float radians) {
        m_rotation = radians;
        m_dirty = true;
    }
    
    void setScale(const glm::vec2& scale) {
        m_scale = scale;
        m_dirty = true;
    }
};
```

### Shader에서 Transform 적용

```glsl
// vertex shader
#version 330 core
layout (location = 0) in vec3 aPos;

uniform mat4 u_Transform;

void main() {
    // 동차 좌표로 확장 (w = 1)
    vec4 position = vec4(aPos, 1.0);
    
    // 변환 행렬 적용
    gl_Position = u_Transform * position;
}
```

## 🎓 핵심 정리

### 행렬 곱셈 규칙

1. **결합 법칙**: $(AB)C = A(BC)$
2. **교환 법칙 성립 안함**: $AB \neq BA$
3. **역순 적용**: 오른쪽 행렬부터 적용

### 변환 조합 공식

복잡한 변환을 단순 변환의 조합으로:

$$
M_{total} = T_2 \cdot R_2 \cdot S_2 \cdot T_1 \cdot R_1 \cdot S_1
$$

### 역변환

각 변환의 역행렬:

- **Scale 역변환**: $S^{-1} = \text{diag}(1/s_x, 1/s_y, 1)$
- **Rotation 역변환**: $R^{-1} = R^T$ (전치 행렬)
- **Translation 역변환**: $T^{-1} = T(-t_x, -t_y)$

전체 역변환:
$$
M^{-1} = (TRS)^{-1} = S^{-1}R^{-1}T^{-1}
$$

## 📚 참고 자료

- [3Blue1Brown - Linear Transformations](https://www.youtube.com/watch?v=kYB8IZa5AuE)
- [Interactive Matrix Multiplication](http://matrixmultiplication.xyz/)
- [OpenGL Transformation](http://www.songho.ca/opengl/gl_transform.html)

---

### 💬 댓글

Transform 행렬의 수학적 개념에 대한 질문이 있다면 아래 댓글로 남겨주세요!
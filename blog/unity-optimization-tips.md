---
layout: post
title: "Unity 모바일 게임 최적화 완벽 가이드(테스트 문서)"
date: 2024-01-15
categories: [Unity, Optimization, Mobile]
---

모바일 게임 개발에서 가장 중요한 것 중 하나는 **성능 최적화**입니다. 
Witch One 프로젝트를 진행하면서 적용했던 최적화 기법들을 공유합니다.

## 📊 최적화 전후 성능 비교

| 항목 | 최적화 전 | 최적화 후 | 개선율 |
|------|----------|----------|--------|
| FPS | 25-30 | 55-60 | 100% ↑ |
| Draw Calls | 250+ | 80 | 68% ↓ |
| 메모리 사용량 | 1.2GB | 650MB | 46% ↓ |
| 배터리 소모 | 높음 | 보통 | - |

## 1. Draw Call 최적화

### 📦 Static Batching
정적 오브젝트들을 하나의 메시로 결합하여 Draw Call을 줄입니다.

```csharp
// Inspector에서 Static 체크
GameObject.isStatic = true;

// 또는 코드에서 설정
StaticBatchingUtility.Combine(gameObject);
```

### 🎨 Dynamic Batching
작은 동적 오브젝트들을 자동으로 배칭합니다.

**조건:**
- 버텍스 300개 이하
- 동일한 머티리얼 사용
- 스케일이 uniform (x, y, z 동일)

## 2. 텍스처 최적화

### 압축 포맷 선택
```csharp
// Android
TextureImporter.textureCompression = TextureImporterCompression.CompressedHQ;
TextureImporter.SetPlatformTextureSettings("Android", 2048, TextureImporterFormat.ETC2_RGBA8);

// iOS
TextureImporter.SetPlatformTextureSettings("iOS", 2048, TextureImporterFormat.PVRTC_RGBA4);
```

### 텍스처 아틀라스 활용
여러 스프라이트를 하나의 텍스처로 결합:

```csharp
// Sprite Packer 사용
[System.Serializable]
public class AtlasSettings
{
    public string tag = "UI_Atlas";
    public int maxSize = 2048;
    public TextureFormat format = TextureFormat.RGBA32;
}
```

## 3. LOD (Level of Detail) 시스템

거리에 따라 다른 품질의 모델을 표시합니다.

```csharp
public class LODController : MonoBehaviour
{
    public Mesh[] lodMeshes;
    public float[] distances = { 10f, 25f, 50f };
    
    private MeshFilter meshFilter;
    private Camera mainCamera;
    
    void Start()
    {
        meshFilter = GetComponent<MeshFilter>();
        mainCamera = Camera.main;
    }
    
    void Update()
    {
        float distance = Vector3.Distance(transform.position, mainCamera.transform.position);
        
        for (int i = 0; i < distances.Length; i++)
        {
            if (distance < distances[i])
            {
                meshFilter.mesh = lodMeshes[i];
                break;
            }
        }
    }
}
```

## 4. 오브젝트 풀링

자주 생성/삭제되는 오브젝트를 미리 생성해두고 재사용합니다.

```csharp
public class ObjectPool<T> where T : MonoBehaviour
{
    private Queue<T> pool = new Queue<T>();
    private T prefab;
    private Transform parent;
    
    public ObjectPool(T prefab, int initialSize, Transform parent = null)
    {
        this.prefab = prefab;
        this.parent = parent;
        
        for (int i = 0; i < initialSize; i++)
        {
            T obj = GameObject.Instantiate(prefab, parent);
            obj.gameObject.SetActive(false);
            pool.Enqueue(obj);
        }
    }
    
    public T Get()
    {
        if (pool.Count > 0)
        {
            T obj = pool.Dequeue();
            obj.gameObject.SetActive(true);
            return obj;
        }
        
        return GameObject.Instantiate(prefab, parent);
    }
    
    public void Return(T obj)
    {
        obj.gameObject.SetActive(false);
        pool.Enqueue(obj);
    }
}
```

## 5. 프로파일러 활용 팁

### Unity Profiler 주요 체크 포인트
1. **CPU Usage**: 16ms 이하 유지 (60fps)
2. **GPU Usage**: 과도한 오버드로우 체크
3. **Memory**: 텍스처와 메시 메모리 사용량
4. **Rendering**: SetPass Calls 확인

### 프로파일링 코드
```csharp
using UnityEngine.Profiling;

public class PerformanceMonitor : MonoBehaviour
{
    void Update()
    {
        Profiler.BeginSample("MyCustomProcess");
        // 성능 측정하고 싶은 코드
        ExpensiveOperation();
        Profiler.EndSample();
    }
}
```

## 6. 셰이더 최적화

### 모바일용 간단한 셰이더
```hlsl
Shader "Mobile/SimpleDiffuse"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _Color ("Color", Color) = (1,1,1,1)
    }
    
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100
        
        Pass
        {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma multi_compile_fog
            
            #include "UnityCG.cginc"
            
            struct appdata
            {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };
            
            struct v2f
            {
                float2 uv : TEXCOORD0;
                UNITY_FOG_COORDS(1)
                float4 vertex : SV_POSITION;
            };
            
            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed4 _Color;
            
            v2f vert (appdata v)
            {
                v2f o;
                o.vertex = UnityObjectToClipPos(v.vertex);
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                UNITY_TRANSFER_FOG(o,o.vertex);
                return o;
            }
            
            fixed4 frag (v2f i) : SV_Target
            {
                fixed4 col = tex2D(_MainTex, i.uv) * _Color;
                UNITY_APPLY_FOG(i.fogCoord, col);
                return col;
            }
            ENDCG
        }
    }
}
```

## 💡 최적화 체크리스트

- [ ] Static Batching 활성화
- [ ] 텍스처 압축 포맷 설정
- [ ] LOD 시스템 구현
- [ ] 오브젝트 풀링 적용
- [ ] 불필요한 Update() 제거
- [ ] Raycast 최소화
- [ ] UI Canvas 분리
- [ ] 오클루전 컬링 설정
- [ ] 라이트맵 베이킹
- [ ] 품질 설정 조정

## 🎮 실제 적용 결과

Witch One 프로젝트에 이러한 최적화 기법들을 적용한 결과:
- 저사양 디바이스(Galaxy A32)에서도 안정적인 45fps 유지
- 30분 플레이 시 배터리 소모 20% → 12%로 감소
- 앱 크기 350MB → 180MB로 축소

## 마무리

모바일 게임 최적화는 끝이 없는 과정입니다. 
프로파일러를 자주 확인하고, 타겟 디바이스에서 실제 테스트하는 것이 중요합니다.

다음 포스트에서는 **Photon을 활용한 멀티플레이어 최적화**에 대해 다루겠습니다.

---

### 📚 참고 자료
- [Unity Mobile Optimization Guide](https://docs.unity3d.com/Manual/MobileOptimization.html)
- [ARM Mobile Studio](https://developer.arm.com/tools-anmdd-software/graphics-and-gaming/arm-mobile-studio)
- [Unity Profiler Documentation](https://docs.unity3d.com/Manual/Profiler.html)

### 💬 댓글
질문이나 추가 팁이 있다면 아래 댓글로 남겨주세요!
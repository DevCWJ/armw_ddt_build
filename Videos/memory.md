---
url: 'https://developers-apps-in-toss.toss.im/unity/optimization/runtime/memory.md'
---
# Unity WebGL 메모리 최적화 가이드

Unity WebGL에서 메모리 최적화는 성능과 안정성을 지키는 핵심이에요.\
앱인토스 미니앱에서는 제한된 메모리 안에서 게임을 효율적으로 실행하는 게 특히 중요해요.

***

## 메모리 관리 기본 원칙

### 1. 힙 메모리 관리

Unity WebGL은 고정 크기 힙을 사용합니다.

```c#
using UnityEngine;
using System.Collections;

public class MemoryManager : MonoBehaviour
{
    [Header("메모리 모니터링")]
    public bool enableMemoryMonitoring = true;
    private long lastGCMemory = 0;
    
    private void Start()
    {
        if (enableMemoryMonitoring)
        {
            StartCoroutine(MonitorMemory());
        }
    }
    
    private IEnumerator MonitorMemory()
    {
        while (true)
        {
            long currentMemory = System.GC.GetTotalMemory(false);
            
            if (currentMemory > lastGCMemory * 1.2f)
            {
                System.GC.Collect();
                Resources.UnloadUnusedAssets();
                lastGCMemory = System.GC.GetTotalMemory(true);
            }
            
            yield return new WaitForSeconds(5f);
        }
    }
    
    public void ForceCleanup()
    {
        Resources.UnloadUnusedAssets();
        System.GC.Collect();
        System.GC.WaitForPendingFinalizers();
        System.GC.Collect();
    }
}
```

### 2. 텍스처 메모리 최적화

```c#
using UnityEngine;

[System.Serializable]
public class TextureSettings
{
    [Header("품질 설정")]
    public int maxTextureSize = 512;
    public TextureFormat preferredFormat = TextureFormat.ASTC_4x4;
    public bool generateMipmaps = false;
    
    [Header("압축 설정")]
    public bool useTextureStreaming = true;
    public int memoryBudget = 64; // MB
}

public class TextureOptimizer : MonoBehaviour
{
    public TextureSettings settings;
    
    private void Start()
    {
        OptimizeTextures();
        SetupTextureStreaming();
    }
    
    private void OptimizeTextures()
    {
        QualitySettings.masterTextureLimit = 1; // 절반 해상도
        QualitySettings.globalTextureMipmapLimit = 1;
    }
    
    private void SetupTextureStreaming()
    {
        if (settings.useTextureStreaming)
        {
            QualitySettings.streamingMipmapsActive = true;
            QualitySettings.streamingMipmapsMemoryBudget = settings.memoryBudget;
        }
    }
}
```

***

## AppsInToss 플랫폼 통합

### 1. 메모리 상태 모니터링

```tsx

interface MemoryInfo {
    usedJSHeapSize: number;
    totalJSHeapSize: number;
    jsHeapSizeLimit: number;
}

class MemoryMonitor {
    private static instance: MemoryMonitor;
    private memoryThreshold: number = 0.8;
    
    public static getInstance(): MemoryMonitor {
        if (!MemoryMonitor.instance) {
            MemoryMonitor.instance = new MemoryMonitor();
        }
        return MemoryMonitor.instance;
    }
    
    public startMonitoring(): void {
        setInterval(() => {
            this.checkMemoryUsage();
        }, 3000);
    }
    
    private checkMemoryUsage(): void {
        const memInfo = this.getMemoryInfo();
        const usage = memInfo.usedJSHeapSize / memInfo.jsHeapSizeLimit;
        
        if (usage > this.memoryThreshold) {
            this.requestMemoryCleanup();
        }
    }
    
    private getMemoryInfo(): MemoryInfo {
        const performance = (window as any).performance;
        return performance.memory || {
            usedJSHeapSize: 0,
            totalJSHeapSize: 0,
            jsHeapSizeLimit: 0
        };
    }
    
    private requestMemoryCleanup(): void {
        // Unity에 메모리 정리 요청
        const unityInstance = (window as any).unityInstance;
        if (unityInstance) {
            unityInstance.SendMessage('MemoryManager', 'ForceCleanup', '');
        }
    }
}
```

### 2. Unity와 JavaScript 간 메모리 공유 최적화

```c#
using UnityEngine;
using System.Runtime.InteropServices;
using System.Text;

public class MemoryBridge : MonoBehaviour
{
    [DllImport("__Internal")]
    private static extern void SendMemoryData(string data);
    
    [DllImport("__Internal")]
    private static extern string GetMemoryInfo();
    
    private StringBuilder stringBuilder = new StringBuilder(1024);
    
    public void SendLargeData(byte[] data)
    {
        // 큰 데이터를 청크로 분할하여 전송
        int chunkSize = 1024;
        for (int i = 0; i < data.Length; i += chunkSize)
        {
            int currentChunkSize = Mathf.Min(chunkSize, data.Length - i);
            byte[] chunk = new byte[currentChunkSize];
            System.Array.Copy(data, i, chunk, 0, currentChunkSize);
            
            string base64Chunk = System.Convert.ToBase64String(chunk);
            SendMemoryData($"{{\"chunk\": \"{base64Chunk}\", \"index\": {i / chunkSize}}}");
        }
    }
    
    public void OptimizeStringOperations(string[] strings)
    {
        stringBuilder.Clear();
        
        foreach (string str in strings)
        {
            stringBuilder.Append(str);
            stringBuilder.Append(",");
        }
        
        if (stringBuilder.Length > 0)
        {
            stringBuilder.Length--; // 마지막 콤마 제거
        }
        
        SendMemoryData(stringBuilder.ToString());
    }
}
```

***

## 오브젝트 풀링

### 1. 범용 오브젝트 풀

```c#
using UnityEngine;
using System.Collections.Generic;

public class ObjectPool<T> : MonoBehaviour where T : MonoBehaviour
{
    [Header("풀 설정")]
    public T prefab;
    public int initialSize = 10;
    public int maxSize = 50;
    public bool allowGrowth = true;
    
    private Queue<T> pool = new Queue<T>();
    private HashSet<T> activeObjects = new HashSet<T>();
    
    private void Start()
    {
        InitializePool();
    }
    
    private void InitializePool()
    {
        for (int i = 0; i < initialSize; i++)
        {
            T obj = CreateNewObject();
            obj.gameObject.SetActive(false);
            pool.Enqueue(obj);
        }
    }
    
    private T CreateNewObject()
    {
        T obj = Instantiate(prefab, transform);
        return obj;
    }
    
    public T GetObject()
    {
        T obj;
        
        if (pool.Count > 0)
        {
            obj = pool.Dequeue();
        }
        else if (allowGrowth && activeObjects.Count < maxSize)
        {
            obj = CreateNewObject();
        }
        else
        {
            return null; // 풀이 가득 참
        }
        
        obj.gameObject.SetActive(true);
        activeObjects.Add(obj);
        return obj;
    }
    
    public void ReturnObject(T obj)
    {
        if (activeObjects.Contains(obj))
        {
            obj.gameObject.SetActive(false);
            activeObjects.Remove(obj);
            pool.Enqueue(obj);
        }
    }
    
    public void ClearPool()
    {
        while (pool.Count > 0)
        {
            T obj = pool.Dequeue();
            if (obj != null)
            {
                DestroyImmediate(obj.gameObject);
            }
        }
        
        foreach (T obj in activeObjects)
        {
            if (obj != null)
            {
                DestroyImmediate(obj.gameObject);
            }
        }
        
        activeObjects.Clear();
    }
}

// 사용 예제
public class BulletPoolManager : MonoBehaviour
{
    public ObjectPool<Bullet> bulletPool;
    
    public void FireBullet(Vector3 position, Vector3 direction)
    {
        Bullet bullet = bulletPool.GetObject();
        if (bullet != null)
        {
            bullet.Initialize(position, direction, () => {
                bulletPool.ReturnObject(bullet);
            });
        }
    }
}
```

### 2. 파티클 시스템 풀링

```c#
using UnityEngine;
using System.Collections;

public class ParticlePool : MonoBehaviour
{
    [Header("파티클 설정")]
    public ParticleSystem particlePrefab;
    public int poolSize = 20;
    
    private ParticleSystem[] particles;
    private int currentIndex = 0;
    
    private void Start()
    {
        InitializeParticlePool();
    }
    
    private void InitializeParticlePool()
    {
        particles = new ParticleSystem[poolSize];
        
        for (int i = 0; i < poolSize; i++)
        {
            particles[i] = Instantiate(particlePrefab, transform);
            particles[i].gameObject.SetActive(false);
        }
    }
    
    public ParticleSystem GetParticle()
    {
        ParticleSystem particle = particles[currentIndex];
        currentIndex = (currentIndex + 1) % poolSize;
        
        if (particle.isPlaying)
        {
            particle.Stop();
            particle.Clear();
        }
        
        particle.gameObject.SetActive(true);
        return particle;
    }
    
    public void PlayEffect(Vector3 position, ParticleSystem.MainModule mainSettings = default)
    {
        ParticleSystem particle = GetParticle();
        particle.transform.position = position;
        
        if (mainSettings.startColor.color != Color.clear)
        {
            var main = particle.main;
            main.startColor = mainSettings.startColor;
        }
        
        particle.Play();
        StartCoroutine(ReturnToPool(particle));
    }
    
    private IEnumerator ReturnToPool(ParticleSystem particle)
    {
        yield return new WaitUntil(() => !particle.isPlaying);
        particle.gameObject.SetActive(false);
    }
}
```

***

## 스마트 가비지 컬렉션

### 1. 적응형 GC 관리

```c#
using UnityEngine;
using System.Collections;

public class SmartGarbageCollector : MonoBehaviour
{
    [Header("GC 설정")]
    public float gcInterval = 10f;
    public float memoryThreshold = 0.75f;
    public bool adaptiveGC = true;
    
    private float lastGCTime;
    private long baselineMemory;
    private int framesSinceLastGC;
    
    private void Start()
    {
        baselineMemory = System.GC.GetTotalMemory(true);
        lastGCTime = Time.time;
        
        if (adaptiveGC)
        {
            StartCoroutine(AdaptiveGCRoutine());
        }
        else
        {
            StartCoroutine(RegularGCRoutine());
        }
    }
    
    private IEnumerator AdaptiveGCRoutine()
    {
        while (true)
        {
            framesSinceLastGC++;
            
            if (ShouldRunGC())
            {
                RunGarbageCollection();
            }
            
            yield return new WaitForEndOfFrame();
        }
    }
    
    private IEnumerator RegularGCRoutine()
    {
        while (true)
        {
            yield return new WaitForSeconds(gcInterval);
            RunGarbageCollection();
        }
    }
    
    private bool ShouldRunGC()
    {
        // 시간 기반 조건
        if (Time.time - lastGCTime > gcInterval)
            return true;
            
        // 메모리 사용량 기반 조건
        long currentMemory = System.GC.GetTotalMemory(false);
        float memoryGrowth = (float)currentMemory / baselineMemory;
        
        if (memoryGrowth > 1f + memoryThreshold)
            return true;
            
        // 프레임 드롭 기반 조건
        if (Time.deltaTime > 1f / 30f && framesSinceLastGC > 300)
            return true;
            
        return false;
    }
    
    private void RunGarbageCollection()
    {
        long memoryBefore = System.GC.GetTotalMemory(false);
        
        Resources.UnloadUnusedAssets();
        System.GC.Collect();
        System.GC.WaitForPendingFinalizers();
        System.GC.Collect();
        
        long memoryAfter = System.GC.GetTotalMemory(true);
        
        Debug.Log($"GC 실행: {(memoryBefore - memoryAfter) / 1024 / 1024}MB 해제됨");
        
        lastGCTime = Time.time;
        framesSinceLastGC = 0;
        
        if (baselineMemory == 0 || memoryAfter < baselineMemory)
        {
            baselineMemory = memoryAfter;
        }
    }
}
```

***

## 메모리 프로파일링 도구

### 1. 실시간 메모리 분석기

```c#
using UnityEngine;
using System.Collections.Generic;
using System.Text;

public class MemoryProfiler : MonoBehaviour
{
    [Header("프로파일링 설정")]
    public bool enableProfiling = true;
    public float updateInterval = 1f;
    public int maxSamples = 60;
    
    private Queue<MemorySample> samples = new Queue<MemorySample>();
    private StringBuilder logBuilder = new StringBuilder();
    
    [System.Serializable]
    public struct MemorySample
    {
        public float timestamp;
        public long totalMemory;
        public long textureMemory;
        public long meshMemory;
        public long audioMemory;
    }
    
    private void Start()
    {
        if (enableProfiling)
        {
            InvokeRepeating(nameof(TakeSample), 0f, updateInterval);
        }
    }
    
    private void TakeSample()
    {
        MemorySample sample = new MemorySample
        {
            timestamp = Time.time,
            totalMemory = System.GC.GetTotalMemory(false),
            textureMemory = GetTextureMemoryUsage(),
            meshMemory = GetMeshMemoryUsage(),
            audioMemory = GetAudioMemoryUsage()
        };
        
        samples.Enqueue(sample);
        
        if (samples.Count > maxSamples)
        {
            samples.Dequeue();
        }
        
        // AppsInToss 플랫폼에 메모리 정보 전송
        SendMemoryDataToAppsInToss(sample);
    }
    
    private long GetTextureMemoryUsage()
    {
        long totalMemory = 0;
        Texture[] textures = Resources.FindObjectsOfTypeAll<Texture>();
        
        foreach (Texture texture in textures)
        {
            if (texture is Texture2D tex2D)
            {
                totalMemory += UnityEngine.Profiling.Profiler.GetRuntimeMemorySizeLong(tex2D);
            }
        }
        
        return totalMemory;
    }
    
    private long GetMeshMemoryUsage()
    {
        long totalMemory = 0;
        Mesh[] meshes = Resources.FindObjectsOfTypeAll<Mesh>();
        
        foreach (Mesh mesh in meshes)
        {
            totalMemory += UnityEngine.Profiling.Profiler.GetRuntimeMemorySizeLong(mesh);
        }
        
        return totalMemory;
    }
    
    private long GetAudioMemoryUsage()
    {
        long totalMemory = 0;
        AudioClip[] clips = Resources.FindObjectsOfTypeAll<AudioClip>();
        
        foreach (AudioClip clip in clips)
        {
            totalMemory += UnityEngine.Profiling.Profiler.GetRuntimeMemorySizeLong(clip);
        }
        
        return totalMemory;
    }
    
    private void SendMemoryDataToAppsInToss(MemorySample sample)
    {
        logBuilder.Clear();
        logBuilder.Append("{");
        logBuilder.AppendFormat("\"timestamp\":{0},", sample.timestamp);
        logBuilder.AppendFormat("\"totalMemory\":{0},", sample.totalMemory);
        logBuilder.AppendFormat("\"textureMemory\":{0},", sample.textureMemory);
        logBuilder.AppendFormat("\"meshMemory\":{0},", sample.meshMemory);
        logBuilder.AppendFormat("\"audioMemory\":{0}", sample.audioMemory);
        logBuilder.Append("}");
        
        Application.ExternalCall("SendMemoryDataToAppsInToss", logBuilder.ToString());
    }
    
    public string GetMemoryReport()
    {
        if (samples.Count == 0) return "메모리 샘플이 없습니다.";
        
        logBuilder.Clear();
        logBuilder.AppendLine("=== 메모리 사용량 보고서 ===");
        
        MemorySample latest = samples.ToArray()[samples.Count - 1];
        logBuilder.AppendFormat("총 메모리: {0:F2} MB\n", latest.totalMemory / 1024f / 1024f);
        logBuilder.AppendFormat("텍스처 메모리: {0:F2} MB\n", latest.textureMemory / 1024f / 1024f);
        logBuilder.AppendFormat("메시 메모리: {0:F2} MB\n", latest.meshMemory / 1024f / 1024f);
        logBuilder.AppendFormat("오디오 메모리: {0:F2} MB\n", latest.audioMemory / 1024f / 1024f);
        
        return logBuilder.ToString();
    }
}
```

***

## 트러블 슈팅

### 일반적인 메모리 문제들

1. 메모리 누수

* 이벤트 핸들러 해제 누락
* 코루틴 정리 누락
* 순환 참조

2. 과도한 메모리 할당

* string concatenation
* 불필요한 new 연산
* 큰 배열 생성

3. 텍스처 메모리 문제

* 압축되지 않은 텍스처
* 불필요한 밉맵
* 잘못된 텍스처 포맷

### 해결 방법

```c#
// 메모리 누수 방지
public class ProperCleanup : MonoBehaviour
{
    private System.Action<int> scoreChanged;
    
    private void OnEnable()
    {
        GameManager.OnScoreChanged += HandleScoreChanged;
    }
    
    private void OnDisable()
    {
        GameManager.OnScoreChanged -= HandleScoreChanged; // 중요!
    }
    
    private void HandleScoreChanged(int newScore)
    {
        // 처리 로직
    }
}

// 효율적인 문자열 처리
public class StringOptimization : MonoBehaviour
{
    private StringBuilder stringBuilder = new StringBuilder(256);
    
    public string CreateFormattedString(int value1, float value2)
    {
        stringBuilder.Clear();
        stringBuilder.AppendFormat("값1: {0}, 값2: {1:F2}", value1, value2);
        return stringBuilder.ToString();
    }
}
```

***

## 베스트 프랙티스

1. 정기적인 메모리 모니터링
2. 적절한 오브젝트 풀링 사용
3. 불필요한 할당 최소화
4. 리소스 해제 자동화
5. AppsInToss 플랫폼 특성 고려

이 가이드를 통해 Unity WebGL 게임의 메모리 효율성을 크게 향상시킬 수 있어요.

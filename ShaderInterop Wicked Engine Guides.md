# WickedEngine ShaderInterop Technical Documentation
## Техническая документация по структурам данных CPU-GPU

> **Назначение**: Эта документация описывает структуры данных, используемые для передачи параметров между CPU (игровой код на C++) и GPU (шейдеры HLSL) в WickedEngine. Особое внимание уделено погодной системе и связанным компонентам.

---

## 1. БАЗОВЫЕ КОНЦЕПЦИИ

### 1.1 Что такое ShaderInterop файлы?

ShaderInterop (Shader Interoperability) файлы — это заголовочные файлы, которые:
- Включаются **И** в C++ код (CPU-сторона)
- Включаются **И** в HLSL шейдеры (GPU-сторона)
- Обеспечивают **идентичную** структуру данных на обеих сторонах
- Используют макросы для адаптации под разные языки

### 1.2 Ключевые макросы

```cpp
// В C++:
#define CBUFFER(name, slot) static const int CB_GETBINDSLOT(name) = slot; struct alignas(16) name
#define alignas(x) alignas(x)

// В HLSL:
#define CBUFFER(name, slot) cbuffer name : register(PASTE(b, slot))
#define alignas(x) // игнорируется
```

### 1.3 Alignment Rules (Правила выравнивания)

**КРИТИЧЕСКИ ВАЖНО**: Все структуры, передаваемые в GPU через constant buffers, должны быть выровнены по 16 байт!

```cpp
struct alignas(16) MyData {  // alignas(16) - обязательно!
    float3 position;   // 12 bytes
    float padding;     // 4 bytes  <- дополнение до 16
};
```

---

## 2. ПОГОДНАЯ СИСТЕМА (ShaderInterop_Weather.h)

### 2.1 ShaderWeather - Главная структура погоды

Это **центральная** структура, содержащая все параметры погоды и атмосферы.

```cpp
struct alignas(16) ShaderWeather
{
    // === ОСВЕЩЕНИЕ ===
    uint2 sun_direction;              // packed half3 (направление солнца)
    uint2 sun_color;                  // packed half3 (цвет солнца)
    
    uint2 ambient;                    // packed half3 (ambient освещение)
    uint most_important_light_index;  // индекс главного источника света
    float stars;                      // количество звезд (0 = выключены)
    
    uint2 horizon;                    // packed half3 (цвет горизонта)
    uint2 zenith;                     // packed half3 (цвет зенита)
    
    float4 stars_rotation;            // quaternion (поворот звезд)
    float3 padding_stars;
    
    // === НЕБО ===
    float sky_rotation_sin;           // sin угла поворота неба
    float sky_rotation_cos;           // cos угла поворота неба
    float sky_exposure;               // экспозиция неба
    float rain_amount;                // количество дождя [0-1]
    
    // === ДОЖДЬ ===
    float rain_length;                // длина капель дождя
    float rain_speed;                 // скорость падения
    float rain_scale;                 // масштаб частиц дождя
    float rain_splash_scale;          // масштаб всплесков
    float4 rain_color;                // цвет дождя
    
    // === ПОДСИСТЕМЫ ===
    ShaderFog fog;                           // туман
    ShaderWind wind;                         // ветер
    ShaderOcean ocean;                       // океан
    AtmosphereParameters atmosphere;         // атмосфера
    VolumetricCloudParameters volumetric_clouds; // облака
};
```

**Использование в коде**:
```cpp
// CPU (C++):
ShaderWeather weather;
weather.rain_amount = ClimateSystem::GetPrecipitation(); // 0.0-1.0
weather.rain_color = XMFLOAT4(0.7f, 0.8f, 0.9f, 1.0f);
weather.rain_length = 1.0f;
weather.rain_speed = 50.0f;

// GPU (HLSL) - автоматически доступно через constant buffer
float rainIntensity = xWeather.rain_amount;
```

---

### 2.2 ShaderFog - Туман

```cpp
struct alignas(16) ShaderFog
{
    float start;         // начало тумана (расстояние от камеры)
    float density;       // плотность тумана
    float height_start;  // высота начала тумана (мировые координаты)
    float height_end;    // высота конца тумана
};
```

**Пример связи с климатом**:
```cpp
ShaderFog fog;
fog.density = climate.GetHumidity() * 0.5f; // туман зависит от влажности
fog.height_start = 0.0f;
fog.height_end = 500.0f;
```

---

### 2.3 ShaderWind - Ветер

```cpp
struct alignas(16) ShaderWind
{
    float3 direction;    // направление ветра (нормализованный вектор)
    float speed;         // скорость ветра
    
    float wavesize;      // размер волн на растительности
    float randomness;    // случайность ветра [0-1]
    float padding0;
    float padding1;
};
```

**Интеграция с ClimateSystem**:
```cpp
ShaderWind wind;
wind.direction = climateSystem.GetWindDirection(); // XMFLOAT3 из ClimateSystem
wind.speed = climateSystem.GetClimate().wind * 50.0f; // [0-1] -> m/s
wind.wavesize = 1.0f;
wind.randomness = 0.3f;
```

---

### 2.4 ShaderOcean - Океан/Вода

```cpp
struct alignas(16) ShaderOcean
{
    float4 water_color;           // цвет воды
    float4 extinction_color;      // цвет затухания (глубокая вода)
    float water_height;           // высота уровня воды
    float patch_size_rcp;         // 1.0 / размер патча
    int texture_displacementmap;  // индекс текстуры смещения
    int texture_gradientmap;      // индекс текстуры градиента
    
    bool IsValid() { return texture_displacementmap >= 0; }
};
```

---

### 2.5 AtmosphereParameters - Атмосферное рассеяние

Моделирует физически корректное рассеяние света в атмосфере (Rayleigh + Mie scattering).

```cpp
struct alignas(16) AtmosphereParameters
{
    // === ГЕОМЕТРИЯ ПЛАНЕТЫ ===
    float bottomRadius;    // радиус планеты (от центра до земли), км
    float topRadius;       // радиус атмосферы (от центра до края), км
    float3 planetCenter;   // центр планеты в мировых координатах
    
    // === RAYLEIGH SCATTERING (синий свет неба) ===
    float rayleighDensityExpScale;  // масштаб экспоненциального распределения
    float3 rayleighScattering;      // коэффициенты рассеяния RGB
    
    // === MIE SCATTERING (дымка, туман) ===
    float mieDensityExpScale;       // масштаб распределения
    float3 mieScattering;           // коэффициенты рассеяния
    float3 mieExtinction;           // коэффициенты затухания
    float3 mieAbsorption;           // коэффициенты поглощения
    float miePhaseG;                // эксцентриситет фазовой функции [-1, 1]
    
    // === ABSORPTION (озоновый слой) ===
    float absorptionDensity0LayerWidth;
    float absorptionDensity0ConstantTerm;
    float absorptionDensity0LinearTerm;
    float absorptionDensity1ConstantTerm;
    float absorptionDensity1LinearTerm;
    float3 absorptionExtinction;
    
    // === РАЗНОЕ ===
    float3 groundAlbedo;              // альбедо земли (отражение)
    float2 rayMarchMinMaxSPP;         // min/max количество сэмплов
    float distanceSPPMaxInv;          // распределение сэмплов
    float aerialPerspectiveScale;     // масштаб aerial perspective
    
    // Конструктор с дефолтными значениями для Земли
    void init(
        float earthBottomRadius = 6360.0f,
        float earthTopRadius = 6460.0f,
        float earthRayleighScaleHeight = 8.0f,
        float earthMieScaleHeight = 1.2f
    );
};
```

**Как использовать**:
```cpp
AtmosphereParameters atmosphere;
atmosphere.init(); // дефолтные значения для Земли

// Изменение для более туманной атмосферы:
atmosphere.mieScattering = XMFLOAT3(0.006f, 0.006f, 0.006f); // больше дымки
```

---

### 2.6 VolumetricCloudParameters - Объемные облака

**Очень сложная структура** с двумя слоями облаков и множеством параметров.

```cpp
struct alignas(16) VolumetricCloudParameters
{
    // === ОСВЕЩЕНИЕ ===
    float beerPowder;               // эффект Beer's powder (рассеяние в облаках)
    float beerPowderPower;          // степень эффекта
    float ambientGroundMultiplier;  // сколько ambient света достигает низа облаков [0-1]
    float phaseG;                   // phase function параметр [-0.999, 0.999]
    float phaseG2;                  // второй phase параметр
    float phaseBlend;               // смешивание двух phase функций [0-1]
    
    float multiScatteringScattering;  // multi-scattering параметры
    float multiScatteringExtinction;
    float multiScatteringEccentricity;
    
    float shadowStepLength;         // длина шага для теней
    float horizonBlendAmount;       // смешивание на горизонте
    float horizonBlendPower;        // степень смешивания
    
    // === ГЕОМЕТРИЯ ===
    float cloudStartHeight;         // высота начала облаков (м)
    float cloudThickness;           // толщина слоя облаков (м)
    float animationMultiplier;      // множитель для анимации
    
    // === ДВА СЛОЯ ОБЛАКОВ ===
    VolumetricCloudLayer layerFirst;   // первый слой
    VolumetricCloudLayer layerSecond;  // второй слой
    
    // === ПРОИЗВОДИТЕЛЬНОСТЬ ===
    int maxStepCount;               // макс. шагов ray marching
    float maxMarchingDistance;      // макс. дистанция ray marching
    float inverseDistanceStepCount; // распределение шагов
    float renderDistance;           // макс. дистанция рендера
    float LODDistance;              // дистанция для LOD переключения
    float LODMin;                   // минимальный LOD
    float bigStepMarch;             // размер больших шагов
    float transmittanceThreshold;   // порог прозрачности (0.005)
    
    float shadowSampleCount;              // количество сэмплов для теней
    float groundContributionSampleCount;  // сэмплы для освещения от земли
    
    void init();  // инициализация с дефолтными значениями
};
```

#### VolumetricCloudLayer - Слой облаков

```cpp
struct alignas(16) VolumetricCloudLayer
{
    // === ОСВЕЩЕНИЕ ===
    float3 albedo;                  // альбедо облаков (обычно ~0.9)
    float3 extinctionCoefficient;   // коэффициент затухания света RGB
    
    // === МОДЕЛИРОВАНИЕ ФОРМЫ ===
    float skewAlongWindDirection;           // наклон по ветру
    float totalNoiseScale;                  // общий масштаб шума
    float curlScale;                        // масштаб curl шума
    float curlNoiseHeightFraction;          // высотная фракция curl
    float curlNoiseModifier;                // модификатор curl
    
    float detailScale;                      // масштаб детального шума
    float detailNoiseHeightFraction;        // высотная фракция деталей
    float detailNoiseModifier;              // модификатор деталей
    
    float skewAlongCoverageWindDirection;   // наклон по coverage ветру
    float weatherScale;                     // масштаб weather текстуры
    
    // === ПОКРЫТИЕ (COVERAGE) ===
    float coverageAmount;    // количество облаков [0-1]
    float coverageMinimum;   // минимальное покрытие [0-1]
    
    // === ТИПЫ ОБЛАКОВ ===
    float typeAmount;        // количество разных типов [0-1]
    float typeMinimum;       // минимум типов
    float rainAmount;        // дождевые облака [0-1]
    float rainMinimum;       // минимум дождя
    
    // Градиенты для типов облаков (4 точки: black, white, white, black)
    float4 gradientSmall;    // маленькие облака (кучевые)
    float4 gradientMedium;   // средние облака
    float4 gradientLarge;    // большие облака (кучево-дождевые)
    
    // Деформация "наковальни" (верх грозовых облаков)
    // (amountTop, offsetTop, amountBot, offsetBot)
    float4 anvilDeformationSmall;
    float4 anvilDeformationMedium;
    float4 anvilDeformationLarge;
    
    // === АНИМАЦИЯ ===
    float windSpeed;         // скорость ветра облаков
    float windAngle;         // направление ветра (радианы)
    float windUpAmount;      // вертикальное движение ветра
    float coverageWindSpeed; // скорость ветра для coverage
    float coverageWindAngle; // направление ветра для coverage
};
```

**Как связать с погодой**:
```cpp
// В твоем ClimateSystem:
void UpdateCloudCoverage(VolumetricCloudParameters& clouds) {
    float precipitation = m_climate.precipitation; // 0-1
    
    // Больше осадков = больше облаков
    clouds.layerFirst.coverageAmount = precipitation * 0.8f;
    
    // Дождевые облака появляются при высоких осадках
    if (precipitation > 0.6f) {
        clouds.layerFirst.rainAmount = (precipitation - 0.6f) / 0.4f;
    }
    
    // Ветер влияет на движение облаков
    clouds.layerFirst.windSpeed = m_climate.wind * 30.0f;
}
```

---

## 3. TERRAIN SYSTEM (ShaderInterop_Terrain.h)

### 3.1 ShaderTerrain

```cpp
struct alignas(16) ShaderTerrain
{
    float3 center_chunk_pos;  // позиция центрального чанка
    float chunk_size;         // размер чанка
    
    int chunk_buffer;         // индекс буфера чанков
    int chunk_buffer_range;   // диапазон буфера
    float min_height;         // минимальная высота террейна
    float max_height;         // максимальная высота
    
    void init();
};
```

### 3.2 ShaderTerrainChunk

```cpp
struct ShaderTerrainChunk
{
    int heightmap;      // индекс текстуры heightmap
    uint materialID;    // ID материала чанка
    
    void init();
};
```

---

## 4. PARTICLE SYSTEMS (ShaderInterop_EmittedParticle.h)

### 4.1 Флаги эмиттера

```cpp
static const uint EMITTER_OPTION_BIT_USE_RAIN_BLOCKER = 1 << 4;  // ВАЖНО ДЛЯ ДОЖДЯ!
```

**Использование для дождя**:
```cpp
EmittedParticleSystem rainEmitter;
rainEmitter.options |= EMITTER_OPTION_BIT_USE_RAIN_BLOCKER; // дождь не идет под крышами
```

### 4.2 EmittedParticleCB - Constant Buffer для частиц

```cpp
CBUFFER(EmittedParticleCB, CBSLOT_OTHER_EMITTEDPARTICLE)
{
    uint xEmitBufferOffset;
    float xEmitterRandomness;
    float xParticleRandomColorFactor;
    float xParticleSize;
    
    float xParticleScaling;
    float xParticleRotation;
    float xParticleRandomFactor;
    float xParticleNormalFactor;
    
    float xParticleLifeSpan;
    float xParticleLifeSpanRandomness;
    float xParticleMass;
    float xParticleMotionBlurAmount;
    
    uint xEmitterMaxParticleCount;
    uint xEmitterInstanceIndex;
    uint xEmitterMeshGeometryOffset;
    uint xEmitterMeshGeometryCount;
    
    uint2 xEmitterFramesXY;        // размер atlas текстуры (frames)
    uint xEmitterFrameCount;       // количество кадров анимации
    uint xEmitterFrameStart;       // начальный кадр
    
    float2 xEmitterTexMul;
    float xEmitterFrameRate;       // FPS анимации частиц
    uint xEmitterLayerMask;
    
    // SPH (Smoothed Particle Hydrodynamics) - для жидкостей
    float xSPH_h;           // smoothing radius
    float xSPH_h_rcp;       // 1.0 / smoothing radius
    float xSPH_h2;          // radius^2
    float xSPH_h3;          // radius^3
    
    float xSPH_poly6_constant;
    float xSPH_spiky_constant;
    float xSPH_visc_constant;
    float xSPH_K;           // pressure constant
    
    float xSPH_e;           // viscosity
    float xSPH_p0;          // reference density
    uint xEmitterOptions;
    float xEmitterFixedTimestep;
    
    float3 xParticleGravity;
    float xEmitterRestitution;  // упругость (bounce)
    
    float3 xParticleVelocity;   // начальная скорость
    float xParticleDrag;        // сопротивление воздуха
    
    ShaderTransform xEmitterBaseMeshUnormRemap;
};
```

---

## 5. RENDERER КОНСТАНТЫ (ShaderInterop_Renderer.h)

Хотя этот файл не был загружен, из других файлов видно, что он содержит:

```cpp
// Из других ShaderInterop файлов видно:
#include "ShaderInterop_Renderer.h"

// Типичные структуры:
// - ShaderCamera (параметры камеры)
// - ShaderScene (параметры сцены)
// - ShaderTransform (трансформации)
// - PrimitiveID (идентификация примитивов)
```

---

## 6. RAYTRACING (ShaderInterop_Raytracing.h)

```cpp
static const uint RAYTRACING_LAUNCH_BLOCKSIZE = 8;

CBUFFER(RaytracingCB, CBSLOT_RENDERER_TRACED)
{
    float2 xTracePixelOffset;      // смещение пикселя для AA
    uint xTraceSampleIndex;        // индекс сэмпла (для накопления)
    float xTraceAccumulationFactor; // фактор накопления
    uint2 xTraceResolution;        // разрешение трассировки
    float2 xTraceResolution_rcp;   // 1.0 / resolution
    uint4 xTraceUserData;          // пользовательские данные
};
```

---

## 7. GLOBAL ILLUMINATION SYSTEMS

### 7.1 DDGI (Dynamic Diffuse Global Illumination)

**Константы**:
```cpp
static const uint DDGI_MAX_RAYCOUNT = 512;
static const uint DDGI_COLOR_RESOLUTION = 8;
static const uint DDGI_DEPTH_RESOLUTION = 16;
static const float DDGI_KEEP_DISTANCE = 0.1f;
```

### 7.2 Surfel GI (Surface Element Global Illumination)

**Константы**:
```cpp
static const uint SURFEL_CAPACITY = 100000;
static const float SURFEL_MAX_RADIUS = 2;
static const uint SURFEL_RAY_BUDGET = 500000;
```

### 7.3 VXGI (Voxel Global Illumination)

```cpp
static const uint VXGI_CLIPMAP_COUNT = 6;  // 6 уровней voxel clipmaps

struct alignas(16) VoxelClipMap
{
    float3 center;      // центр clipmap в мировых координатах
    float voxelSize;    // размер одного вокселя
};
```

---

## 8. ПРАКТИЧЕСКИЕ ПРИМЕРЫ ИНТЕГРАЦИИ

### 8.1 Создание системы погодных эффектов

```cpp
// В твоем WeatherParticleSystem или GameState:

void UpdateWeatherShaderData(const ClimateSystem& climate) {
    wi::scene::Scene& scene = wi::scene::GetScene();
    wi::scene::WeatherComponent* weather = scene.weathers.GetComponent(m_weatherEntity);
    
    if (!weather) {
        wi::backlog::post("[WeatherSystem] UpdateWeatherShaderData: weather component not found!",
                          wi::backlog::LogLevel::Warning);
        return;
    }
    
    // === ДОЖДЬ ===
    if (climate.IsRaining()) {
        weather->rain_amount = climate.GetClimate().precipitation;
        weather->rain_length = 1.0f;
        weather->rain_speed = 50.0f;
        weather->rain_scale = 1.0f;
        weather->rain_splash_scale = 1.0f;
        weather->rain_color = XMFLOAT4(0.7f, 0.8f, 0.9f, 0.5f);
        
        if (g_debugLoggingEnabled) {
            wi::backlog::post("[WeatherSystem] Rain active: amount=" + 
                              std::to_string(weather->rain_amount));
        }
    } else {
        weather->rain_amount = 0.0f;
    }
    
    // === ВЕТЕР ===
    XMFLOAT3 windDir = climate.GetWindDirection();
    weather->wind.direction = windDir;
    weather->wind.speed = climate.GetClimate().wind * 50.0f;
    weather->wind.wavesize = 1.0f;
    weather->wind.randomness = 0.3f;
    
    // === ТУМАН ===
    weather->fog.density = climate.GetEffectiveHumidity() * 0.01f;
    weather->fog.height_start = 0.0f;
    weather->fog.height_end = 1000.0f;
    
    // === ОБЛАКА ===
    float cloudCoverage = climate.GetClimate().precipitation * 0.8f;
    weather->volumetric_clouds.layerFirst.coverageAmount = cloudCoverage;
    
    if (climate.GetClimate().precipitation > 0.6f) {
        // Дождевые облака
        float rainClouds = (climate.GetClimate().precipitation - 0.6f) / 0.4f;
        weather->volumetric_clouds.layerFirst.rainAmount = rainClouds;
    }
    
    // Направление ветра для облаков (в радианах)
    float windAngle = std::atan2(windDir.z, windDir.x);
    weather->volumetric_clouds.layerFirst.windAngle = windAngle;
    weather->volumetric_clouds.layerFirst.windSpeed = climate.GetClimate().wind * 30.0f;
    
    if (g_extendedDebugLoggingEnabled) {
        wi::backlog::post("[WeatherSystem] Updated shader data: " +
                          "rain=" + std::to_string(weather->rain_amount) +
                          ", wind_speed=" + std::to_string(weather->wind.speed) +
                          ", fog_density=" + std::to_string(weather->fog.density));
    }
}
```

### 8.2 Создание дождевого эмиттера

```cpp
void CreateRainEmitter(wi::ecs::Entity entity) {
    wi::scene::Scene& scene = wi::scene::GetScene();
    
    wi::EmittedParticleSystem* emitter = scene.emitters.GetComponent(entity);
    if (!emitter) {
        wi::backlog::post("[WeatherSystem] CreateRainEmitter: emitter component not found!",
                          wi::backlog::LogLevel::Warning);
        return;
    }
    
    // Базовые параметры дождя
    emitter->size = 0.02f;                    // размер капель
    emitter->normal_factor = 0.0f;            // не следуют нормалям поверхности
    emitter->randomness = 0.3f;               // небольшая случайность
    emitter->lifespan = 2.0f;                 // время жизни капли
    emitter->mass = 1.0f;                     // масса частицы
    emitter->motionBlurAmount = 1.0f;         // motion blur для скорости
    
    // Гравитация и сопротивление
    emitter->gravity = XMFLOAT3(0, -9.8f, 0); // стандартная гравитация
    emitter->drag = 0.98f;                    // небольшое сопротивление воздуха
    
    // ВАЖНО: Включаем rain blocker чтобы дождь не шел под крышами!
    emitter->SetOption(EMITTER_OPTION_BIT_USE_RAIN_BLOCKER, true);
    
    // Отключаем коллизии для производительности
    emitter->SetOption(EMITTER_OPTION_BIT_COLLIDERS_DISABLED, true);
    
    if (g_debugLoggingEnabled) {
        wi::backlog::post("[WeatherSystem] Rain emitter created successfully");
    }
}
```

### 8.3 Адаптивное обновление облаков

```cpp
void UpdateCloudParameters(VolumetricCloudParameters& clouds, 
                           const ClimateSystem& climate,
                           float deltaTime) {
    // Получаем текущее состояние климата
    const auto& state = climate.GetClimate();
    
    // === ДИНАМИЧЕСКОЕ ПОКРЫТИЕ ===
    // Целевое покрытие зависит от осадков
    float targetCoverage = state.precipitation * 0.8f;
    
    // Плавное изменение покрытия (не мгновенное!)
    float coverageBlendSpeed = 0.1f * deltaTime;
    clouds.layerFirst.coverageAmount = 
        std::lerp(clouds.layerFirst.coverageAmount, targetCoverage, coverageBlendSpeed);
    
    // === ТИПЫ ОБЛАКОВ ===
    // При высоких осадках появляются дождевые облака
    if (state.precipitation > 0.6f) {
        float rainIntensity = (state.precipitation - 0.6f) / 0.4f;
        clouds.layerFirst.rainAmount = 
            std::lerp(clouds.layerFirst.rainAmount, rainIntensity, coverageBlendSpeed);
        
        // Дождевые облака более темные
        clouds.layerFirst.extinctionCoefficient = XMFLOAT3(
            0.12f * rainIntensity,
            0.14f * rainIntensity, 
            0.15f * rainIntensity
        );
    } else {
        clouds.layerFirst.rainAmount = 
            std::lerp(clouds.layerFirst.rainAmount, 0.0f, coverageBlendSpeed);
    }
    
    // === ВЕТЕР ===
    XMFLOAT3 windDir = climate.GetWindDirection();
    float windAngle = std::atan2(windDir.z, windDir.x);
    
    clouds.layerFirst.windAngle = windAngle;
    clouds.layerFirst.windSpeed = state.wind * 30.0f; // масштабируем [0-1] в скорость
    clouds.layerFirst.windUpAmount = 0.5f;            // вертикальное движение
    
    // === ПЛОТНОСТЬ ОТ ВЛАЖНОСТИ ===
    // Больше влажности = более плотные облака
    float densityMultiplier = 1.0f + state.humidity * 0.5f;
    clouds.layerFirst.totalNoiseScale = 0.0006f * densityMultiplier;
    
    if (g_extendedDebugLoggingEnabled) {
        wi::backlog::post("[WeatherSystem] Cloud params: coverage=" + 
                          std::to_string(clouds.layerFirst.coverageAmount) +
                          ", rain=" + std::to_string(clouds.layerFirst.rainAmount) +
                          ", wind_speed=" + std::to_string(clouds.layerFirst.windSpeed));
    }
}
```

---

## 9. ВАЖНЫЕ ЗАМЕЧАНИЯ

### 9.1 Упаковка данных (Packing)

Многие поля используют **упаковку** (packing) для экономии памяти:

```cpp
uint2 sun_direction;  // packed half3 - три float16 упакованы в два uint32
```

**Функции упаковки/распаковки** (обычно в ShaderInterop.h):
```cpp
// C++:
uint2 PackHalf3(XMFLOAT3 value);
XMFLOAT3 UnpackHalf3(uint2 packed);

// HLSL:
uint2 pack_half3(float3 value);
float3 unpack_half3(uint2 packed);
```

### 9.2 Буферные слоты (CBSLOT)

Каждый constant buffer имеет свой **слот регистра**:

```cpp
#define CBSLOT_RENDERER_FRAME    0  // общий для всех
#define CBSLOT_RENDERER_CAMERA   1  // общий для всех
#define CBSLOT_OTHER_EMITTEDPARTICLE 4  // для частиц
```

**Важно**: Слоты должны быть уникальны, иначе будет конфликт!

### 9.3 Alignment Padding

Всегда добавляй **padding** для выравнивания структур:

```cpp
struct MyStruct {
    float3 position;  // 12 bytes
    float padding0;   // 4 bytes - НЕОБХОДИМ!
    // Итого: 16 bytes (aligned)
};
```

### 9.4 Инициализация

Многие структуры имеют метод `init()`:

```cpp
ShaderTerrain terrain;
terrain.init(); // ВСЕГДА вызывай перед использованием!
```

---

## 10. CHECKLIST ДЛЯ ИНТЕГРАЦИИ ПОГОДЫ

- [ ] Создать `wi::scene::WeatherComponent` в сцене
- [ ] Связать `ClimateSystem::UpdateClimate()` с `ShaderWeather`
- [ ] Настроить дождевой эмиттер с `EMITTER_OPTION_BIT_USE_RAIN_BLOCKER`
- [ ] Установить `ShaderWind` из `ClimateSystem::GetWindDirection()`
- [ ] Настроить `ShaderFog` в зависимости от влажности
- [ ] Обновлять `VolumetricCloudParameters` каждый кадр
- [ ] Добавить логирование с флагами `g_debugLoggingEnabled` и `g_extendedDebugLoggingEnabled`
- [ ] Проверить alignment всех структур (alignas(16))
- [ ] Плавно интерполировать изменения погоды (не мгновенно!)

---

## 11. ДОПОЛНИТЕЛЬНЫЕ РЕСУРСЫ

### Файлы для изучения в WickedEngine:
- `wiScene.h/.cpp` - WeatherComponent implementation
- `wiEmittedParticle.h/.cpp` - Particle system
- `wiRenderer.cpp` - Использование ShaderWeather в рендере
- `volumetricCloud.hlsl` - Шейдер облаков

### Связанные системы в твоем проекте:
- `CalendarSystem.h` - время суток влияет на освещение
- `ClimateSystem.h` - источник данных для погоды
- `WeatherParticleSystem.h` - создание дождя/снега
- `SkyboxSystem.h` - может быть связан с атмосферой

---

**Последнее обновление**: 2026-01-07  
**Версия документа**: 1.0  
**Автор**: Claude (Anthropic)

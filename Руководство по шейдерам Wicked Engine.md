# Руководство по созданию кастомных шейдеров в Wicked Engine

## 1. Архитектура шейдерной системы движка

### 1.1 Ключевые файлы

| Файл | Назначение |
|------|------------|
| `globals.hlsli` | Глобальные константы, структуры, функции доступа к данным кадра/камеры |
| `objectHF.hlsli` | Инфраструктура для object shaders (материалы, текстуры, вершины) |
| `ShaderInterop_Renderer.h` | Shared структуры между C++ и HLSL |
| `brdf.hlsli` | Функции освещения и BRDF |
| `lightingHF.hlsli` | Система освещения |

### 1.2 Как движок загружает данные вершин

**ВАЖНО:** Wicked Engine использует **bindless буферы**, а не классический Input Layout!

**ВАЖНО:** Wicked Engine допускает МАКСИМУМ 48 байт в PushConstants. Большее количество переменных передаются через CBUFFER (константный буффер)
```
Вершинынзагружаются через:
- load_instance() - данные инстанса
- load_geometry() - геометрия меша  
- push.geometryIndex, push.materialIndex - индексы через Push Constants
```

**Следствие:** При создании PSO для object shaders `psoDesc.il = nullptr`!

---

## 2. Макросы управления шейдерами

### 2.1 Layout макросы (определяют какие данные доступны)

```hlsl
#define OBJECTSHADER_LAYOUT_COMMON      // Стандартный набор: UV, нормали, тангенты
#define OBJECTSHADER_LAYOUT_SHADOW      // Для shadow pass
#define OBJECTSHADER_LAYOUT_SHADOW_TEX  // Shadow с текстурами
#define OBJECTSHADER_LAYOUT_PREPASS     // Для depth prepass
#define OBJECTSHADER_LAYOUT_PREPASS_TEX // Prepass с текстурами
```

### 2.2 Compile макросы (определяют что компилируется)

```hlsl
#define OBJECTSHADER_COMPILE_VS  // Компилировать Vertex Shader
#define OBJECTSHADER_COMPILE_PS  // Компилировать Pixel Shader
```

### 2.3 Feature макросы

```hlsl
#define DISABLE_ALPHATEST              // Отключить alpha test
#define DISABLE_TRANSPARENT_SHADOWMAP  // Отключить прозрачные тени
#define DISABLE_VOXELGI                // Отключить voxel GI
#define TRANSPARENT                    // Пометить как прозрачный
#define PLANARREFLECTION              // Включить planar reflections
```

---

## 3. Правила написания Pixel Shader

### 3.1 ГЛАВНОЕ ПРАВИЛО: Когда можно писать свою main()

```hlsl
// ✅ ПРАВИЛЬНО: НЕ определяем OBJECTSHADER_COMPILE_PS
#define OBJECTSHADER_LAYOUT_COMMON
#include "objectHF.hlsli"

[earlydepthstencil]
float4 main(PixelInput input) : SV_TARGET
{
    // Ваш код
}
```

```hlsl
// ❌ НЕПРАВИЛЬНО: Определяем OBJECTSHADER_COMPILE_PS
#define OBJECTSHADER_COMPILE_PS        // <-- objectHF создаст свою main()!
#define OBJECTSHADER_LAYOUT_COMMON
#include "objectHF.hlsli"

float4 main(PixelInput input) : SV_TARGET  // <-- КОНФЛИКТ! Две main()!
{
    // Ошибка компиляции
}
```

### 3.2 Шаблон кастомного Pixel Shader

```hlsl
// MyCustomShader_PS.hlsl

#define OBJECTSHADER_LAYOUT_COMMON
#include "objectHF.hlsli"

[earlydepthstencil]
float4 main(PixelInput input) : SV_TARGET
{
    // 1. Получаем материал
    ShaderMaterial material = GetMaterial();
    
    // 2. Получаем UV координаты
    float4 uvsets = input.GetUVSets();
    
    // 3. Mipmap feedback для texture streaming (ОБЯЗАТЕЛЬНО!)
    write_mipmap_feedback(push.materialIndex, ddx_coarse(uvsets), ddy_coarse(uvsets));
    
    // 4. Базовый цвет из текстуры
    float4 color = float4(1, 1, 1, 1);
    [branch]
    if (material.textures[BASECOLORMAP].IsValid())
    {
        color = material.textures[BASECOLORMAP].Sample(sampler_objectshader, uvsets);
    }
    
    // 5. Применяем vertex color
    color *= input.color;
    
    // 6. Ваши кастомные эффекты здесь...
    
    // 7. Возвращаем цвет (используем saturateMediump для half precision)
    return saturateMediump(color);
}
```

### 3.3 Доступные данные в PixelInput

```hlsl
PixelInput input;

input.pos           // SV_Position (screen space)
input.color         // Vertex color
input.uvsets        // UV координаты (через GetUVSets())
input.nor           // Нормаль (world space)
input.tan           // Тангент

// Методы:
input.GetUVSets()      // float4: xy = UV0, zw = UV1
input.GetPos3D()       // float3: world position
input.GetViewVector()  // float3: направление к камере
```

### 3.4 Доступные данные материала

```hlsl
ShaderMaterial material = GetMaterial();

material.GetBaseColor()    // float4: базовый цвет
material.GetEmissive()     // float3: emissive цвет
material.GetRoughness()    // float
material.GetMetalness()    // float

// Текстуры:
material.textures[BASECOLORMAP]   // Базовая текстура
material.textures[NORMALMAP]      // Normal map
material.textures[EMISSIVEMAP]    // Emissive map
// и другие...

// Проверка валидности текстуры:
if (material.textures[BASECOLORMAP].IsValid())
{
    float4 tex = material.textures[BASECOLORMAP].Sample(sampler_objectshader, uvsets);
}
```

---

## 4. Правила написания C++ кода

### 4.1 КРИТИЧНО: Время жизни объектов

**Проблема:** `PipelineStateDesc` хранит УКАЗАТЕЛИ на состояния. Если объекты на стеке — после выхода из функции указатели невалидны!

```cpp
// ❌ НЕПРАВИЛЬНО: Объекты на стеке функции
void CreateMyShader()
{
    wi::graphics::Shader ps;                    // Умрёт при выходе!
    wi::graphics::BlendState blendState;        // Умрёт при выходе!
    wi::graphics::RasterizerState rastState;    // Умрёт при выходе!
    wi::graphics::DepthStencilState dssState;   // Умрёт при выходе!
    
    wi::graphics::PipelineStateDesc psoDesc;
    psoDesc.ps = &ps;           // Указатель на локальную переменную!
    psoDesc.bs = &blendState;   // Указатель на локальную переменную!
    // ... PSO создастся, но потом все указатели станут мусором
}
```

```cpp
// ✅ ПРАВИЛЬНО: Глобальный namespace или static члены класса
namespace MyShaderGlobals
{
    wi::graphics::Shader pixelShader;
    wi::graphics::BlendState blendState;
    wi::graphics::RasterizerState rasterizerState;
    wi::graphics::DepthStencilState depthStencilState;
    wi::graphics::PipelineState pipelineState;
    bool initialized = false;
}

void CreateMyShader()
{
    if (MyShaderGlobals::initialized) return;
    
    // Загружаем в глобальные объекты
    wi::renderer::LoadShader(
        wi::graphics::ShaderStage::PS,
        MyShaderGlobals::pixelShader,  // <-- Живёт вечно
        "MyShader_PS.cso"
    );
    
    // Настраиваем глобальные состояния
    MyShaderGlobals::blendState = { ... };
    
    wi::graphics::PipelineStateDesc psoDesc;
    psoDesc.ps = &MyShaderGlobals::pixelShader;  // <-- Указатель валиден всегда
    psoDesc.bs = &MyShaderGlobals::blendState;
    // ...
    
    MyShaderGlobals::initialized = true;
}
```

### 4.2 Шаблон регистрации Custom Shader

```cpp
// В .cpp файле, ВНЕ функций:
namespace MyShaderGlobals
{
    wi::graphics::Shader pixelShader;
    wi::graphics::BlendState blendState;
    wi::graphics::RasterizerState rasterizerState;
    wi::graphics::DepthStencilState depthStencilState;
    wi::graphics::PipelineState pipelineState;
    bool initialized = false;
}

void InitializeMyCustomShader()
{
    if (MyShaderGlobals::initialized) return;

    // 1. Загрузка pixel shader
    bool loaded = wi::renderer::LoadShader(
        wi::graphics::ShaderStage::PS,
        MyShaderGlobals::pixelShader,
        "MyShader_PS.cso"
    );
    
    if (!loaded || !MyShaderGlobals::pixelShader.IsValid())
    {
        wi::backlog::post("Failed to load shader", wi::backlog::LogLevel::Error);
        return;
    }

    // 2. Настройка Blend State
    MyShaderGlobals::blendState = {};
    MyShaderGlobals::blendState.render_target[0].blend_enable = true;
    MyShaderGlobals::blendState.render_target[0].src_blend = wi::graphics::Blend::SRC_ALPHA;
    MyShaderGlobals::blendState.render_target[0].dest_blend = wi::graphics::Blend::INV_SRC_ALPHA;
    MyShaderGlobals::blendState.render_target[0].blend_op = wi::graphics::BlendOp::ADD;
    MyShaderGlobals::blendState.render_target[0].src_blend_alpha = wi::graphics::Blend::ONE;
    MyShaderGlobals::blendState.render_target[0].dest_blend_alpha = wi::graphics::Blend::ONE;
    MyShaderGlobals::blendState.render_target[0].blend_op_alpha = wi::graphics::BlendOp::ADD;
    MyShaderGlobals::blendState.render_target[0].render_target_write_mask = wi::graphics::ColorWrite::ENABLE_ALL;

    // 3. Настройка Rasterizer State
    MyShaderGlobals::rasterizerState = {};
    MyShaderGlobals::rasterizerState.fill_mode = wi::graphics::FillMode::SOLID;
    MyShaderGlobals::rasterizerState.cull_mode = wi::graphics::CullMode::NONE;  // Для billboard
    MyShaderGlobals::rasterizerState.depth_clip_enable = true;

    // 4. Настройка Depth Stencil State
    MyShaderGlobals::depthStencilState = {};
    MyShaderGlobals::depthStencilState.depth_enable = true;
    MyShaderGlobals::depthStencilState.depth_write_mask = wi::graphics::DepthWriteMask::ZERO;  // Только чтение
    MyShaderGlobals::depthStencilState.depth_func = wi::graphics::ComparisonFunc::GREATER;    // Reversed-Z!

    // 5. Создание PSO
    wi::graphics::PipelineStateDesc psoDesc = {};
    psoDesc.vs = wi::renderer::GetShader(wi::enums::VSTYPE_OBJECT_COMMON);  // Стандартный VS!
    psoDesc.ps = &MyShaderGlobals::pixelShader;
    psoDesc.bs = &MyShaderGlobals::blendState;
    psoDesc.rs = &MyShaderGlobals::rasterizerState;
    psoDesc.dss = &MyShaderGlobals::depthStencilState;
    psoDesc.il = nullptr;  // НЕ НУЖЕН для object shaders!
    psoDesc.pt = wi::graphics::PrimitiveTopology::TRIANGLELIST;

    wi::graphics::GraphicsDevice* device = wi::graphics::GetDevice();
    if (!device->CreatePipelineState(&psoDesc, &MyShaderGlobals::pipelineState))
    {
        wi::backlog::post("Failed to create PSO", wi::backlog::LogLevel::Error);
        return;
    }

    // 6. Регистрация Custom Shader
    wi::renderer::CustomShader customShader;
    customShader.name = "MyShader";
    customShader.filterMask = wi::enums::FILTER_TRANSPARENT;  // Для transparent pass
    customShader.pso[wi::enums::RENDERPASS_MAIN] = MyShaderGlobals::pipelineState;

    int shaderID = wi::renderer::RegisterCustomShader(customShader);
    
    if (shaderID >= 0)
    {
        MyShaderGlobals::initialized = true;
        wi::backlog::post("Custom shader registered with ID: " + std::to_string(shaderID));
    }
}

// Применение к материалу:
void ApplyShaderToMaterial(wi::scene::MaterialComponent* material)
{
    const auto& customShaders = wi::renderer::GetCustomShaders();
    for (size_t i = 0; i < customShaders.size(); ++i)
    {
        if (customShaders[i].name == "MyShader")
        {
            material->SetCustomShaderID(static_cast<int>(i));
            break;
        }
    }
}
```

---

## 5. Типичные ошибки и решения

### 5.1 Ошибка: "D3D12CreateVersionedRootSignatureDeserializer failed"

**Причина:** Попытка использовать собственную root signature или Input Layout с object shaders.

**Решение:** 
- Использовать `objectHF.hlsli` 
- НЕ задавать Input Layout (`psoDesc.il = nullptr`)
- Использовать стандартный VS движка

### 5.2 Ошибка: Конфликт двух main() функций

**Причина:** Определён `OBJECTSHADER_COMPILE_PS` + своя `main()`.

**Решение:** НЕ определять `OBJECTSHADER_COMPILE_PS` при написании своей `main()`.

### 5.3 Ошибка: Шейдер компилируется, но ничего не рисует

**Причины:**
1. Указатели в `PipelineStateDesc` на локальные переменные
2. Custom shader не зарегистрирован
3. Material не имеет CustomShaderID
4. Неправильный filterMask

**Решение:** Проверить все пункты из раздела 4.

### 5.4 Ошибка: Эффекты не видны (glow, halo исчезают)

**Причина:** Умножение на альфу текстуры в прозрачных областях.

```hlsl
// ❌ НЕПРАВИЛЬНО:
float3 finalColor = baseColor + glow + halo;
float finalAlpha = textureColor.a;  // = 0 за пределами текстуры!
return float4(finalColor * finalAlpha, finalAlpha);  // Всё умножается на 0!
```

```hlsl
// ✅ ПРАВИЛЬНО:
color.rgb += emissiveColor + glow + halo;  // Добавляем к цвету
color.a *= smoothstep(1.0, 0.7, distFromCenter);  // Модифицируем альфу отдельно
return saturateMediump(color);
```

---

## 6. Reference: Пример Hologram Shader (из движка)

```hlsl
#define OBJECTSHADER_LAYOUT_COMMON
#include "objectHF.hlsli"

[earlydepthstencil]
float4 main(PixelInput input) : SV_TARGET
{
    ShaderMaterial material = GetMaterial();
    float4 uvsets = input.GetUVSets();
    
    write_mipmap_feedback(push.materialIndex, ddx_coarse(uvsets), ddy_coarse(uvsets));
    
    float4 color;
    [branch]
    if (material.textures[BASECOLORMAP].IsValid() && (GetFrame().options & OPTION_BIT_DISABLE_ALBEDO_MAPS) == 0)
    {
        color = material.textures[BASECOLORMAP].Sample(sampler_objectshader, uvsets);
        color.rgb = max(color.r, max(color.g, color.b));  // Grayscale
    }
    else
    {
        color = 1;
    }
    color *= input.color;

    // Emissive
    float3 emissiveColor = material.GetEmissive();
    [branch]
    if (any(emissiveColor) && material.textures[EMISSIVEMAP].IsValid())
    {
        float4 emissiveMap = material.textures[EMISSIVEMAP].Sample(sampler_objectshader, uvsets);
        emissiveColor *= emissiveMap.rgb * emissiveMap.a;
    }
    color.rgb += emissiveColor;

    float time = GetFrame().time;
    float2 uv = input.pos.xy * GetCamera().internal_resolution_rcp;

    // Wave effect
    color.a *= sin(input.GetPos3D().y * 30 + time * 10) * 0.5 + 0.5;

    // Rim lighting
    color *= lerp(0.3, 6, pow(1 - saturate(dot(normalize(input.nor), normalize(input.GetViewVector()))), 2));

    // Base alpha
    color.a += 0.2;
    color.a = saturate(color.a);

    // Noise grain
    float noise = 1;
    noise *= texture_random64x64.SampleLevel(sampler_linear_wrap, uv * 4 + time, 0).r * 2 - 1;
    noise *= texture_random64x64.SampleLevel(sampler_linear_wrap, uv * float2(-2, 3) + time, 0).g * 2 - 1;
    noise = noise * 0.5 + 0.5;
    color *= noise;

    return saturateMediump(color);
}
```

---

## 7. Чек-лист перед созданием кастомного шейдера

- [ ] Pixel Shader использует `#define OBJECTSHADER_LAYOUT_COMMON` (или другой layout)
- [ ] Pixel Shader **НЕ** использует `#define OBJECTSHADER_COMPILE_PS`
- [ ] Pixel Shader включает `#include "objectHF.hlsli"`
- [ ] Pixel Shader имеет `[earlydepthstencil]` перед `main()`
- [ ] Pixel Shader вызывает `write_mipmap_feedback()` в начале
- [ ] C++ код хранит Shader, BlendState, RasterizerState, DepthStencilState в глобальной области
- [ ] `PipelineStateDesc.il = nullptr` (Input Layout не нужен)
- [ ] `PipelineStateDesc.vs` использует стандартный VS: `wi::renderer::GetShader(wi::enums::VSTYPE_OBJECT_COMMON)`
- [ ] Custom Shader зарегистрирован через `wi::renderer::RegisterCustomShader()`
- [ ] Material имеет установленный `SetCustomShaderID()`
- [ ] Для transparent объектов: `filterMask = wi::enums::FILTER_TRANSPARENT`
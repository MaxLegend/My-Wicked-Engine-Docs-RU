
## Анализ паттернов Compute Shaders в Wicked Engine

### 1. **Базовая структура файла**

```hlsl
// 1. Включения и определения
#include "globals.hlsli"                    // ОБЯЗАТЕЛЬНО - базовые функции и структуры
#include "ShaderInterop_Postprocess.h"       // Опционально - для постобработки
#include "другие_include.hlsli"              // Специфичные хелперы

// 2. Push Constants (параметры с CPU)
PUSHCONSTANT(имя_структуры, ТипСтруктуры);

// 3. Входные ресурсы (SRV - Shader Resource View)
Texture2D<float4> input_texture : register(t0);
StructuredBuffer<DataType> input_buffer : register(t1);
Texture3D<float4> volume_texture : register(t2);

// 4. Выходные ресурсы (UAV - Unordered Access View)
RWTexture2D<float4> output : register(u0);
RWTexture2D<float2> output_secondary : register(u1);
RWStructuredBuffer<uint4> output_buffer : register(u2);

// 5. Вспомогательные функции
void HelperFunction(...) { ... }

// 6. Точка входа
[numthreads(X, Y, Z)]
void main(uint3 DTid : SV_DispatchThreadID, ...)
{
    // Логика
}
```

---

### 2. **Типичные размеры групп потоков**

| Тип задачи | numthreads | Комментарий |
|------------|------------|-------------|
| **2D изображения** | `[numthreads(8, 8, 1)]` | `POSTPROCESS_BLOCKSIZE = 8` |
| **1D буферы/вершины** | `[numthreads(64, 1, 1)]` | Для линейных данных |
| **Visibility тайлы** | `[numthreads(8, 8, 1)]` | `VISIBILITY_BLOCKSIZE = 8` |
| **3D текстуры** | `[numthreads(8, 8, 8)]` | Для объёмных данных |

---

### 3. **Семантики входных параметров main()**

```hlsl
[numthreads(8, 8, 1)]
void main(
    uint3 DTid : SV_DispatchThreadID,     // Глобальный ID потока (самый частый)
    uint3 GTid : SV_GroupThreadID,         // ID потока внутри группы
    uint3 Gid : SV_GroupID,                // ID группы
    uint groupIndex : SV_GroupIndex        // Линейный индекс внутри группы (0..63)
)
```

**Использование:**
- `DTid` — для адресации пикселей/вершин напрямую
- `groupIndex` — для доступа к shared memory или Wave операций
- `Gid` — для тайловой обработки

---

### 4. **Паттерн: Обработка 2D изображения (постобработка)**

```hlsl
#include "globals.hlsli"
#include "ShaderInterop_Postprocess.h"

PUSHCONSTANT(postprocess, PostProcess);

Texture2D<float4> input : register(t0);
RWTexture2D<float4> output : register(u0);

[numthreads(POSTPROCESS_BLOCKSIZE, POSTPROCESS_BLOCKSIZE, 1)]
void main(uint3 DTid : SV_DispatchThreadID)
{
    // 1. Вычисляем UV координаты
    float2 uv = (DTid.xy + 0.5f) * postprocess.resolution_rcp;
    
    // 2. Читаем входные данные
    float4 color = input.SampleLevel(sampler_linear_clamp, uv, 0);
    
    // 3. Обрабатываем
    color.rgb = ProcessColor(color.rgb);
    
    // 4. Записываем результат
    output[DTid.xy] = color;
}
```

---

### 5. **Паттерн: Обработка буферов/вершин (skinning)**

```hlsl
#include "globals.hlsli"

PUSHCONSTANT(push, SkinningPushConstants);

[numthreads(64, 1, 1)]
void main(uint3 DTid : SV_DispatchThreadID)
{
    const uint vertexID = DTid.x;
    
    // 1. Ранний выход для избыточных потоков
    [branch]
    if (vertexID >= push.vertexCount)
        return;
    
    // 2. Чтение данных через bindless
    float4 pos = 0;
    [branch]
    if (push.vb_pos >= 0)
    {
        pos = bindless_buffers_float4[descriptor_index(push.vb_pos)][vertexID];
    }
    
    // 3. Обработка данных
    // ...
    
    // 4. Запись результата через bindless
    [branch]
    if (push.so_pos >= 0)
    {
        bindless_rwbuffers_float4[descriptor_index(push.so_pos)][vertexID] = pos;
    }
}
```

---

### 6. **Паттерн: Тайловая обработка (visibility)**

```hlsl
struct VisibilityPushConstants
{
    uint global_tile_offset;
};
PUSHCONSTANT(push, VisibilityPushConstants);

StructuredBuffer<VisibilityTile> binned_tiles : register(t0);
RWTexture2D<float4> output : register(u0);

[numthreads(VISIBILITY_BLOCKSIZE, VISIBILITY_BLOCKSIZE, 1)]
void main(uint Gid : SV_GroupID, uint groupIndex : SV_GroupIndex)
{
    // 1. Загрузка данных тайла
    const uint tile_offset = push.global_tile_offset + Gid.x;
    VisibilityTile tile = binned_tiles[tile_offset];
    
    // 2. Проверка валидности потока
    [branch]
    if (!tile.check_thread_valid(groupIndex))
        return;
    
    // 3. Вычисление координат пикселя
    const uint2 GTid = remap_lane_8x8(groupIndex);
    const uint2 pixel = unpack_pixel(tile.visibility_tile_id) * VISIBILITY_BLOCKSIZE + GTid;
    const float2 uv = ((float2)pixel + 0.5) * GetCamera().internal_resolution_rcp;
    
    // 4. Обработка
    // ...
    
    // 5. Запись
    output[pixel] = result;
}
```

---

### 7. **Ключевые функции доступа к данным**

```hlsl
// Получить данные кадра (время, опции и т.д.)
GetFrame().time
GetFrame().options & OPTION_BIT_XXX

// Получить данные камеры
GetCamera().position
GetCamera().internal_resolution_rcp
GetCamera().inverse_view_projection

// Получить данные погоды
GetWeather().volumetric_clouds.xxx
GetWeather().ocean.xxx
GetWeather().atmosphere.xxx

// Bindless доступ к ресурсам
bindless_textures[descriptor_index(index)]
bindless_buffers[descriptor_index(index)]
bindless_buffers_float4[descriptor_index(index)]
bindless_rwbuffers_float4[descriptor_index(index)]

// Реконструкция мировой позиции из depth
float3 P = reconstruct_position(uv, depth);
float3 P = reconstruct_position(uv, depth, inverse_matrix);
```

---

### 8. **Push Constants - типичные структуры**

```hlsl
// Для постобработки (ShaderInterop_Postprocess.h)
struct PostProcess
{
    float2 resolution;
    float2 resolution_rcp;    // 1.0 / resolution
    float4 params0;           // Пользовательские параметры
    float4 params1;
    // ...
};

// Для кастомных эффектов - своя структура
struct MyEffectPushConstants
{
    uint param1;
    float param2;
    // ... (должна быть в ShaderInterop файле!)
};
```

---

### 9. **Типичные проверки и ранний выход**

```hlsl
// Проверка границ буфера
[branch]
if (vertexID >= push.vertexCount)
    return;

// Проверка глубины (пропуск неба)
const float depth = texture_depth[DTid.xy];
if (depth == 0)
    return;

// Проверка UV в пределах scissor
[branch]
if (!GetCamera().is_uv_inside_scissor(uv))
{
    output[pixel] = 0;
    return;
}

// Проверка валидности дескриптора
[branch]
if (push.texture_index >= 0)
{
    // Безопасный доступ
}
```

---

### 10. **Выводы для создания своего Compute Shader**

1. **Всегда включай `globals.hlsli`** — это база всего
2. **Используй `PUSHCONSTANT`** для передачи параметров с CPU
3. **Стандартный размер группы — 8x8x1** для 2D, 64x1x1 для 1D
4. **Проверяй границы** перед записью в UAV
5. **Используй `[branch]`** для условных проверок (оптимизация)
6. **Bindless доступ** через `descriptor_index(push.index)`
7. **UV вычисляется как** `(DTid.xy + 0.5) * resolution_rcp`

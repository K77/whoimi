

# Vulkan Shader编译与加载过程

##  间接

参考了[Vulkan教程](https://vulkan-tutorial.com/Drawing_a_triangle/Graphics_pipeline_basics/Shader_modules)

​	和一般的图形API不同（GL\DX），Vulkan 的shader代码是通过字节码的格式保存的，而非具备可读性的HLSL和GLSL。字节码的格式交过SPIR-V，它设计出来和Vulkan和OpenCL一同使用。可以用来编写图形Shader和计算Shader，这部分专注于图形Shader。

使用字节码的优点是由GPU供应商提供的把Shader代码转换成本地代码的编译器能复杂度更低。

过去使用可读代码的风险，编译器实现灵活，尝试特性实现不同：The past has shown that with human-readable syntax like GLSL, some GPU vendors were rather flexible with their interpretation of the standard. If you happen to write non-trivial shaders with a GPU from one of these vendors, then you'd risk other vendor's drivers rejecting your code due to syntax errors, or worse, your shader running differently because of compiler bugs. With a straightforward bytecode format like SPIR-V that will hopefully be avoided.

我们不需要手写字节码，而是需要用与供应商独立的编译器把GLSL转化成SPIR-V.编译器用来验证Shader代码是完全标准的，并且生成 SPIR-V 代码集成在程序中。编译器提供了library用来集成在程序中，也可以用现有编译好的程序工具。

我们可以直接使用编译器glslangValidator.exe，不过这里使用glslc工具，因为这个工具提供了includes功能，并且它的参数和gcc和Clang很相似。这些已经集成在了VulkanSDK当中。

后面先展示GLSL代码，然后转换成SPIR-V，并在运行时加载。

GLSL语法：[语法简介](http://nehe.gamedev.net/article/glsl_an_introduction/25007/)

## Shader基本使用

### 编写GLSL代码

首先先编写GLSL代码文件shader.vert：

```c
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) out vec3 fragColor;

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

### 编译成字节码

代码写完后使用编译器的工具编译成字节码：

```c
C:/VulkanSDK/x.x.x.x/Bin32/glslc.exe shader.vert -o vert.spv
```

之后我们就得到了字节码文件vert.spv。

### 程序运行时加载字节码文件

需要用于渲染是加载字节码文件：

```c
static std::vector<char> readFile(const std::string& filename) {
    std::ifstream file(filename, std::ios::ate | std::ios::binary);

    if (!file.is_open()) {
        throw std::runtime_error("failed to open file!");
    }
    
    size_t fileSize = (size_t) file.tellg();
std::vector<char> buffer(fileSize);
    file.seekg(0);
file.read(buffer.data(), fileSize);
    file.close();

	return buffer;
}
```

### 根据字节码数据创建 Shader Modules

这部分内容就进入了图形API的部分。根据ShaderModule就可以吧这个Shader代码设置成渲染状态的一部分。

## 进一步分析

SPIR-V 是二进制的中间码，优点是不需要考虑供应商特性。缺点是需要考虑如何将shader语言编译成SPIR-V.

最好的方案是：[Khronos' glslang library](https://github.com/KhronosGroup/glslang) 和 [Google's shaderc](https://github.com/google/shaderc).

都是开源的Shader编译库。

使用GLSL到SPIR-V的优点是不需要在程序中发布Shader代码

### 一种GLSL不同的版本

GLSL有很多不同的版本：

-   Modern mobile (GLES3+)
-   Legacy mobile (GLES2)
-   Modern desktop (GL3+)
-   Legacy desktop (GL2)
-   Vulkan GLSL

Vulkan当中有很多和其他变体不兼容的地方

-   Descriptor sets, no such concept in OpenGL
-   Push constants, no such concept in OpenGL, but very similar to "uniform vec4 UsuallyRegisterMappedUniform;"
-   Subpass input attachments, maps well to [Pixel Local Storage on ARM Mali GPUs](https://community.arm.com/graphics/b/blog/posts/pixel-local-storage-on-arm-mali-gpus)
-   gl_InstanceIndex vs gl_InstanceID. Same, but Vulkan GLSL's version InstanceIndex (finally!) adds base instance offset for you



## Shader 变种与变体策略

在 Vulkan 中，同一份 SPIR-V 可以通过不同方式产生多种“变体”行为，常见手段有：**Specialization Constants（特化常量）**、**UBO/Push Constants** 以及**多份 Shader 源码/多 Pipeline**。下面主要说明特化常量与变体策略。

### Specialization Constants（特化常量）

与 OpenGL/GL 中的 uniform 不同，特化常量在**创建 Pipeline 之前**就确定数值，由驱动在编译/链接阶段带入，因此编译器可以做**常量折叠、死代码消除、循环展开**等优化，得到接近“手写多份 Shader”的性能，同时只维护一份 SPIR-V。

**GLSL 中声明：**

```glsl
layout(constant_id = 0) const int NUM_LIGHTS = 4;
layout(constant_id = 1) const bool USE_SHADOW = true;
```

- `constant_id` 对应后面在 API 里填写的映射 ID。
- 若不提供特化数据，则使用 GLSL 中的默认值。

**运行时填入：C++ 侧**

1. 准备一块数据（按 offset 排好各常量的值）：
   ```cpp
   struct SpecData {
       int numLights;
       uint32_t useShadow;  // bool 在 SPIR-V 中常为 32bit
   } specData = { 4, 1 };
   ```

2. 为每个常量填写 `VkSpecializationMapEntry`（constantID、offset、size）：
   ```cpp
   VkSpecializationMapEntry entries[] = {
       { 0, offsetof(SpecData, numLights), sizeof(specData.numLights) },
       { 1, offsetof(SpecData, useShadow), sizeof(specData.useShadow) },
   };
   ```

3. 用 `VkSpecializationInfo` 把 data、size 和 entries 绑在一起，再在创建 `VkPipeline` 时把该 `VkSpecializationInfo` 填到对应 stage 的 `pSpecializationInfo` 中（每个 stage 可各有一份）。

同一份 SPIR-V + 不同的 `VkSpecializationInfo` = 不同的变体；通常对应不同的 `VkPipeline`（或通过 pipeline cache 复用编译结果）。

**限制与注意：**

- 特化常量一般为**标量**（int、float、bool 等）；数组不能直接作为带 `constant_id` 的常量，若需要可对每个元素单独给一个 constant_id，或通过其他方式传入。
- 修改特化常量值即得到新变体，需要**新 Pipeline 或走 pipeline 创建流程**，因此变体数量不宜爆炸（例如按“灯光数、是否阴影、材质类型”等离散维度组合成有限个 Pipeline）。

### 何时用特化常量 vs UBO / Push Constants

| 方式 | 何时使用 |
|------|----------|
| **Specialization Constants** | 在管线创建前就确定的离散选项：灯光数量、是否用阴影、分支类型等；需要编译器做激进优化、减少运行时分支。 |
| **Push Constants** | 每帧或每 draw 变化的小数据（矩阵、少量参数），低延迟、无 descriptor 绑定。 |
| **UBO / SSBO** | 大量数据、或需要跨多个 draw 共享、或运行时才确定的配置。 |

实践中可以：**特化常量**定“变体维度”（几盏灯、有没有阴影），**Push Constants** 传矩阵和当前帧参数，**UBO** 传大块材质/场景数据。

### 多 Pipeline 与变体组合

若变体维度较多（例如 灯光数 × 是否阴影 × 若干材质类型），可以：

1. 为每种**常用组合**预创建一条 Pipeline，创建时对 vertex/fragment 的 `VkSpecializationInfo` 传入对应常量；
2. 运行时根据当前“灯光数、是否阴影、材质”等选择对应 Pipeline 绑定并绘制；
3. 使用 **Pipeline Cache**（`VkPipelineCache`）缓存已编译结果，加速同设备上的重复创建。

这样既保留“Shader 变种”的优化空间，又避免运行时动态分支带来的性能损失。




















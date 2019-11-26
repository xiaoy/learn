# Graphic pipeline basics
## 简介
图形渲染管线是将网格(mesh)上的顶点和贴图进行一些列的操作后最终表现到渲染目标的像素上。渲染流程图如下：

![graphic pipeline](../res/vulkan_simplified_pipeline.svg)

*input assembler* 从指定的缓存里收集顶点数据，有些为了节省空间，使用额外的顶点索引缓存来指定顶点数据。

*vertex shader* 每个顶点都要执行的程序，通常是将顶点从模型空间转换到屏幕空间。并且将转换后的顶点数据传到管线下一流程。

*tessellation shader* 用来通过一定的规则来细分几何体，从而来增加网格(mesh)质量。通常用在砖块，楼梯这些有突出结构的物件上，当摄像机靠近时不会显得太平坦。

*geometry shader* 运行在每个基础几何体上(三角形，线段，点)，此shader可以让输入的几何体降级为更简单的几何体，或升级为更复杂的几何体。这和*tessellation shader*有些相似，但更加灵活。但当前大多数显卡运行此shader效率不高，除了intel的集成显卡。

*rasterization* 阶段将几何体离散为片元(fragments)。这就是充满framebuffer的像素。所有在framebuffer外的像素被丢弃，*vertex shader*输出的顶点属性变量插值到每个片元上。因为深度检测，在片元后方的其它片元被丢弃。

*fragment shader* 每个存活的片元都要经过此shader处理并且需要判断使用对应的颜色和深度值写入相应的framebuffer。在这个阶段可以使用*vertex shader*传递来的贴图坐标(uv)和法线变量，以及其它变量。

*color blending*阶段将映射到同一像素的不同片元混合到一起，片元之间可以简单的互相覆盖，或者叠加，或者根据透明度来混合。

颜色为绿色阶段(stages)称作*固定管线阶段(fixed-funciton stage)*。本阶段只能设置参数，运行流程是预定义的，不可改变。

颜色为橙色阶段(stages)称作*可编程阶段*，这些阶段可以上传自己的shader代码，来实现自己想完成的操作。这些程序并行运行在GPU的多核上。

像OpenGL，Directx3D旧的APIs，控制管线只能通过类似于`glBlendFunc`和`OMSetBlendState`这样的接口设置参数来实现。在Vulkan里的图像管线几乎全部都是可修改的，因此当你想修改shader，绑定不同的framebuffers或修改融合函数时，都需要重新创建管线。缺点是你需要根据不同的组合创建不同的管线来实现自定义的渲染操作。但是这种高度透明的自定义管线可以让驱动更好的优化。

一些可编程的阶段可以根据需要是否开启。比如如果只是渲染简单的几何体，可以选择关闭*tessellation shader*和*geometry shader*。再者比如只是想渲染*shadow map*，可以关闭*fragment shader*。

## Shader 模块
相比早期的APIs使用`GLSL`和`HLSL`语法来编写可读性很高的shader代码，Vulkan的shader代码转换为字节码格式。这种字节码的格式称作`SPIR-V`被设计用在Vulkan和OpenCL。这种格式用在图形和计算shader上。

优点是GPU厂商的编译器将字节码格式的shader代码转换为运行时需要的指令更加简单。过去使用类似`GLSL`写的shader代码，一些GPU厂商对标准的解释过于灵活，很有可能在不同GPU上编译结果不一致，使用字节码可以有效的避免类似问题。

但这并不是说要手写字节码，Khronos发布了和厂商无关的编译器，将`GLSL`编译为`SPIR-V`。编译器被设计为完全符合标准的输出字节码。也可将编译器嵌入到程序里，在运行时将`GLSL`编译为`SPIR-V`供程序使用。我们可以直接使用Khronos提供的`glslangValidator.exe`，但我们选择使用google提供的`glslc.exe`。优点是`glslc`使用的参数格式和著名的编译器GCC，CLang一致，并且还提供了类似于`include`的功能。

GLSL是一种C语法风格的shading语言。程序里包含`main`函数被每个对象调用。除了使用函数参数当做输入，返回值当做输出，GLSL使用许多全局变量来处理输入输出。本语言包括很多特性来辅助图形编程，比如内置的基础`vector`和`matrix`，还包括函数操作，比如点积(cross-product)，矩阵和向量乘法(matrix-vector product)，向量反射(reflections around a vector)。向量类型使用`vec`以及后缀数字来标识向量包含的元素数量。

### Vetex shader
顶点shader处理每个输入的顶点。输入顶点携带多个属性，比如世界坐标，颜色，法线，以及贴图坐标。输出的是最终在裁剪空间的位置以及片元shader阶段需要使用的属性，比如颜色和贴图坐标(uv)。最终通过光栅化平滑将属性插值到每个片元的输入参数里。

裁剪空间(clip coordinate)是来自顶点shader的有四个元素向量，此向量通过除以最后一个变量转换到归一化设备空间(normalized device coordinate)。归一化设备空间是齐次坐标(homogeneous coordinate)将framebuffer的坐标映射到[-1,1]和[-1, 1]坐标系统，如下所示：

![normalized_device_coordinate](../res/normalized_device_coordinates.svg)

相比OpenGL Y坐标进行了翻转。Z坐标和Directx3D一样区间变为[0,1]。

当前先将坐标添加到顶点shader里，以下为三角形三个顶点代码如下：

```GLSL
#version 450

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

`main`函数被每个顶点调用。内置`gl_VertextIndex`变量为当前顶点的索引。内置`gl_Position`变量为输出位置。

### Fragment Shader
通过三个顶点，光栅化将三角形覆盖的面积通过fragment覆盖。每个fragment通过相应的shader执行来向fragmentbuffer写入颜色和深度值。以下为红色三角形fragment shader代码：

```GLSL
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```
`layout(location = 0)`指定索引为0的framebuffer，指定输出颜色为`out vec4 outColor`。

### 单个顶点颜色
通过指定单个顶点的颜色，然后利用光栅化将顶点属性值光滑插值后可以输出一个渐变颜色三角形，要做的就是从顶点输出颜色，然后在fragment shader 接收输入颜色。

```GLSL
vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

然后fragment shader 接收渐变值，输出如下：

```GLSL
layout(location = 0) in vec3 fragColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

### 编译shader
通过`glslc.exe`编译输出字节码。

### 加载shader
读取对应位置的编译后的文件：

```C++
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
### 创建shader模块
挂载到渲染管线，需要将shader使用`VkShaderModule`来封装，代码如下：

```C++
VkShaderModule createShaderModule(const std::vector<char>& code) {
    VkShaderModuleCreateInfo createInfo = {};
    createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
    createInfo.codeSize = code.size();
    createInfo.pCode = reinterpret_cast<const uint32_t*>(code.data());

    VkShaderModule shaderModule;
    if (vkCreateShaderModule(device, &createInfo, nullptr, &shaderModule) != VK_SUCCESS) {
        throw std::runtime_error("failed to create shader module!");
    }

    return shaderModule;
}
```

在`VkShaderModule`创建信息传递变量为`pCode`，但结构体变量类型为`const uint32_t*`指针，所以需要`reinterpret_cast`转换。

### Shader 阶段创建
Shader使用必须指定具体渲染管线阶段，所以要使用`VkPipelineShaderStageCreateInfo`来指定Shader使用的阶段。以下为顶点shader创建信息：

```C++
VkPipelineShaderStageCreateInfo vertShaderStageInfo = {};
vertShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
vertShaderStageInfo.stage = VK_SHADER_STAGE_VERTEX_BIT;
vertShaderStageInfo.module = vertShaderModule;
vertShaderStageInfo.pName = "main";
```

变量`pName`为shader主函数入口。

创建信息成员还有`pSpecializationInfo`可以指定Shader里使用常量。使用这些常量，编译器可以优化相对应的`if`条件语句。

创建fragment 阶段信息如下：

```C++
VkPipelineShaderStageCreateInfo fragShaderStageInfo = {};
fragShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
fragShaderStageInfo.stage = VK_SHADER_STAGE_FRAGMENT_BIT;
fragShaderStageInfo.module = fragShaderModule;
fragShaderStageInfo.pName = "main";
```

## 固定管线(fixed function)
### 顶点输入
`VkPipelineVertexInputStateCreateInfo`结构体描述了要传递给顶点shader的数据格式。粗略描述有两点：

* 绑定(bindings)：数据间距以及数据是否是单顶点或单实例(per-instance)
* 属性描述(Attribute descriptions)：传递到顶点shader的属性类型，通过多大间距用在那个绑定点

因为当前是将顶点数据写在shader里，所以结构体初始化如下：

```C++
VkPipelineVertexInputStateCreateInfo vertexInputInfo = {};
vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
vertexInputInfo.vertexBindingDescriptionCount = 0;
vertexInputInfo.pVertexBindingDescriptions = nullptr; // Optional
vertexInputInfo.vertexAttributeDescriptionCount = 0;
vertexInputInfo.pVertexAttributeDescriptions = nullptr; // Optional
```
`pVertexBindingDescriptions` 和`pVertexAttributeDescriptions` 指向描述顶点数据的结构体数组。

### 输入汇编(input assembly)
`VkPipelineInputAssemblyStateCreateInfo`描述两件事：

* 通过顶点绘制什么形状的几何体，`topology`成员来指定：
    * `VK_PRIMITIVE_TOPOLOGY_POINT_LIST`：绘制点
    * `VK_PRIMITIVE_TOPOLOGY_LINE_LIST`：绘制线段
    * `VK_PRIMITIVE_TOPOLOGY_LINE_STRIP`：使用前一条线段的终点当下一线段的起点，表现为连续线段
    * `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST`：绘制三角形
    * `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP`：使用前一三角形的后两个点到新的三角形前两个点
* 基础图形是否开启重启，正常按顺序使用顶点，但也可以通过索引去使用顶点，开启重启选项后，可以任意获取顶点

```C++
VkPipelineInputAssemblyStateCreateInfo inputAssembly = {};
inputAssembly.sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO;
inputAssembly.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
inputAssembly.primitiveRestartEnable = VK_FALSE;
```

### 视窗和裁剪(viewports and scissors)
视窗信息描述了在framebuffer上绘制的区域，矩形通过位置`(x,y)`和宽高`(width, height)`来定义。深度使用默认区间`[0,1)`即可。

裁剪信息描述了在其定义的区域像素保留，裁剪区域以外的像素丢弃。

需要注意的是窗口尺寸(宽度和高度)不一定和framebuffer尺寸相同，因为要使用swapchain 图像尺寸当做framebuffer尺寸。

![viewports_and_scissors](../res/viewports_scissors.png)

使用`VkPipelineViewportStateCreateInfo`来定义视窗和裁剪：

```C++
VkViewport viewport = {};
viewport.x = 0.0f;
viewport.y = 0.0f;
viewport.width = (float) swapChainExtent.width;
viewport.height = (float) swapChainExtent.height;
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;

VkRect2D scissor = {};
scissor.offset = {0, 0};
scissor.extent = swapChainExtent;

VkPipelineViewportStateCreateInfo viewportState = {};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.pViewports = &viewport;
viewportState.scissorCount = 1;
viewportState.pScissors = &scissor;
```

### 光栅化(Rasterizer)
光栅化器使用从顶点shader输出顶点组成的几何图形将其转化成片元集合，然后每个片元执行片元shader来着色。同时也执行深度检测(depth test)，面剔除(face culling)，裁剪检测(scissor test)，并且还可以配置输出的片元是充满整个几何图形还是使用线段绘制(wireframe rendering)。所有这些通过结构体`VkPipelineRasterizationStateCreateInfo`来配置。

```C++
VkPipelineRasterizationStateCreateInfo rasterizer = {};
rasterizer.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
rasterizer.depthClampEnable = VK_FALSE;
```

`depthClampEnable`设置为`VK_TRUE`时，近平面到第一个对象时的深度值保留，其它深度值舍弃。这通常用在阴影贴图(shadow map)上，使用这个功能需要开启GPU特性。

```C++
rasterizer.rasterizerDiscardEnable = VK_FALSE;
```
当`rasterizerDiscardEnable`设置为`VK_TRUE`时，所有的几何体不能通过此阶段，相当于关闭光栅化。

```C++
rasterizer.polygonMode = VK_POLYGON_MODE_FILL;
```

`polygonMode`定义了片元如何生成几何体，有如下几种模式：

* `VK_POLYGON_MODE_FILL`：使用片元充满多边形区域
* `VK_POLYGON_MODE_LINE`：多边形的边使用线来绘制
* `VK_POLYGON_MODE_POINT`：多边形的顶点使用点来绘制

使用其他模式，而不是填充渲染，需要开启GPU特性。

```C++
rasterizer.lineWidth = 1.0f;
```

`lineWidth`通过片元的数量来描述线的粗细，最大线宽依据硬件支持，线的厚度低于`1.0`需要开启GPU`wideLines`特性。

```C++
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_CLOCKWISE;
```

`cullMode`指定要剔除的面类型。可以选择关闭面剔除，或者剔除正面，或剔除反面或都剔除。`frontFace`定义正面顶点顺序，可以是顺时针，也可以是逆时针。

### 多重采样(Multisampling)
`VkPipelineMultisampleStateCreateInfo`结构体描述多重采样，这是一种解决抗锯齿(anti-aliasing)的方式。它通过将多个多边形的渲染片元光栅化到同一个像素。开启多重采样需要打开GPU特性。当前先用默认值。

```C++
VkPipelineMultisampleStateCreateInfo multisampling = {};
multisampling.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
multisampling.sampleShadingEnable = VK_FALSE;
multisampling.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
multisampling.minSampleShading = 1.0f; // Optional
multisampling.pSampleMask = nullptr; // Optional
multisampling.alphaToCoverageEnable = VK_FALSE; // Optional
multisampling.alphaToOneEnable = VK_FALSE; // Optional
```

### 深度和裁剪测试
如果使用了深度/裁剪缓存，那需要配置深度和裁剪测试使用`VkPipelineDepthStencilStateCreateInfo`。当前不使用，暂时传递`nullptr`。

### 颜色混合
当片元shader执行后返回一个颜色值，这个颜色值需要和已经在framebuffer里的颜色值结合。这种变换称作颜色混合，两种类型处理方式：

* 每个framebuffer混合旧和新的颜色值产生最终结果
* 全局framebuffer使用位操作来结合旧和新的颜色值

有两个结构体来描述来配置颜色混合。第一个结构体`VkPipelineColorBlendAttachmentState`包含单个绑定的framebuffer配置第二个结构体`VkPipelineColorBlendStateCreateInfo`包含全局颜色混合设置。单个framebuffer颜色融合配置如下：

```C++
VkPipelineColorBlendAttachmentState colorBlendAttachment = {};
colorBlendAttachment.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
colorBlendAttachment.blendEnable = VK_FALSE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD; // Optional
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD; // Optional
```

上述代码实现的效果和如下的伪码实现的结果一致：

```C++
if (blendEnable) {
    finalColor.rgb = (srcColorBlendFactor * newColor.rgb) <colorBlendOp> (dstColorBlendFactor * oldColor.rgb);
    finalColor.a = (srcAlphaBlendFactor * newColor.a) <alphaBlendOp> (dstAlphaBlendFactor * oldColor.a);
} else {
    finalColor = newColor;
}

finalColor = finalColor & colorWriteMask;
```

如果`blendEnable`设置为`VK_FALSE`，则通过当前片元shader的颜色为framebuffer的颜色。反之需要通过两个颜色以及设置的参数进行混合为最终结果。最终结果需要和`colorWriteMask`进行与的操作来筛选对应的颜色通道(选择对应的颜色通道来做遮罩)。

最常用的颜色混合方式是使用透明值来进行颜色混合，通过当前透明值结合之前的颜色和当前颜色计算最终颜色。`finalColor`计算结果伪代码如下:

```C++
finalColor.rgb = newAlpha * newColor + (1 - newAlpha) * oldColor;
finalColor.a = newAlpha.a;
```

可以通过如下结构体进行配置：

```C++
colorBlendAttachment.blendEnable = VK_TRUE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_SRC_ALPHA;
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD;
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD;
```

第二个结构体引用所有framebuffer用的颜色混合结构体数组，并且可以设置常量来让所对应的的framebuffer来计算相应的颜色混合。

```C++
VkPipelineColorBlendStateCreateInfo colorBlending = {};
colorBlending.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
colorBlending.logicOpEnable = VK_FALSE;
colorBlending.logicOp = VK_LOGIC_OP_COPY; // Optional
colorBlending.attachmentCount = 1;
colorBlending.pAttachments = &colorBlendAttachment;
colorBlending.blendConstants[0] = 0.0f; // Optional
colorBlending.blendConstants[1] = 0.0f; // Optional
colorBlending.blendConstants[2] = 0.0f; // Optional
colorBlending.blendConstants[3] = 0.0f; // Optional
```

如果想使用第二种方式的混合(位结合)，则需要开启`logicOpEnable`为`VK_TRUE`。具体的位操作通过`logicOp`来具体指定，但是开启了这种方式的颜色混合，第一种对应每个framebuffer配置颜色混合的方式将关闭。`colorWriteMask`依然会生效，来筛选每个framebuffer要使用的通道。当然两种方式都可以关闭，则每个片元shader输出的颜色直接覆盖framebuffer对应的像素变量。

### 动态状态
当管线创建后有几个状态可以动态改变，比如视图(viewport)的尺寸，线宽(line width)，融合常量(blend constants)。这些通过结构体`VkPipelineDynamicStateCreateInfo`来指定。

```C++
VkDynamicState dynamicStates[] = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_LINE_WIDTH
};

VkPipelineDynamicStateCreateInfo dynamicState = {};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = 2;
dynamicState.pDynamicStates = dynamicStates;
```

这将导致上面配置的参数失效，需要我们在运行时指定新的参数。后续章节再讨论。

### 管线布局(pipeline layout)
在shader里可以使用`uniform`变量，它们是shader里用到的全局变量类似于可以修改的动态状态，而不需要重建shader。它们通常用在传递顶点shader使用的转换矩阵(transformation matrix)，或者片元shader里用到贴图采样(texture samples)。

当管线创建时，通过`VkPipelineLayout`来指定对应的`uniform`变量。现在还用不到，所以先创建空白管线布局。

```C++
VkPipelineLayoutCreateInfo pipelineLayoutInfo = {};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 0; // Optional
pipelineLayoutInfo.pSetLayouts = nullptr; // Optional
pipelineLayoutInfo.pushConstantRangeCount = 0; // Optional
pipelineLayoutInfo.pPushConstantRanges = nullptr; // Optional

if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create pipeline layout!");
}
```

此结构体还指定常量`push constants`，这是另一种向shader传递变量的方式。因为管线布局伴随整个程序周期，所以在最后销毁。

```C++
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

### 总结
这就是设置所有固定管线状态的方式，虽然有很多东西，但当程序运行和预期不一致时，相对容易的查出问题所在。

## 渲染批次
### 设置
在完成创建管线之前，需要设置Vulkan渲染使用的framebuffer附件(attachments)。需要指定使用到的颜色和深度缓存，采样数以及在整个渲染流程如何处理它们。所有这些信息封装在*渲染通道对象(render pass object)*上。所以需要在创建渲染管线之前创建渲染批次对象。

### 附件说明(attachment description)
当前情况是仅仅使用颜色缓存附件，这里所说的颜色缓存既是swapchain里的图像。

```C++
void createRenderPass() {
    VkAttachmentDescription colorAttachment = {};
    colorAttachment.format = swapChainImageFormat;
    colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
}
```

`format`使用和swapchain 图像格式一致即可，不使用多重采样，因此使用`VK_SAMPLE_COUNT_1_BIT`即可。

```C++
colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
```

`loadOp`和`storeOp`表示在渲染前和渲染后如何处理颜色附件(color attachment)里的数据。`loadOp`有以下处理方式：

* `VK_ATTACHMENT_LOAD_OP_LOAD`：保留附件里存在的数据
* `VK_ATTACHMENT_LOAD_OP_CLEAR`：使用常量清除附件里的数据
* `VK_ATTACHMENT_LOAD_OP_DONT_CARE`：存在附件里的常量未定义，不需要关心

对于`storeOp`有两种可能：

* `VK_ATTACHMENT_STORE_OP_STORE`：渲染结果将存储在内存里后续可以读取
* `VK_ATTACHMENT_STORE_OP_DONT_CARE`：渲染结束后framebuffer里的数据处于未定义状态

`storeOp`和`loadOp`作用在颜色和深度framebuffer上，`stencilLoadOp`和`stencilStoreOp`作用在裁剪buffer上，因此这里忽略即可。

```C++
colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
```

贴图和framebuffer在Vulkan里通过`VKImage`对象使用特定的像素格式构成，但是像素在内存里布局(layout)根据使用方式不同而改变。以下为最常用的布局方式：

* `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`：图像用作颜色附件
* `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`：图像用来呈现到swapchain
* `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`：图像用作内存拷贝

`initialLayout`来指定渲染前图像的像素内存布局，`finalLayout`指定渲染结束后图像像素的内存布局，很明显不用关心渲染图像像素内存布局，后续还会使用常量清理。渲染结束后需要呈现到swapchain，因此指定`VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`即可自动转换为合适的图像内存布局。

### 子批次和附件引用(subpass and attachment references)
一个渲染批次可以包含多个子渲染批次，子批次是根据前一批次渲染framebuffer内容来进行此次渲染，例如系列后处理特效(post-processing effects)一个接一个执行。如果将这些操作组织到一个渲染批次，则Vulkan可以重排操作顺序并节省内存带宽以达到更好的性能。这里只是用单个渲染批次。

每个子渲染批次引用一个或多个附件，代码如下：

```C++
VkAttachmentReference colorAttachmentRef = {};
colorAttachmentRef.attachment = 0;
colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
```

`attachment`参数指定通过指向附件描述数组的索引来引用。当前数组只包含一个`VkAttachmentDescription`，因此索引为`0`。`layout`指定在子批次渲染过程中使用引用到的附件所用的布局。Vulkan在子批次开始之前会自动将附件转换为对应的布局，这里将使用此附件当做颜色缓存，因此使用的布局参数为最佳选择。

子批次通过以下代码来描述：

```C++
VkSubpassDescription subpass = {};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;    // 这里还以指定为计算子批次
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &colorAttachmentRef;
```
在这个描述数组里的索引直接关联片元shader指令`layout(location = 0) out vec4 outColor`。

以下还有几种附件可以被子批次索引：

* `pInputAttachments`：从shader里读取的附件
* `pResolveAttachments`：用在多重采样里的颜色附件
* `pDepthStencilAttachment`：用在深度和裁剪数据的附件
* `pPreserveAttachments`：在此子批次不使用，但需要保留

### 渲染批次
创建渲染批次还是通过其创建信息结构体来实现，创建信息需要传递附件数组和子渲染批次。

```C++
VkRenderPassCreateInfo renderPassInfo = {};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
renderPassInfo.attachmentCount = 1;
renderPassInfo.pAttachments = &colorAttachment;
renderPassInfo.subpassCount = 1;
renderPassInfo.pSubpasses = &subpass;

if (vkCreateRenderPass(device, &renderPassInfo, nullptr, &renderPass) != VK_SUCCESS) {
    throw std::runtime_error("failed to create render pass!");
}
```

最后需要在管线销毁后销毁：

```C++
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);
    ...
}
```

## 结论
现在可以将前序创建的结构体和对象组合起来创建图形管线，前面涉及的内容如下：

* Shader 阶段：定义了图像管线的可编程阶段
* 固定管线状态：定义管线固定阶段的所有结构体，比如输入汇编(input assembly)，光栅化(rasterizer)，视窗(viewport) 以及颜色融合(color blend)
* 管线布局：shader执行阶段引用的`uniform`和变量
* 渲染批次：管线阶段引用的附件(attachment)以及它们的使用方式

现在来创建图形管线：

```C++
VkGraphicsPipelineCreateInfo pipelineInfo = {};
pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
pipelineInfo.stageCount = 2;
pipelineInfo.pStages = shaderStages;

pipelineInfo.pVertexInputState = &vertexInputInfo;
pipelineInfo.pInputAssemblyState = &inputAssembly;
pipelineInfo.pViewportState = &viewportState;
pipelineInfo.pRasterizationState = &rasterizer;
pipelineInfo.pMultisampleState = &multisampling;
pipelineInfo.pDepthStencilState = nullptr; // Optional
pipelineInfo.pColorBlendState = &colorBlending;
pipelineInfo.pDynamicState = nullptr; // Optional

pipelineInfo.layout = pipelineLayout;       // 固定阶段状态设置

pipelineInfo.renderPass = renderPass;       // 渲染批次
pipelineInfo.subpass = 0;                   // 子批次索引

pipelineInfo.basePipelineHandle = VK_NULL_HANDLE; // Optional
pipelineInfo.basePipelineIndex = -1; // Optional
```
最后的`basePipelineHandle`和`basePipelineIndex`用来选择继承其它已经存在的管线创建信息，一个用来直接使用引用，另一个通过索引来继承。继承的好处是共用代码，同时从同一个父类管线转换速度更快。要使用继承功能需要开启`flags`变量的`VK_PIPELINE_CREATE_DERIVATIVE_BIT`位。

最终创建图形管线：

```C++
if (vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &graphicsPipeline) != VK_SUCCESS) {
    throw std::runtime_error("failed to create graphics pipeline!");
}
```

相比其他创建函数，`vkCreateGraphicsPipelines`有更多的参数，主要为了实现指定多个创建信息，创建过个图形管线。第二个参数类型是`VkPipelineCache`，主要用在创建图形管线时使用缓存对象，来加快创建速度。最后需要在管线布局销毁之前销毁。

```C++
void cleanup() {
    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```


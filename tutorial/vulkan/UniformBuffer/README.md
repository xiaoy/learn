# Uniform buffers
## Discriptor layout and buffer
### 简介
当前我们可以为顶点shader上的每个顶点传递任意属性，如果是所有顶点共用的属性呢？在后面的3d编程中将使用`model-view-projection`矩阵。我们可以传递到每个顶点，但是这种方式浪费内存，并且当我们修改此矩阵后，所有的顶点属性都需要更新。这种变换矩阵几乎每帧都会修改。

在Vulkan中使用*资源描述符(resource descriptors)*是解决此问题的正确方式。描述符是shader自由访问例如buffers和images一种方式。我们将创建一块包含转换矩阵的buffer然后让shader通过描述符来访问。使用描述符包含三个部分：

* 在管线创建时指定描述符布局(descriptor layout)
* 从描述符池分配描述符设置(descriptor set)
* 在渲染时绑定描述符设置

*描述符布局* 指定了将被管线访问的资源类型，正如渲染批次申明了将被访问的附件(attachment)类型。*描述符设置*申明了将要被正真绑定到描述符上的buffer 或 image 资源，就像framebuffer申明了正真绑定到渲染批次附件上的image views。*描述符设置*类似顶点buffer和framebuffer那样绑定到渲染命令。

有很多种类的描述符，这里我们只用到`uniform buffer object(UBO)`。虽说描述符种类不同，但使用流程相同。我们想让shader访问到如下的结构体。

```C++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

然后将数据拷贝到`VKBuffer`，然后通过shader里的`UBO`描述符来访问数据：

```C++
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

通过每帧修改`UBO`数据来实现模型的各种变换。

### Vertex shader

最终的顶点shader 如下：

```C++
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```
关键字`uniform`，`in`，`out` 分别代表显卡设备里不同的内存区域，`binding`来指定内存插槽访问索引，每个内存插槽的大小是`vec4`，对应 `16byte`。

### Descriptor set layout
下一步是在C++端定义`UBO`对象，并将通知Vulkan在顶点shader里的描述符。

```C++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

使用`GLM`结构类型定义的数据和shader里的使用的数据类型定义的数据在内存里的表现完全一致，因此我们可以直接使用`memcpy`一个`UniformBufferObject`对象到`VkBuffer`。

我们需要提供在管线创建时shader里使用的每个描述符绑定的细节，正如我们为每个顶点属性和它的位置索引所提供的。这里需要在创建管线前设置好对应的描述符。

```C++
void createDescriptorSetLayout() {
    VkDescriptorSetLayoutBinding uboLayoutBinding = {};
    // 指定绑定位置
    uboLayoutBinding.binding = 0;
    // 绑定类型
    uboLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    // 绑定数量，可以为多个
    uboLayoutBinding.descriptorCount = 1;
    // 指定描述符在渲染管线哪个阶段使用，此处为顶点shader阶段
    uboLayoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
    // 贴图采样相关
    uboLayoutBinding.pImmutableSamplers = nullptr; // Optional

    // 创建描述符布局
    VkDescriptorSetLayoutCreateInfo layoutInfo = {};
    layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
    layoutInfo.bindingCount = 1;
    layoutInfo.pBindings = &uboLayoutBinding;

    // descriptorSetLayout 为描述符绑定要使用的变量类型为：VkDescriptorSetLayout
    if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &descriptorSetLayout) != VK_SUCCESS) {
        throw std::runtime_error("failed to create descriptor set layout!");
    }
}
```
在管线创建时指定shader要使用的描述符布局：

```C++
VkPipelineLayoutCreateInfo pipelineLayoutInfo = {};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 1;
pipelineLayoutInfo.pSetLayouts = &descriptorSetLayout;
```

这里可能疑惑为啥可以指定多个描述符布局？上面不是可以做到一个描述符里面包含多个绑定吗？在下一章节中关于描述符池和描述符设置时会说明这个问题。

描述符布局应给一直保留，因为可能新建别的管线。直到程序退出时清理：

```C++
void cleanup() {
    cleanupSwapChain();

    vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);

    ...
}
```

### Uniform buffer
下一章中将指定shader中将使用包含`UBO`数据的buffer，首先的创建此buffer。每帧都将把新数据拷贝到`UBO` buffer中，这里使用*过渡(staging)* buffer 没有意义。在这种情况下只会增加性能开销而不是提升性能。

我们需要多个buffer，因为可能导致多帧同时意见冲突，我们不想在为下一帧准备内容时，上一帧还在读取当前帧的内容。因此需要为每一帧或每一个swapchain image 准备一个`uniform buffer`。因为要在每个`swapchain image`的`command buffer`里设定对应的`uniform buffer`，所以要为每个`swapchain image`准备一个`uniform buffer`。

为此先创建`buffer`和内存：

```C++
VkBuffer indexBuffer;
VkDeviceMemory indexBufferMemory;

std::vector<VkBuffer> uniformBuffers;
std::vector<VkDeviceMemory> uniformBuffersMemory;
```

然后创建`buffer`:

```C++
void createUniformBuffers() {
    VkDeviceSize bufferSize = sizeof(UniformBufferObject);

    uniformBuffers.resize(swapChainImages.size());
    uniformBuffersMemory.resize(swapChainImages.size());

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        createBuffer(bufferSize, VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, uniformBuffers[i], uniformBuffersMemory[i]);
    }
}
```

因为要在每帧更新`buffer`里内容，所以这里不会调用`vkMapMemory`来映射内存。因为和`swapchain image`绑定，所以在其销毁时，也需要销毁此`buffer`，`swapchain image`重建时也需要重建。

```C++
// 清除swapchain
void cleanupSwapChain() {
    ...

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        vkDestroyBuffer(device, uniformBuffers[i], nullptr);
        vkFreeMemory(device, uniformBuffersMemory[i], nullptr);
    }
}

// 重建swapchain
void recreateSwapChain() {
    ...

    createFramebuffers();
    createUniformBuffers();
    createCommandBuffers();
}
```

### Updating uniform data
在`drawFrame`里根据对应`swapchain image`索引来更新`uniform data`。

```C++
void drawFrame() {
    ...

    uint32_t imageIndex;
    VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

    ...

    updateUniformBuffer(imageIndex);

    VkSubmitInfo submitInfo = {};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

    ...
}

...

void updateUniformBuffer(uint32_t currentImage) {

}
```

首先包含要使用的头文件：

```C++
#define GLM_FORCE_RADIANS
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>

#include <chrono>
```
`glm/gtc/matrix_transform.hpp`包含模型转换`glm::rotate`，view 转换 `glm::lookAt`，projection 转换`glm::perspective`。`GLM_FORCE_RADIANS`宏强制使用弧度为单位，以免造成误解。

`chrono`高精度时间相关头文件。用来让模型每秒渲染90度，和帧率无关。

```C++
void updateUniformBuffer(uint32_t currentImage) {
    static auto startTime = std::chrono::high_resolution_clock::now();

    auto currentTime = std::chrono::high_resolution_clock::now();
    float time = std::chrono::duration<float, std::chrono::seconds::period>(currentTime - startTime).count();

    UniformBufferObject ubo = {};
    // 将模型在z轴方向旋转 time * glm::radians(90.0f)
    ubo.model = glm::rotate(glm::mat4(1.0f), time * glm::radians(90.0f), glm::vec3(0.0f, 0.0f, 1.0f));
    // 第一个参数为 eye的位置
    // 第二个参数为 eye指向的方向
    // 第三个参数为 向上的方向
    ubo.view = glm::lookAt(glm::vec3(2.0f, 2.0f, 2.0f), glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f, 0.0f, 1.0f));

    // 第一个参数为竖向角度
    // 横向和竖向的比例
    // 近端距离
    // 远端距离
    ubo.proj = glm::perspective(glm::radians(45.0f), swapChainExtent.width / (float) swapChainExtent.height, 0.1f, 10.0f);

    // 因为glm为opengl设计，所以在裁剪坐标系内，y方向是反的，所以需要转换回来
    ubo.proj[1][1] *= -1;

    // 将内存映射到buffer
    void* data;
    vkMapMemory(device, uniformBuffersMemory[currentImage], 0, sizeof(ubo), 0, &data);
        memcpy(data, &ubo, sizeof(ubo));
    vkUnmapMemory(device, uniformBuffersMemory[currentImage]);

}
```
 这种使用`UBO`的方式不是最高效的将频繁改变的数据传递到shader。更加高效的方式是通过`push constants`的方式将一小片buffer数据传送到shader。

 ## Descriptor pool and sets

 ### 简介
上一章介绍了描述符布局，用来描述绑定不同类型的描述符。这一章将为每个`VkBuffer`资源创建对应的描述符设置来绑定到`uniform buffer descriptor`。

### 描述符池
描述符设置不可以直接创建，必须像`command buffer` 那样从池子分配。创建函数`createDescriptorPool`来设置：

```C++
void createDescriptorPool() {
    VkDescriptorPoolSize poolSize = {};
    // 描述符类型
    poolSize.type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    // 个数
    poolSize.descriptorCount = static_cast<uint32_t>(swapChainImages.size());

    // 创建池子需要的信息
    VkDescriptorPoolCreateInfo poolInfo = {};
    poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
    poolInfo.poolSizeCount = 1;
    poolInfo.pPoolSizes = &poolSize;
    // 最大描述符数量
    poolInfo.maxSets = static_cast<uint32_t>(swapChainImages.size());;

    // descriptorPool 类型为 VkDescriptorPool
    if (vkCreateDescriptorPool(device, &poolInfo, nullptr, &descriptorPool) != VK_SUCCESS) {
        throw std::runtime_error("failed to create descriptor pool!");
    }
}
```

描述符池和`swapchain`相关，所以伴随`swapchain`的改变而改变：

```C++
void cleanupSwapChain() {
    ...

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        vkDestroyBuffer(device, uniformBuffers[i], nullptr);
        vkFreeMemory(device, uniformBuffersMemory[i], nullptr);
    }

    vkDestroyDescriptorPool(device, descriptorPool, nullptr);
}

void recreateSwapChain() {
    ...

    createUniformBuffers();
    createDescriptorPool();
    createCommandBuffers();
}
```

### Descriptor set
现在可以创建描述符设置：

```C++
void createDescriptorSets() {
    std::vector<VkDescriptorSetLayout> layouts(swapChainImages.size(), descriptorSetLayout);
    VkDescriptorSetAllocateInfo allocInfo = {};
    allocInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO;
    // 指定分配池
    allocInfo.descriptorPool = descriptorPool;
    // 数量
    allocInfo.descriptorSetCount = static_cast<uint32_t>(swapChainImages.size());
    // 传递布局数据
    allocInfo.pSetLayouts = layouts.data();
    
    // descriptorSets 类型为 vector<VkDescriptorSet>
    descriptorSets.resize(swapChainImages.size());
    if (vkAllocateDescriptorSets(device, &allocInfo, descriptorSets.data()) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate descriptor sets!");
    }

    // 分配好后，来设置和UBO的关联

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        VkDescriptorBufferInfo bufferInfo = {};
        bufferInfo.buffer = uniformBuffers[i];
        bufferInfo.offset = 0;
        bufferInfo.range = sizeof(UniformBufferObject);

        VkWriteDescriptorSet descriptorWrite = {};
        descriptorWrite.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
        descriptorWrite.dstSet = descriptorSets[i];
        descriptorWrite.dstBinding = 0;
        // 描述符可能是数组，此参数为数组起始索引值
        descriptorWrite.dstArrayElement = 0;
        descriptorWrite.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
        // 因为可以设置数组里的多个描述符，所以这里指定数量
        descriptorWrite.descriptorCount = 1;
        // buffer 相关    
        descriptorWrite.pBufferInfo = &bufferInfo;
        // image 相关
        descriptorWrite.pImageInfo = nullptr; // Optional
        // buffer view 相关
        descriptorWrite.pTexelBufferView = nullptr; // Optional

        // 接收两种类型的参数，分别为 VkWriteDescriptorSet 和 VkCopyDescriptorSet
        // 当前使用写入描述符
        vkUpdateDescriptorSets(device, 1, &descriptorWrite, 0, nullptr);
    }
}
```

### Using desciptor sets
现在需要在函数`createCommandBuffers`里为每个`swapchain image`绑定描述符设置。这个绑定需要在`vkCmdDrawIndexed`前调用：

```C++
// 参数二：指定管线类型
// 参数三：描述符设定依赖的管线布局
// 参数四：首个描述符设定的索引值
// 参数五：描述符设定个数
// 参数六：描述符设定数组
// 参数七：动态描述符设定起始偏移
vkCmdBindDescriptorSets(commandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout, 0, 1, &descriptorSets[i], 0, nullptr);
vkCmdDrawIndexed(commandBuffers[i], static_cast<uint32_t>(indices.size()), 1, 0, 0, 0);
```
不像顶点和索引缓冲区，描述符不是图形管线独有的。因此需要指定是为图形管线还是计算管线绑定描述符。

现在运行程序发现没有任何东西渲染出来，主要原因是倒置了Y坐标致使按照逆时针方向渲染三角形，这样当前三角形就变成了背面三角形，由于我们在创建管线时剔除了背面的三角形，因此渲染结果是空的。所以在创建管线时修改绘制三角形方向：

```C++
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_COUNTER_CLOCKWISE;
```

### Alignment requirements
现在考虑C++定义的数据结构如何和shader里定义的数据结构匹配，当前使用的结构体如下：

```C++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

两边定义的结构体完全匹配，但是如下的定义则不匹配：

```C++
struct UniformBufferObject {
    glm::vec2 foo;
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    vec2 foo;
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

这就是不满足Vulkan的对齐要求，要求如下：

* 单变量必须对齐到`N`(=4bytes 等价于 32bit浮点)
* `vec2`必须对齐到`2N`(=8bytes)
* `vec3`或`vec4`必须对齐到`4N`(=16bytes)
* 网状结构体根据首个成员变量长度，然后让整个结构体对齐到16的倍数
* `mat4`和`vec4`对齐需求一致

第二个结构体未对齐的说明如下：

* `glm::vec2` 占用8byte
* `glm::mat4` 从8byte开始，到72byte结束
* `glm::mat4` 从72byte开始，到136byte结束
* `glm::mat4` 从136byte开始，到190byte结束

因此都不是16的整数倍，所以到shader里取值是错误的，导致渲染结果错误。C++11中提供了关键字`alignas`来解决此问题：

```C++
struct UniformBufferObject {
    glm::vec2 foo;
    alignas(16) glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

幸运的是GLM提供了宏`GLM_FORCE_DEFAULT_ALIGNED_GENTYPES`来强制对齐到16的倍数：

```C++
#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEFAULT_ALIGNED_GENTYPES
#include <glm/glm.hpp>
```

但是到遇到网状结构体，使用GLM的对齐方式又出现问题：

```C++
// C++ 中定义
struct Foo {
    glm::vec2 v;
};

// 这里按照GLM的强制对齐，f1占用16byte，f2占用16byte
struct UniformBufferObject {
    Foo f1;
    Foo f2;
};

// shader 中定义
struct Foo {
    vec2 v;
};

// 根据成员一的长度为 8byte，成员二为8byte，整体为16的倍数
layout(binding = 0) uniform UniformBufferObject {
    Foo f1;
    Foo f2;
} ubo;
```

所以最后的结论还是需要自己手动对齐：

```C++
struct UniformBufferObject {
    Foo f1;
    alignas(16) Foo f2;
};
```

本章使用的结构体手动对齐结果如下：

```C++
struct UniformBufferObject {
    alignas(16) glm::mat4 model;
    alignas(16) glm::mat4 view;
    alignas(16) glm::mat4 proj;
};
```

### Multiple descriptor sets
由于一些结构体和函数的调用需要多个描述符设定，同时绑定多个描述符设定是可行的。当创建管线布局时要为每个描述符设定指定描述符布局。shader里通过以下方式引用对应的描述符设定：

```C++
layout(set = 0, binding = 0) uniform UniformBufferObject { ... }
```
使用此特性可以将描述符放入不同的物件或者共享分开的描述符设置里。通过这种方式在整个渲染批次里不需要重建大多数的描述符从而来提高效率。
[返回](../README.md)
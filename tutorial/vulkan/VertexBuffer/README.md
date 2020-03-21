# 顶点缓冲区

## 顶点缓冲区描述
### 介绍
接下来的章节，将顶点shader里硬编码顶点数据替换为使用在内存里的顶点缓冲区。首先使用最简单的方式创建CPU端可见的顶点缓冲区，然后使用`memcpy`将内存数据拷贝到顶点缓冲区，之后展示如何使用过渡缓冲区(staging buffer)将顶点缓冲区数据拷贝到高性能显卡内存。

### 顶点shader
顶点shader通过关键字`in`来存顶点缓冲区获取顶点数据。

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

`inPosition`和`inColor`变量为*顶点成员(vertex attributes)*。它们定义了在顶点缓冲区中每个顶点的属性，正如使用两个数组指定每个顶点的位置和颜色值。

类似`fragColor`，`layout(location = x)`标记将索引值绑定到输入缓冲区用来后面引用。数据类型非常重要，比如`dvec3`是64bit矢量，使用多了个*插槽(multiple slots)*。意味着在其之后的索引值至少高于2。

```glsl
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```

查看[OpenGL wiki](https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL)) 获取更多信息。

### 顶点数据(vertex data)
因为位置和颜色矢量要用到线性代数类型矢量(vector)和矩阵(matrices)，**GLM**包含我们需要的数据类型。

```C++
#include <glm/glm.hpp>
```

创建结构体`Vertex`包含两种数据类型用在顶点shader里。

```C++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
};
```

**GLM**提供的C++类型和在shader里使用的类型匹配，非常方便。

```C++
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```

使用`Vertex`数据结构定义顶点数据数组。和之前写在shader里的数据一样，但现在是结合到顶点数组。这也称作 *插入(interleaving)* 顶点属性。

### 绑定描述(binding descriptions)
当顶点数据发送到GPU内存，需要通知Vulkan如何将数据格式传递到顶点shader。这里需要两种类型结构体来传达信息。

首个结构体是`VkVertexInputBindingDescription`，通过在`Vertex`结构体添加成员函数来返回正确的值。

```C++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription = {};
        // binding 为数组索引值，索引绑定数组数据
        bindingDescription.binding = 0;
        // stride 指定每个实体到下一个实体的数据长度
        bindingDescription.stride = sizeof(Vertex);
        // 指定渲染节奏
        bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
        return bindingDescription;
    }
};
```

`inputRate` 有如下两种类型：

* `VK_VERTEX_INPUT_RATE_VERTEX`：每个顶点移动一个实体数据(结构体)
* `VK_VERTEX_INPUT_RATE_INSTANCE`：每个instance移动一个实体数据

### 属性描述(attribute descriptions)
每个顶点的数据输入格式确定后需要添加顶点数据的属性描述，使用`VkVertexInputAttributeDescription`结构体来完成，顶点shader里有种类型的数据，顶点和颜色，所以是两个属性。通过添加`Vertex`成员函数来完成：

```C++
#include <array>

...

static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
    std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions = {};

    attributeDescriptions[0].binding = 0;
    attributeDescriptions[0].location = 0;
    attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
    attributeDescriptions[0].offset = offsetof(Vertex, pos);

    attributeDescriptions[1].binding = 0;
    attributeDescriptions[1].location = 1;
    attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
    attributeDescriptions[1].offset = offsetof(Vertex, color);

    return attributeDescriptions;
}
```

`binding`参数描述了数据来源于那个绑定索引指向的数据。

`location` 参数索引顶点shader里的输入`location`指令，顶点shader里在位置0输入的是位置信息，两个32位浮点数。

`format` 参数描述了顶点数据属性类型。稍微有点困扰的是位置和颜色使用同一个枚举类型。以下为shader里常用类型和对应数据格式：

* `float: VK_FORMAT_R32_SFLOAT`
* `vec2: VK_FORMAT_R32G32_SFLOAT`
* `vec3: VK_FORMAT_R32G32B32_SFLOAT`
* `vec4: VK_FORMAT_R32G32B32A32_SFLOAT`

使用的格式，颜色通道数量和shader 数据类型组件元素(components)数量一致。也允许颜色通道数量大于shader组件元素数量，但是多余的会被隐式忽略。如果颜色通道数量小于shader组件元素数量，则BGA组件元素使用默认值`(0, 0, 1)`。颜色类型`(SFLOAT,UNIT,SINT)`和位宽度匹配shader输入类型。

`format`参数隐式的定义了属性数据的字节长度以及`offset`参数指定了每个每个顶点使用的数据偏移。绑定每次加载一个顶点数据，位置参数从步长`0`字节开始加载，使用宏`offsetof`自动计算。

### 管线顶点输入(pipeline vertex input)
然后设置图形管线来接收顶点数据。

```C++
auto bindingDescription = Vertex::getBindingDescription();
auto attributeDescriptions = Vertex::getAttributeDescriptions();

vertexInputInfo.vertexBindingDescriptionCount = 1;
vertexInputInfo.vertexAttributeDescriptionCount = static_cast<uint32_t>(attributeDescriptions.size());
vertexInputInfo.pVertexBindingDescriptions = &bindingDescription;
vertexInputInfo.pVertexAttributeDescriptions = attributeDescriptions.data();
```

现在管线准备接收顶点数据使用`vertices`定义的数据格式传递到顶点shader。当前打开验证层运行程序，会警告没有顶点缓冲区绑定。下一个步骤是创建顶点缓冲区，然后将数据移动到顶点缓冲区，供GPU使用。
## 顶点缓冲区创建
### 简介
在Vulkan里的缓冲区是一个区域内的内存用来存储任意数据用来被显卡读取。它们可以被用来存储顶点数据，也可以用来其它目的。和处理其它Vulkan对象不同的是，缓冲区不会自动分配内存。从前序的章节里已经展示了Vulkan API控制几乎所有事情以及内存管理。

### 缓冲区创建
创建缓存区：

```C++

...

VkBuffer vertexBuffer;

void createVertexBuffer() {
    // 缓冲共享模式，可以单个队列家族独享，或多个队列家族共用，这里只有图形队列家族使用
    VkBufferCreateInfo bufferInfo = {};

    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    // 缓冲长度，单位为字节
    bufferInfo.size = sizeof(vertices[0]) * vertices.size();

    // 缓冲使用目的，可以有多个使用目的，此变量通过位控制
    bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;


    if (vkCreateBuffer(device, &bufferInfo, nullptr, &vertexBuffer) != VK_SUCCESS) {
        throw std::runtime_error("failed to create vertex buffer!");
    }
}
```

缓冲区不依赖swapchain，在渲染命令中使用，所以在最后单独清理。

```C++
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, vertexBuffer, nullptr);

    ...
}
```

### 内存需求
缓冲区已经创建，但没有真正的内存绑定到缓存区。为缓存区分配内存首先查询内存需求，使用函数`vkGetBufferMemoryRequirements`：

```C++
// 三个参数
// size：需求内存大小，单位为字节，可能和bufferInfo.size不同
// alignment：分配的内存区域缓冲区起始步长，依赖于bufferInfo.usage 和 bufferInfo.flags
// memoryTypeBits：适用于缓冲区的内存类型位字段
VkMemoryRequirements memRequirements;
vkGetBufferMemoryRequirements(device, vertexBuffer, &memRequirements);
```

显卡提供不同类型的内存供分配。不同类型的内存满足不同的操作和性能需求。我们需要结合缓存区需求以及应用需求来找到最佳内存类型。创建函数`findMemoryType`来满足需求：

```C++
uint32_t findMemoryType(uint32_t typeFilter, VkMemoryPropertyFlags properties) {
    // VkPhysicalDeviceMemoryProperties 有两个数组：memoryTypes 和 memoryHeaps
    // 堆内存是独立的内存资源就如独立的显存和内存里的交换区当显存用光时使用的资源
    // 不同类型的内存被包含在堆中，此刻只关心内存类型，但不同的堆影响性能 
    VkPhysicalDeviceMemoryProperties memProperties;
    vkGetPhysicalDeviceMemoryProperties(physicalDevice, &memProperties);

    // typeFilter:参数用来指定内存类型的位字段来匹配合适的内存类型
    // memoryTypes:数组包含 VkMemoryType 结构体指定堆和每种内存类型的属性。属性定义了内存特性，比如内存
    // 映射，以便从CPU端将数据写入，此属性通过 VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT，但是也需要
    // VK_MEMORY_PROPERTY_HOST_COHERENT_BIT 属性，后续解释

    for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++) {
        if ((typeFilter & (1 << i)) && (memProperties.memoryTypes[i].propertyFlags & properties) == properties) {
            return i;
        }
    }

    throw std::runtime_error("failed to find suitable memory type!");
}
```

因为有多个属性，所以不仅要判断结果是否为零，还需要判断属性是否匹配。

### 内存分配
已经获取到正确内存类型，因此内存的分配只要指定类型和大小即可，两个参数都继承自顶点缓冲区的内存和属性需求。

```C++
VkBuffer vertexBuffer;
VkDeviceMemory vertexBufferMemory;

...
    VkMemoryAllocateInfo allocInfo = {};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
    if (vkAllocateMemory(device, &allocInfo, nullptr, &vertexBufferMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate vertex buffer memory!");
    }

    // 将内存绑定到顶点缓冲区
    // 参数四：指定分配的内存在此内存区域的偏移量，通常为零
    // 如果不为零，则需要被memRequirements.aligment 整除
    vkBindBufferMemory(device, vertexBuffer, vertexBufferMemory, 0);
```

正如C++内存分配，销毁。当绑定在顶点缓冲区的内存不再使用时，也需要释放。

```C++
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, vertexBuffer, nullptr);
    vkFreeMemory(device, vertexBufferMemory, nullptr);
    ...
}
```

### 填充顶点缓冲区
通过`vkMapMemory`将缓冲区内存映射到CPU可以访问的内存。

```C++
void* data;
// 此函数允许我们访问指定内存起始偏移量位置和大小的内存
// 参数三，指定内存起始偏移量
// 参数四，指定内存大小
// 参数五，指定标志量，此处默认为0即可
// 参数六，绑定的映射内存，CPU同此映射内存访问顶点缓存区内存
vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
    memcpy(data, vertices.data(), (size_t) bufferInfo.size);
vkUnmapMemory(device, vertexBufferMemory);
```
调用了`memcpy`后，驱动不一定立刻将数据拷贝到缓冲区内存，比如缓存原因。也有可能与之关联的缓冲区内存对于映射的CPU内存不可见。两种方式解决问题：

* 使用和主机(这里指GPU)对应的堆内存，使用`VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`指定
* 在写入映射内存后，调用`vkFlushMappedMemoryRanges`，在读取映射内存前调用`vkInvalidateMappedMemoryRanges`

这里使用第一种方式，保证映射内存数据和主内存数据对齐。相比显示刷新(flushing)内存数据的方式效率低，下一章会解释也没啥问题。

刷新内存区域或使用内存对齐意味着驱动意识到输入写入了缓冲区，但是并不一定对于GPU是可见的。将数据传输到GPU是后台异步操作，Vulkan说明文档告知我们在下一次`vkQueueSubmit`调用之前保证完成传输。

### 绑定顶点缓冲区
渲染操作时需要绑定顶点缓存区内存。

```C++
vkCmdBindPipeline(commandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);

VkBuffer vertexBuffers[] = {vertexBuffer};
VkDeviceSize offsets[] = {0};
// 参数二指定偏移量
// 参数三指定绑定数量
// 参数五指定从顶点缓存读取数据的偏移量
vkCmdBindVertexBuffers(commandBuffers[i], 0, 1, vertexBuffers, offsets);

vkCmdDraw(commandBuffers[i], static_cast<uint32_t>(vertices.size()), 1, 0, 0);
```

## 过渡缓冲区 (Staging buffer)
### 简介
现在使用的顶点缓存区可以正常工作，这种类型缓存区CPU可以正常访问，但对于GPU的读取并不是最合适类型的缓冲区。在独立显卡上，最优化的内存是有`VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT`标志，并且CPU不能直接访问。本节将创建两块顶点缓冲区。一个*过渡缓冲区(staging buffer)* 用来从CPU上传顶点数据，最终的顶点缓冲区在设备本地内存。然后将过渡缓冲区数据拷贝到最终顶点缓冲区。

### 转移队列
缓冲区拷贝命令需要支持转移操作的队列家族，此队列通过`VK_QUEUE_TRANSFER_BIT`来标识。好消息是拥有`VK_QUEUE_GRAPHICS_BIT`或`VK_QUEUE_COMPUTE_BIT`属性的队列家族隐式支持`VK_QUEUE_TRANSFER_BIT`操作。这种情况下，不需要显示的在`queueFlags`列出此标识。

如果喜欢挑战，仍然可以尝试使用不同的队列家族来实现转移操作。需要对程序进行以下修改：

* 修改`QueueFamilyIndices` 和 `findQueueFamilies` 显示查找有`VK_QUEUE_TRANSFER_BIT`位的队列家族，而不是`VK_QUEUE_GRAPHICS_BIT`位的家族队列
* 修改`createLogicalDevice`请求指向转移队列家族的句柄
* 创建第二个命令池用来生成提交到转移队列家族的命令缓冲区
* 修改资源的`sharingMode`属性为`VK_SHARING_MODE_CONCURRENT`，图形和转移队列家族都需要指定
* 提交类似于`vkCmdCopyBuffer`的*转移命令(transfer command)* 到转移队列而不是图形队列

### 抽象缓冲区的创建
因为要创建多个缓冲区，所以应当将创建缓冲区独立成一个函数。新建一个函数`createBuffer`将`createVertexBuffer`中除映射内存的代码移进去。

```C++
void createBuffer(VkDeviceSize size, VkBufferUsageFlags usage, VkMemoryPropertyFlags properties, VkBuffer& buffer, VkDeviceMemory& bufferMemory) {
    VkBufferCreateInfo bufferInfo = {};
    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    bufferInfo.size = size;
    bufferInfo.usage = usage;
    bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(device, &bufferInfo, nullptr, &buffer) != VK_SUCCESS) {
        throw std::runtime_error("failed to create buffer!");
    }

    VkMemoryRequirements memRequirements;
    vkGetBufferMemoryRequirements(device, buffer, &memRequirements);

    VkMemoryAllocateInfo allocInfo = {};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

    if (vkAllocateMemory(device, &allocInfo, nullptr, &bufferMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate buffer memory!");
    }

    vkBindBufferMemory(device, buffer, bufferMemory, 0);
}
```

然后`createVertexBuffer`改写为：

```C++
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();
    createBuffer(bufferSize, VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, vertexBuffer, vertexBufferMemory);

    void* data;
    vkMapMemory(device, vertexBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, vertexBufferMemory);
}
```

### 使用过渡缓冲区
这里通过修改`createVertexBuffer`来创建主机(host)可见的缓冲区当做零时缓冲区，然后创建本地设备(device local)缓冲区当做真正的顶点缓冲区。

```C++
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);
}
```

现在是用绑定`stagingBufferMemory`的`stagingBuffer`用来映射和拷贝顶点数据。本章使用两个新的缓冲区使用标志：

* `VK_BUFFER_USAGE_TRANSFER_SRC_BIT`：当做转移操作的源缓冲区
* `VK_BUFFER_USAGE_TRANSFER_DST_BIT`：当做转移操作的目标缓冲区

`vertexBuffer`创建在设备本地，因此不可以使用`memcpy`函数来拷贝数据。但是可以通过将`stagingBuffer`的数据转移到`vertexBuffer`，因此需要指定`stagingBuffer`为源缓冲区，`vertexBuffer`为目标缓冲区。

新建函数`copyBuffer`将`stagingBuffer`上的数据拷贝到`vertexBuffer`。内存转移操作通过使用命令缓冲区来执行，正如绘制命令。因此首先分配零时命令缓冲区。也许此刻你希望为这 *短实时缓冲区(short-lived buffer)* 创建独立的命令池，因为这样的实现可以优化内存分配。在这种情况下生成内存池使用标志`VK_COMMAND_POOL_CREATE_TRANSIENT_BIT`。

```C++
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {
    VkCommandBufferAllocateInfo allocInfo = {};
    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandPool = commandPool;
    allocInfo.commandBufferCount = 1;

    VkCommandBuffer commandBuffer;
    vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);

    VkCommandBufferBeginInfo beginInfo = {};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;
    // --------------------------
    // 录制命令开始
    vkBeginCommandBuffer(commandBuffer, &beginInfo);

    VkBufferCopy copyRegion = {};
    copyRegion.srcOffset = 0; // Optional
    copyRegion.dstOffset = 0; // Optional
    copyRegion.size = size;
    // 拷贝转移缓冲区内容
    // 使用copyRegion 传递拷贝前后的参数
    vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);

    vkEndCommandBuffer(commandBuffer);
    // 录制命令结束
    // ---------------------------
    VkSubmitInfo submitInfo = {};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
    submitInfo.commandBufferCount = 1;
    submitInfo.pCommandBuffers = &commandBuffer;

    // 提交命令，并等待完成
    vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
    vkQueueWaitIdle(graphicsQueue);

    vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
}
```

现在可以完成整个流程，从过渡缓冲区内容拷贝到顶点缓冲区。

```C++
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);

    copyBuffer(stagingBuffer, vertexBuffer, bufferSize);

    // 然后清理过渡缓冲区
    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
```

### 结论
在真正的应用中，不可能为每个独立的缓冲区申请调用`vkAllocateMemory`。最大同时存在内存申请数量被物理设备的`maxMemoryAllocationCount`参数所限制。就算在 *NVIDIA GTX1080* 设备上此参数只有`4096`。合理的方式是申请一块缓存区，然后通过`offset`来为每个对象划定使用区域。可以自己实现分配器，也可以使用*GPUOpen*提供的[VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator)库。

## 索引缓冲区 (Index buffer)
### 简介
当绘制3d网格时，多个三角形总是可以共享顶点。甚至绘制一个矩形时都有共用顶点。

![](../res/vertex_vs_index.svg)

绘制矩形需要两个三角形，6个顶点。其中两个顶点是共用的，导致50%的冗余。当绘制复杂网格式，平均每个顶点被三个三角形共用。解决顶点冗余的办法是使用顶点索引缓冲区。 

顶点索引缓冲区是一组指向顶点缓存区的数组变量。用来重排顶点数据，以及让多个顶点复用一个顶点数据。上图说明使用顶点索引来描述矩形所使用三角形顶点。

### 顶点缓冲区创建
首先修改之前的顶点数据:

```C++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}},
    {{-0.5f, 0.5f}, {1.0f, 1.0f, 1.0f}}
};
```

然后创建索引数据来描述矩形：

```C++
// 可以使用 unit16_t 或 unit32_t，具体和顶点索引数量相关
const std::vector<uint16_t> indices = {
    0, 1, 2, 2, 3, 0
};
```

最后创建索引缓冲区：

```C++
void createIndexBuffer() {
    VkDeviceSize bufferSize = sizeof(indices[0]) * indices.size();

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
    memcpy(data, indices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_INDEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, indexBuffer, indexBufferMemory);

    copyBuffer(stagingBuffer, indexBuffer, bufferSize);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

和创建顶点缓冲区的区别有两点：

* `buffSize` 为索引变量数量乘上单个索引类型长度
* `indexBuffer`使用`VK_BUFFER_USAGE_INDEX_BUFFER_BIT` 而不是`VK_BUFFER_USAGE_VERTEX_BUFFER_BIT`

### 使用索引缓冲区
使用索引缓冲绘制涉及到`createCommandBuffers`两个改变。首先绑定绑定索引缓冲。区别是只能有一个索引缓冲。因为每个顶点集合只能使用唯一一个顶点索引缓冲，所以当顶点数据不同时，需要重新复制顶点数据到顶点缓冲。

```C++
vkCmdBindVertexBuffers(commandBuffers[i], 0, 1, vertexBuffers, offsets);

// 参数三是索引缓冲的起始步长
// 参数四是索引缓冲变量类型
vkCmdBindIndexBuffer(commandBuffers[i], indexBuffer, 0, VK_INDEX_TYPE_UINT16);
```

绑定后需要修改绘制命令，移除`vkCmdDraw`，代替为`vkCmdDrawIndexed`：

```C++
// 参数三为使用instance的数量，因为不使用批次渲染，所以指定为1
// 参数四为使用索引缓冲区的起始偏移
// 参数五为每个索引使用时需要增加的偏移
// 参数六为instance 使用的偏移
vkCmdDrawIndexed(commandBuffers[i], static_cast<uint32_t>(indices.size()), 1, 0, 0, 0);
```

上一章提示应当从单次的内存分配来实现多个资源的分配，但事实上可以走的更远。[Driver developers recommend ](https://developer.nvidia.com/vulkan-memory-management) 建议将多个缓冲，例如顶点和顶点索引缓冲存储到单个`VkBuffer`，通过`vkCmdBindVertexBuffers`命令，使用偏移来访问。在这种情况下，数据缓存的访问更加友好，因为数据在一起。甚至不在同一渲染操作的多个资源也可以共用一块内存区域，并且提供刷新数据的方式。这就是 *aliasing*，一些Vulkan函数有相应的参数来做这件事。

[返回](../README.md)
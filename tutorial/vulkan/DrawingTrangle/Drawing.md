# Drawing
## Framebuffers
在渲染批次创建过程指定的附件通过封装在`VkFramebuffer`来整合。framebuffer对象索引所有代表附件的`VkImageView`对象。当前只有一个颜色附件，然而我们图像来自swapchain，所以要为所有从swapchain里获取的图像创建framebuffer，然后选择合适的一个返回用在渲染时。

定义存储framebuffer的变量如下：
```C++
std::vector<VkFramebuffer> swapChainFramebuffers;
```

创建framebuffer如下：

```C++
for (size_t i = 0; i < swapChainImageViews.size(); i++) {
    VkImageView attachments[] = {
        swapChainImageViews[i]
    };

    VkFramebufferCreateInfo framebufferInfo = {};
    framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
    framebufferInfo.renderPass = renderPass;
    framebufferInfo.attachmentCount = 1;
    framebufferInfo.pAttachments = attachments;
    framebufferInfo.width = swapChainExtent.width;
    framebufferInfo.height = swapChainExtent.height;
    framebufferInfo.layers = 1;

    if (vkCreateFramebuffer(device, &framebufferInfo, nullptr, &swapChainFramebuffers[i]) != VK_SUCCESS) {
        throw std::runtime_error("failed to create framebuffer!");
    }
}
```
首先指定兼容的`renderPass`，兼容framebuffer的批次大概就是使用附件的数量以及附件尺寸一致。

`pAttachments`指向`swapChainImageViews`，`layers`指定framebuffer用到的层数，这里绘制2d图形，所以为一层。
## Command buffers
在Vulkan里绘制操作和内存转移操作不可以直接调用函数，而是使用命令。因此想执行命令需要将所有操作录制到到命令缓存对象中。优点是所有困难的绘制命令可以提前设置并且使用多线程。之后只需告知Vulkan执行命令缓存对象即可。

### 命令池(Command pools)
在我们创建命令缓存前需要创建命令池。命令池管理保存缓存的内存以及分配出的命令缓存对象。命令池类型为`VkCommandPool`。创建命令池如下：

```C++
QueueFamilyIndices queueFamilyIndices = findQueueFamilies(physicalDevice);

VkCommandPoolCreateInfo poolInfo = {};
poolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
poolInfo.queueFamilyIndex = queueFamilyIndices.graphicsFamily.value();
poolInfo.flags = 0; // Optional
```

通过将命令缓存提交到一个设备队列里来执行，比如获取到的图形和呈现队列。每个命令池分配出的命令缓存只能提交到对应单个类型队列。这里录制的命令用来绘制，因此选择图形队列家族。

命令池有两种类型的`flags`：

* `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT`：标记命令缓存使用新的命令录制非常频繁(可能修改内存分配行为)
* `VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT`：允许命令缓存单独录制，没有此标记，所有的命令缓存必须一起重置

这里仅仅在程序开头录制命令缓存，然后在主循环里执行几次，这里不适用标记。创建命令池如下：

```C++
if (vkCreateCommandPool(device, &poolInfo, nullptr, &commandPool) != VK_SUCCESS) {
    throw std::runtime_error("failed to create command pool!");
}
```

### 命令缓存分配(command buffer allocation)
命令缓存是每个framebuffer对应一个，代码如下：

```C++
VkCommandBufferAllocateInfo allocInfo = {};
allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
allocInfo.commandPool = commandPool;
allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
allocInfo.commandBufferCount = (uint32_t) commandBuffers.size();

if (vkAllocateCommandBuffers(device, &allocInfo, commandBuffers.data()) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate command buffers!");
}
```

`level`参数指定是否分配的命令缓存是主要的还是次要的命令缓存。

* `VK_COMMAND_BUFFER_LEVEL_PRIMARY`：可以提交到队列上执行，但不能从别的命令缓存调用
* `VK_COMMAND_BUFFER_LEVEL_SECONDARY`：不能直接提交，但可以从主命令缓存调用

这里不适用次命令缓存的功能，但是可以共用主命令缓存的操作。

### 开始命令缓存录制
录制命令缓存代码如下：

```C++
for (size_t i = 0; i < commandBuffers.size(); i++) {
    VkCommandBufferBeginInfo beginInfo = {};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = 0; // Optional
    beginInfo.pInheritanceInfo = nullptr; // Optional

    if (vkBeginCommandBuffer(commandBuffers[i], &beginInfo) != VK_SUCCESS) {
        throw std::runtime_error("failed to begin recording command buffer!");
    }
}
```

`flags`参数指定如何使用命令缓存，有以下值可使用：

* `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT`：命令缓存执行后立刻录制
* `VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT`：这是次级命令缓存，将单独用在一个渲染批次
* `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT`：命令缓存已经在排队执行中，依然可以再次提交

`pInheritanceInfo`参数只和次命令缓存关联。它指定从正在调用的主命令缓存继承那些状态。

如果命令缓存已经录制过一次，然后调用`vkBeginCommandBuffer`会立马隐式的重置命令缓存。所以在此调用此命令的后续不可能增加新的命令。

### 开始渲染批次
绘制从渲染批次开始：

```C++
VkRenderPassBeginInfo renderPassInfo = {};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
renderPassInfo.renderPass = renderPass;
renderPassInfo.framebuffer = swapChainFramebuffers[i];

renderPassInfo.renderArea.offset = {0, 0};
renderPassInfo.renderArea.extent = swapChainExtent;
```
`renderArea`定义shader加载和执行的区域，在区域外的值未定义。当然和附件尺寸相匹配性能最优化。


```C++
VkClearValue clearColor = {0.0f, 0.0f, 0.0f, 1.0f};
renderPassInfo.clearValueCount = 1;
renderPassInfo.pClearValues = &clearColor;
```
`clearColor`定义了当加载颜色附件时使用参数`VK_ATTACHMENT_LOAD_OP_CLEAR`对应的清除值。

```C++
vkCmdBeginRenderPass(commandBuffers[i], &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);
```

所有录制命令函数都可以通过前缀`vkCmd`前缀来识别，因为所有的函数返回`void`，所以只到完成命令录制都没有错误异常处理。最后一个参数控制使用了渲染批次的绘制命令如何提供，有如下两种情况：

* `VK_SUBPASS_CONTENTS_INLINE`：渲染批次命令将嵌入到主命令缓存中并且没有次要缓存将被执行
* `VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS`：渲染批次命令将从次要命令缓存执行

### 基础绘制命令
绑定图像管线：

```C++
vkCmdBindPipeline(commandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);
```

第二个参数指定是图形还是计算管线。当前已经设定了图形管线要执行的操作以及片元shader使用的附件，接下来则是绘制三角形：

```C++
// 参数一：命令缓存
// 参数二：顶点个数
// 参数三：批次个数，用在批渲染(instanced rendering)，如果不使用设置为1
// 参数四：顶点缓存步长，定义gl_VertexIndex的最小值
// 参数五：批次渲染步长，定义gl_InstanceIndex的最小值
vkCmdDraw(commandBuffers[i], 3, 1, 0, 0);
```

### 结束
渲染批次结束：

```C++
vkCmdEndRenderPass(commandBuffers[i]);
```

最后完成录制命令缓存：

```C++
if (vkEndCommandBuffer(commandBuffers[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to record command buffer!");
}
```
## Rendering and presentation
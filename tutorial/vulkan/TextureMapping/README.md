# Texture mapping
## Images
### 简介
当前的几何体通过每个顶点着色的方式来渲染，这种方式太限制发挥。本章通过图片映射的方式来让几何体看起来更加有趣。3d模型的绘制也要通过这种方式。

添加一张贴图到程序里需要以下几个步骤：

* 创建一个设备内存支持的图像对象
* 使用贴图文件填充片元
* 创建一个图像采样器
* 添加一个结合图像采样器的描述符从贴图里采样颜色

之前已经和图像对象打过交道，但是是通过swapchain扩展自动创建的。这里要自己创建，创建图像并且使用数据填充和顶点buffer的创建相似。首先创建过渡资源，然后使用数据填充，最终拷贝到用来渲染的图像对象。尽管创建过渡图像来做这件事是可行的，但Vulkan同时允许从`VKBuffer`拷贝像素到图像，并且此API在一些硬件上更快速。首先创建此buffer并使用像素数据填充，然后创建图像并将buffer数据拷贝到其中。创建图像和创建buffer区别不大。包括从设备中按照需求查询内存，然后分配内存并绑定。

但图像也有一些不同，图像内容有不同的*布局(layout)* 影响像素在内存里的组织。依照图形硬件的工作方式，简单按行排列像素不能最大发挥性能。例如，当执行某项图像操作时，必须确保图像的像素布局对于此项操作是最优的。以下是当指定渲染批次时指定一些图像布局：

* `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`：优化用来呈现
* `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`：优化作为用来从片元shader里向其写入颜色的附件
* `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL`：优化用来当做转移操作的源数据，例如`vkCommandCopyImageToBuffer`
* `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`：优化用来当做转移操作的目标数据，例如`vkCommndCopyImageToBuffer`
* `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`：用来在shader里采样

转换图像布局最常见的方式是使用`pipeline barrier`。`pipeline barrier`主要用来同步资源的访问，例如确保资源在读取之前已经写入完成，但也可以用来转换布局。当使用`VK_SHARING_MODE_EXCLUSIVE`，`barriers`也可以用来转移队列家族的拥有权。

### 图像库
这里使用来自[std_collection](https://github.com/nothings/stb)的*stb_image*库。优点是只包含一个头文件`stb_image.h`，不需要繁琐的配置。

### 加载贴图
包含头文件：

```C++
#define STB_IMAGE_IMPLEMENTATION
#include <stb_image.h>
```
头文件只包含了函数申明，通过定义宏的方式，将函数展开。创建文件夹`textures`，将贴图放到此文件夹。加载贴图：

```C++
void createTextureImage() {
    int texWidth, texHeight, texChannels;
    stbi_uc* pixels = stbi_load("textures/texture.jpg", &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
    VkDeviceSize imageSize = texWidth * texHeight * 4;

    if (!pixels) {
        throw std::runtime_error("failed to load texture image!");
    }
}
```

通过`STBI_rgb_alpha`来强制加载贴图附带透明通道，虽然jpg没有透明通道，对将来的扩展比较方便。每个像素占用4字节，所以整个贴图大小为`texWidth * texheight * 4`字节。

### 过渡buffer
创建GPU可见的buffer，使用`vkMapMemory`映射到CPU可见，然后将贴图内容拷贝到buffer。

```C++
void createTextureImage() {
    int texWidth, texHeight, texChannels;
    stbi_uc* pixels = stbi_load("textures/texture.jpg", &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
    VkDeviceSize imageSize = texWidth * texHeight * 4;

    if (!pixels) {
        throw std::runtime_error("failed to load texture image!");
    }

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;

    createBuffer(imageSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, imageSize, 0, &data);
        memcpy(data, pixels, static_cast<size_t>(imageSize));
    vkUnmapMemory(device, stagingBufferMemory);

    stbi_image_free(pixels);
}
```

### 纹理图像
尽管可以直接让shader访问buffer里像素值，但使用Vulkan里的图像对象更好。通过使用图像对象的2d坐标系获取颜色值更加容易和快速。纹理的像素在图像对象里称作*texels*，后续将使用此名词。添加两个成员变量。

```C++
VkImage textureImage;
VkDeviceMemory textureImageMemory;
```

创建图像的结构体如下：

```C++
VkImageCreateInfo imageInfo = {};
imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
imageInfo.imageType = VK_IMAGE_TYPE_2D;
imageInfo.extent.width = static_cast<uint32_t>(texWidth);
imageInfo.extent.height = static_cast<uint32_t>(texHeight);
imageInfo.extent.depth = 1;
imageInfo.mipLevels = 1;
imageInfo.arrayLayers = 1;
```

`imageType`是图像类型，用来指定*texels*通过何种坐标系存储在图像。可以创建 1d，2d，3d类型的图像。单维度的图像用来存储数组数据或梯度数据。双维度的图像用来存储纹理，三维度的图像用来存储*voxel volumes*。`extent`来指定每个维度上的*texels*的数量。因为图像是2d图像所以`depth`为1，并且这里不使用`mimmaping`。

```C++
imageInfo.format = VK_FORMAT_R8G8B8A8_SRGB;
```

Vulkan 支持很多类型的图像格式，但这里必须要和buffer里的像素格式保持一致，否则拷贝将失败。

```C++
imageInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
```

`tiling`字段有两种类型：

* `VK_IMAGE_TILING_LINEAR`：正如`pixels`数组那样，texels按照行(row-major)的顺序存储
* `VK_IMAGE_TILING_OPTIMAL`：存储的顺序是为了访问优化而设计的

和图像`layout`不同的是，`tiling`后续不可改变。如果想直接访问贴图存放在内存里的texels，必须使用`VK_IMAGE_TILING_LINEAR`。因为将使用过渡buffer而不是过渡图像，所以没必要。为了shader高效的访问因此使用`VK_IMAGE_TILING_OPTIMAL`。

```C++
imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
```

`initialLayout`只有两个值：

* `VK_IMAGE_LAYOUT_UNDEFINED`：不能被GPU使用，并且首次转移会舍弃texels
* `VK_IMAGE_LAYOUT_PREINITIALIZED`：不能被GPU使用，但首次转移会保留texels

首次保留texels的情况很少，一种情况是，如果将图像当做结合`VL_IMAGE_TILING_LINEAR`布局的过渡图像。在这种情况下，将texel的数据上传此图像然后将此图像当做转移源图像。这里是将图像当做转移的目标图像，然后从buffer里将texel的数据拷贝到此图像。所以使用`VK_IMAGE_LAYOUT_PREINITIALIZED`即可。

```C++
imageInfo.usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
```
这里图像的用处有两个，一是作为转移目标，用来将buffer的数据转移到图像，另一个是要在shader里访问图像的texel。

```C++
imageInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
```

只用在一个队列家族：支持图形的家族(同时也支持转移)。

```C++
imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;
imageInfo.flags = 0; // Optional
```

`samples`标记和多重采样相关。因为只当做附件，使用单个采样即可。`flags`和优化*稀疏(sparse)* 图像有关。稀疏图像是值图像中的一部分区域的数据存储在内存。如果使用3d贴图存储地表，这里可以使用此标志来避免分配内存来存储*无用("air")* 的值。

```C++
if (vkCreateImage(device, &imageInfo, nullptr, &textureImage) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image!");
}
```

有可能`VK_FORMAT_R8G8B8A8_SRGB`格式显卡不支持，这时需要查询显卡支持的格式，然后转换到合适的格式。因为此格式应用非常广泛，并且图像格式的转换非常的麻烦，后面在深度buffer里会创建这样一个系统。

```C++
VkMemoryRequirements memRequirements;
vkGetImageMemoryRequirements(device, textureImage, &memRequirements);

VkMemoryAllocateInfo allocInfo = {};
allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
allocInfo.allocationSize = memRequirements.size;
allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);

if (vkAllocateMemory(device, &allocInfo, nullptr, &textureImageMemory) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate image memory!");
}

vkBindImageMemory(device, textureImage, textureImageMemory, 0);
```

为图像分配内存和为buffer分配内存的方式非常的相似。使用`vkGetImageMemoryRequirements`代替`vkGetBufferMemoryRequirements`，使用`vkBindImageMemory`代替`vkBindBufferMemory`。

因为在后续还要创建多个图像，因此和创建buffer类似，将对应创建图像的代码抽象为函数`createImage`:

```C++
void createImage(uint32_t width, uint32_t height, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory) {
    VkImageCreateInfo imageInfo = {};
    imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
    imageInfo.imageType = VK_IMAGE_TYPE_2D;
    imageInfo.extent.width = width;
    imageInfo.extent.height = height;
    imageInfo.extent.depth = 1;
    imageInfo.mipLevels = 1;
    imageInfo.arrayLayers = 1;
    imageInfo.format = format;
    imageInfo.tiling = tiling;
    imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    imageInfo.usage = usage;
    imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;
    imageInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateImage(device, &imageInfo, nullptr, &image) != VK_SUCCESS) {
        throw std::runtime_error("failed to create image!");
    }

    VkMemoryRequirements memRequirements;
    vkGetImageMemoryRequirements(device, image, &memRequirements);

    VkMemoryAllocateInfo allocInfo = {};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

    if (vkAllocateMemory(device, &allocInfo, nullptr, &imageMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate image memory!");
    }

    vkBindImageMemory(device, image, imageMemory, 0);
}
```

函数参数主要是通过创建不同的图像所需要的变量来决定的。现在`createTextureImage`函数简化为：

```C++
void createTextureImage() {
    int texWidth, texHeight, texChannels;
    stbi_uc* pixels = stbi_load("textures/texture.jpg", &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
    VkDeviceSize imageSize = texWidth * texHeight * 4;

    if (!pixels) {
        throw std::runtime_error("failed to load texture image!");
    }

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(imageSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, imageSize, 0, &data);
        memcpy(data, pixels, static_cast<size_t>(imageSize));
    vkUnmapMemory(device, stagingBufferMemory);

    stbi_image_free(pixels);

    createImage(texWidth, texHeight, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, textureImage, textureImageMemory);
}
```

### 布局转移(layout translations)
因为又要执行命令buffer的录制和执行，因此将此逻辑流程重构为两个函数：

```C++
VkCommandBuffer beginSingleTimeCommands() {
    VkCommandBufferAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandPool = commandPool;
    allocInfo.commandBufferCount = 1;

    VkCommandBuffer commandBuffer;
    vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);

    VkCommandBufferBeginInfo beginInfo{};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

    vkBeginCommandBuffer(commandBuffer, &beginInfo);

    return commandBuffer;
}

void endSingleTimeCommands(VkCommandBuffer commandBuffer) {
    vkEndCommandBuffer(commandBuffer);

    VkSubmitInfo submitInfo{};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
    submitInfo.commandBufferCount = 1;
    submitInfo.pCommandBuffers = &commandBuffer;

    vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
    vkQueueWaitIdle(graphicsQueue);

    vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
}
```

之前在`copyBuffer`函数里使用了此流程，当前此函数简化为：

```C++
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    VkBufferCopy copyRegion{};
    copyRegion.size = size;
    vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);

    endSingleTimeCommands(commandBuffer);
}
```

如果仍然使用buffers，可以写个函数录制命令然后调用`vkCommandCopyBufferToImage`来完成，但此命令对图像内容布局有要求。首先来添加函数转移图像布局：

```C++
void transitionImageLayout(VkImage image, VkFormat format, VkImageLayout oldLayout, VkImageLayout newLayout) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    endSingleTimeCommands(commandBuffer);
}
```

最常用的执行图像内容布局的方式是使用 *image memory barrier*。管线的*barrier*通常用来同步资源的访问，比如保证在读buffer之前，确保其写入完成，但它也可以用来转移图像布局和转移队列家族所属权当使用`VK_SHARING_MODE_EXCLUSIVE`。这里也有等价的*buffer memory barrier*用在buffers。

```C++
VkImageMemoryBarrier barrier{};
barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
// 指向要转移的布局，如果不关心要转移的布局可以使用 VK_IMAGE_LAYOUT_UNDEFINED
barrier.oldLayout = oldLayout;          
barrier.newLayout = newLayout;

// 如果要转移队列家族所属权，需要指定对应家族队列，这里不转移，使用 VK_QUEUE_FAMILY_IGNORED
barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;

// image和subresourceRange指定图像和图像受影响的区域
// 因为没有mipmap并且只有一张图像，所以参数如下
barrier.image = image;
barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
barrier.subresourceRange.baseMipLevel = 0;
barrier.subresourceRange.levelCount = 1;
barrier.subresourceRange.baseArrayLayer = 0;
barrier.subresourceRange.layerCount = 1;
// barriers主要目的是用来同步，因此必须指定那种类型的操作以及所包含的资源在barrier之前发生,
// 以及那种类型的操作以及资源要等待barrier。尽管使用vkQueueWaitIdel手动同步，使用barrier
// 必不可少，此参数的值依赖旧布局和新布局，当得知使用的转移类型，再来设置此参数。
barrier.srcAccessMask = 0; // TODO
barrier.dstAccessMask = 0; // TODO

// 所有类型的管线barriers提交使用相同的函数。
// ----------------------------------------------
// 第二组参数，前一个指定在barrier之前要执行操的对应的管线阶段，后一个指定需要等待barrier之后执行操作对应的管线阶段
// barrier之前的管线阶段和之后的管线阶段依赖如何规划资源在管线之前和之后的使用
// ----------------------------------------------
// 第三组参数可以是0或VK_DEPENDENCY_BY_REGION_BIT，后一个参数允许读取已经写入的区域内容，意思就是说还没有完全写完
// 最后三组参数索引三种管线barriers的数组：memory barriers，buffer memory barriers, image memory barriers
// 当前不是用参数VkFormat参数，但用来在深度buffer的章节用来特殊的转移。
vkCmdPipelineBarrier(
    commandBuffer,
    0 /* TODO */, 0 /* TODO */,
    0,
    0, nullptr,
    0, nullptr,
    1, &barrier
);
```

### Copying buffer to image
在回到`createTextureImage`之前，需要添加多个帮助函数，比如`copyBufferToImage`：
```C++
void copyBufferToImage(VkBuffer buffer, VkImage image, uint32_t width, uint32_t height) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    endSingleTimeCommands(commandBuffer);
}
```

和buffer的拷贝类似，需要指定要拷贝buffer的区域以及对应图像的区域。使用`VkBufferImageCopy`结构：

```C++
VkBufferImageCopy region{};
// buffer 起始步长
region.bufferOffset = 0;
// buffer 行和列之间的pading
region.bufferRowLength = 0;
region.bufferImageHeight = 0;

// 贴图区域说明
region.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
region.imageSubresource.mipLevel = 0;
region.imageSubresource.baseArrayLayer = 0;
region.imageSubresource.layerCount = 1;

region.imageOffset = {0, 0, 0};
region.imageExtent = {
    width,
    height,
    1
};
```

Buffer到图像的拷贝使用`vkCmdCopyBufferToImage`函数：

```C++
vkCmdCopyBufferToImage(
    commandBuffer,
    buffer,
    image,
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    1,
    &region
);
```

第四个参数指定当前图像使用的布局。这里假设图像已经转换到合适的布局，用来拷贝像素。此刻将一片像素数据拷贝到整个贴图，但也可以指定一个`VkBufferImageCopy`来从buffer不同的区域通过一个操作拷贝到图像。

### 准备贴图图像
现在具备所有用来设置贴图图像的工具，再次回到`createTextureImage`。最后来创建贴图图像。下一步是将过渡buffer拷贝到贴图图像。这涉及到两步：

* 将贴图图像的布局转换为`VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`
* 执行buffer到图像的拷贝操作

```C++
// 贴图图像创建时使用VK_IAMGE_LAYOUT_UNDEFINED，因为用来拷贝时不关心布局
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL);
copyBufferToImage(stagingBuffer, textureImage, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
```

为了在shader中访问贴图数据，将贴图转换为shader可以访问的布局：

```C++
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);
```

### Transition barrier masks
当前在检测层(validation layer)打开的情况下运行程序，将看到关于在函数`transitionImageLayout`中访问遮罩和管线阶段无效的警报。这里需要根据图像布局来设置参数。

这里有两种情况的转换需要处理：

* 未定义(Undefined)->转移目标(transfer destination)：*转移目标*被写入，不需要等待
* 转移目标(transfer destination)->shader raeading：shader读图像数据需要等待*转移目标*写入完成，特别是片元shader读取，这里要用到贴图

这些规则通过如下访问遮罩和管线阶段来指定：

```C++
VkPipelineStageFlags sourceStage;
VkPipelineStageFlags destinationStage;

if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL) {
    barrier.srcAccessMask = 0;
    barrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;

    sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
    destinationStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
} else if (oldLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL && newLayout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL) {
    barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

    sourceStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
    destinationStage = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
} else {
    throw std::invalid_argument("unsupported layout transition!");
}

vkCmdPipelineBarrier(
    commandBuffer,
    sourceStage, destinationStage,
    0,
    0, nullptr,
    0, nullptr,
    1, &barrier
);
```

在随后的表里可以看到*转移目标*被写入必须发生在*转移管线阶段(pipeline transfer stage)*。因为写入不需要任何等待，这里可以为pre-barrier操作设置空的访问遮罩或最早的管线阶段`VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`。需要备注的是`VK_PIPELINE_STAGE_TRANSFER_BIT`不属于*真正的*的图形或计算管线阶段。只是*pseudo-stage*转移发生的阶段。查看[文档](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkPipelineStageFlagBits.html)来获得更多关于pseudo-stage的信息。

图像将在同一管线阶段被写入然后被图元shader读取，这就是为啥要指定shader要在图元shader阶段读取。

如果需要在将来做更多的转换，可以扩展此函数。

需要注意的一件事是，命令buffer提交的结果在开始时隐含着`VK_ACCESS_HOST_WRITE_BIT`同步。因为`transitionImageLayout`函数使用单个命令执行一个命令buffer，可以使用隐式同步并且设置`srcAccessMask`为`0`，如果在布局的转换中需要`VK_ACCESS_HOST_WRITE_BIT`依赖。这都取决于你是否想明确的表达出来，个人不喜欢这种OpenGL式的隐藏操作。

这里有图像布局的特殊类型支持所有操作，`VK_IMAGE_LAYOUT_GENERAL`。它的问题是不能为所有操作提供最佳性能。在一些情况下需要使用它，比如既当做输入又当做输出图像，或者是在图像提前预初始化布局后读取。

所有的帮助函数提交命令通过队列闲置后按照顺序执行命令。在实际应用中，需要将这些操作组合到一个命令buffer中，然后异步高效的执行，尤其是在`createTextureImage`函数中的转移和拷贝操作。尝试创建`setupCommandBuffer`函数来录制命令，在添加`flushSetupCommands`来执行录制好的命令。最好在贴图映射生效后来做这件事，同时可以检测贴图资源是否依然是设置正确的。

### 清理
完成`createTextureImage`函数后清理过渡buffer以及内存：

```C++
    transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

主贴图一直用到程序最后：
```C++
void cleanup() {
    cleanupSwapChain();

    vkDestroyImage(device, textureImage, nullptr);
    vkFreeMemory(device, textureImageMemory, nullptr);

    ...
}
```

## Image view and sampler
### Texture image view
从之前的swap chain和framebuffer了解到不可以直接访问images而是通过image views 来访问。这里为texture image创建image view。为swap chain 创建image view相对为texture image创建的差别是参数`format`和`image`不同，因此将创建image view的过程单独抽象为独立函数。

```C++
VkImageView createImageView(VkImage image, VkFormat format) {
    VkImageViewCreateInfo viewInfo{};
    viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
    viewInfo.image = image;
    viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
    viewInfo.format = format;
    viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    viewInfo.subresourceRange.baseMipLevel = 0;
    viewInfo.subresourceRange.levelCount = 1;
    viewInfo.subresourceRange.baseArrayLayer = 0;
    viewInfo.subresourceRange.layerCount = 1;

    VkImageView imageView;
    if (vkCreateImageView(device, &viewInfo, nullptr, &imageView) != VK_SUCCESS) {
        throw std::runtime_error("failed to create texture image view!");
    }

    return imageView;
}
```

对于swap chain和texture image创建对应image view如下：
```C++
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

    for (uint32_t i = 0; i < swapChainImages.size(); i++) {
        swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat);
    }
}
```

```C++
void createTextureImageView() {
    textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_SRGB);
}
```

最后在结尾需要销毁image view：

```C++
void cleanup() {
    cleanupSwapChain();

    vkDestroyImageView(device, textureImageView, nullptr);

    vkDestroyImage(device, textureImage, nullptr);
    vkFreeMemory(device, textureImageMemory, nullptr);
```

### Samplers
Shader可以直接从image里读取texels，当它们作为textures时通常不直接读取。Textures通常使用Samplers来访问，Samplers获取texels后经过过滤和转换后再当做运算的颜色。

过滤器解决类似于*过度采样(oversampling)*这种问题。比如一个texture的texels映射到比自己尺寸大的几何片元上。如果近距离观察片元上的texels，就会下边第一张图的效果。

![](../res/texture_filtering.png)

如果通过线性插值结合最近的四个texels，将得到比右边光滑的结果。当然有需要像左边的美术风格(比如Minecraft)，右边的是通用图形需要的效果。Sampler对象用来过滤从texture中读取到的颜色值。

*采样不足(undersampling)*是另一个相反的问题，当texels比片元多的时候。在某个*锐利的(sharp)*的角度观察棋盘在远的方向高频次采样导致出现的*鬼影(artifacts)*。

![](../res/anisotropic_filtering.png)

如图中所示，左边的贴图在远距离变模糊。解决方案使用*各向异性采样(anisotropic filtering)*，同样Sampler来完成工作。

除了这些过滤，Sampler同样关心转换。当读取超过image尺寸外的texels时，Sampler使用自己的*地址模式(addressing mode)*来决定对应的颜色值。以下是一些模式对应的处理方案：

![](../res/texture_addressing.png)

Samplers通过`VkSmamplerCreateInfo`来配置，它来制定过滤和转换信息：

```C++
void createTextureSampler() {
    VkSamplerCreateInfo samplerInfo{};
    samplerInfo.sType = VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO;
    // magFilter和minFilter指定texels被放大或缩小时采用何种方式插值
    // magFilter处理过渡采样问题
    // minFilter处理采样不足问题
    // filter可以选择VK_FILTER_LINEAR和VK_FILTER_NEAREST
    samplerInfo.magFilter = VK_FILTER_LINEAR;
    samplerInfo.minFilter = VK_FILTER_LINEAR;

    // U,V,W是texture坐标定义，对应x,y,z
    // 地址模式分别为：
    // VK_SAMPLER_ADDRESS_MODE_REPEAT：超出image尺寸后重复texture texel
    // VK_SAMPLER_ADDRESS_MODE_MIRRORED_REPEAT：超出image尺寸后，反向重复texture texel
    // VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE：超出image尺寸后，使用最近的边缘颜色
    // VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE：超出image尺寸后，使用最近边缘相反边缘的颜色
    // VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_BORDER：超出image尺寸后，使用固定颜色
    // VK_SAMPLER_ADDRESS_MODE_REPEAT 是最常用的模式，比如砖块，地面
    samplerInfo.addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT;
    samplerInfo.addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT;
    samplerInfo.addressModeW = VK_SAMPLER_ADDRESS_MODE_REPEAT;

    // 这两个参数指定是否开启各向异性过滤，以及过滤使用的最大采样texels
    // 在不考虑性能的情况下，没有不使用的理由，采样数越大渲染质量越好，但是没有图形硬件会超过16，因为超过16后的效果
    // 改变微乎其微
    samplerInfo.anisotropyEnable = VK_TRUE;
    samplerInfo.maxAnisotropy = 16.0f;

    // 当使用clamp to border模式时，要返回固定的颜色，此参数来指定。只能返回黑色或白色以及是否透明，返回值可以是
    // 整形或浮点值，但不可任意指定颜色值
    samplerInfo.borderColor = VK_BORDER_COLOR_INT_OPAQUE_BLACK;

    // 指定是否不使用归一化坐标，如果是VK_FALSE，则坐标范围在[0, 1)范围，否则为[0, texWidth) 和 [0, texHeight]
    samplerInfo.unnormalizedCoordinates = VK_FALSE;

    // 是否开启对比，如果开启后先和一个texel对比，然后使用对比结果来决定使用那个颜色当做后续的参数，将来在shadow map会使用
    samplerInfo.compareEnable = VK_FALSE;
    samplerInfo.compareOp = VK_COMPARE_OP_ALWAYS;

    // mipmap 参数
    samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
    samplerInfo.mipLodBias = 0.0f;
    samplerInfo.minLod = 0.0f;
    samplerInfo.maxLod = 0.0f;

    if (vkCreateSampler(device, &samplerInfo, nullptr, &textureSampler) != VK_SUCCESS) {
        throw std::runtime_error("failed to create texture sampler!");
    }
}
```

Sampler和任何`VkImage`不相关，sampler对象定义了如何从texture中获取颜色值。可以为任何Image指定，无论是 1D,2D,3D。这和老的API的区别是，其将texture image和过滤合并到一个状态。

在不适用sampler的时候，将其销毁：

```C++
void cleanup() {
    cleanupSwapChain();

    vkDestroySampler(device, textureSampler, nullptr);
    vkDestroyImageView(device, textureImageView, nullptr);

    ...
}
```

### Anisotropy device feature
因为*各向异性过滤*是可选择设备特性，所以需要自己请求，在`createLogicalDevice`函数中请求：

```C++
VkPhysicalDeviceFeatures deviceFeatures{};
deviceFeatures.samplerAnisotropy = VK_TRUE;
```

虽然现代显卡不可能不支持，还是需要在`isDeviceSuitable`检查是否有此特性：

```C++
bool isDeviceSuitable(VkPhysicalDevice device) {
    ...

    // 获取硬件支持的特性
    VkPhysicalDeviceFeatures supportedFeatures;
    vkGetPhysicalDeviceFeatures(device, &supportedFeatures);

    return indices.isComplete() && extensionsSupported && swapChainAdequate && supportedFeatures.samplerAnisotropy;
}
```

除了强制使用各向异性过滤，还可以设置不使用：

```C++
samplerInfo.anisotropyEnable = VK_FALSE;
samplerInfo.maxAnisotropy = 1.0f;
```

## Combined image sampler
### 简介
前序章节在*Uniform buffers*中使用了描述符，本章引入新类型描述符：*combined image sampler*。此描述符让shader通过sampler来访问image资源。

首先修改描述符布局，描述符池以及描述符设置来包含*combined image sampler*描述符。之后将texture的坐标加到`Vertex`中，修改fragment shader来从texture中读取颜色，而不是通过顶点颜色插值。

### Updating the descriptors
在函数`createDescriptorSetLayout`中为*combined image sampler*添加`VkDescriptorSetLayoutBinding`。添加在*uniform buffer*的后面即可。

```C++
VkDescriptorSetLayoutBinding samplerLayoutBinding{};
samplerLayoutBinding.binding = 1;
samplerLayoutBinding.descriptorCount = 1;
samplerLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
samplerLayoutBinding.pImmutableSamplers = nullptr;
// 此combined image sampler 用在fragment shader 阶段
// image sampler 也可以用在vertex shader阶段，比如通过高度图扭曲一组顶点
samplerLayoutBinding.stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;

std::array<VkDescriptorSetLayoutBinding, 2> bindings = {uboLayoutBinding, samplerLayoutBinding};
VkDescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = static_cast<uint32_t>(bindings.size());
layoutInfo.pBindings = bindings.data();
```

然后修改分配描述符的池子设置：

```C++
std::array<VkDescriptorPoolSize, 2> poolSizes{};
poolSizes[0].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
poolSizes[0].descriptorCount = static_cast<uint32_t>(swapChainImages.size());
poolSizes[1].type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
poolSizes[1].descriptorCount = static_cast<uint32_t>(swapChainImages.size());

VkDescriptorPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
poolInfo.poolSizeCount = static_cast<uint32_t>(poolSizes.size());
poolInfo.pPoolSizes = poolSizes.data();
poolInfo.maxSets = static_cast<uint32_t>(swapChainImages.size());
```

最后一步是在函数`createDescriptorSets`中将image和sampler资源绑定到描述符设置。

```C++
for (size_t i = 0; i < swapChainImages.size(); i++) {
    VkDescriptorBufferInfo bufferInfo{};
    bufferInfo.buffer = uniformBuffers[i];
    bufferInfo.offset = 0;
    bufferInfo.range = sizeof(UniformBufferObject);

    // 此结构体用来绑定image和sampler
    VkDescriptorImageInfo imageInfo{};
    imageInfo.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    imageInfo.imageView = textureImageView;
    imageInfo.sampler = textureSampler;


    std::array<VkWriteDescriptorSet, 2> descriptorWrites{};

    descriptorWrites[0].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
    descriptorWrites[0].dstSet = descriptorSets[i];
    descriptorWrites[0].dstBinding = 0;
    descriptorWrites[0].dstArrayElement = 0;
    descriptorWrites[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    descriptorWrites[0].descriptorCount = 1;
    descriptorWrites[0].pBufferInfo = &bufferInfo;

    // 设置描述符设置
    descriptorWrites[1].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
    descriptorWrites[1].dstSet = descriptorSets[i];
    descriptorWrites[1].dstBinding = 1;
    descriptorWrites[1].dstArrayElement = 0;
    descriptorWrites[1].descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
    descriptorWrites[1].descriptorCount = 1;
    descriptorWrites[1].pImageInfo = &imageInfo;

    vkUpdateDescriptorSets(device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
}
```

### Texture coordinates
这里还差texture和顶点的映射。

```C++
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
    glm::vec2 texCoord;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription{};
        bindingDescription.binding = 0;
        bindingDescription.stride = sizeof(Vertex);
        bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

        return bindingDescription;
    }

    static std::array<VkVertexInputAttributeDescription, 3> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 3> attributeDescriptions{};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Vertex, pos);

        attributeDescriptions[1].binding = 0;
        attributeDescriptions[1].location = 1;
        attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
        attributeDescriptions[1].offset = offsetof(Vertex, color);

        attributeDescriptions[2].binding = 0;
        attributeDescriptions[2].location = 2;
        attributeDescriptions[2].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[2].offset = offsetof(Vertex, texCoord);

        return attributeDescriptions;
    }
};
```

修改了`Vertex`顶点描述符，这样通过顶点坐标shader将texture坐标传递到fragment shader用来对贴图采样。

```C++
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}}
};
```

贴图坐标`0, 0` 为左上角，`1,1`为右下角。

### Shaders
首先修改顶点shader来通过顶点shader将贴图坐标传递到片元shader。

```GLSL
layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;
layout(location = 2) in vec2 inTexCoord;

layout(location = 0) out vec3 fragColor;
layout(location = 1) out vec2 fragTexCoord;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

正如顶点颜色那样，`fragTexCoord`的值将会被rasterizer光滑插值到四边形上。可以通过fragment shader将贴图顶点按照颜色输出。

```GLSL
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) in vec3 fragColor;
layout(location = 1) in vec2 fragTexCoord;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragTexCoord, 0.0, 1.0);
}
```

看到的结果如下：

![](../res/texcoord_visualization.png)

绿色坐标代表竖坐标系，红色代表横坐标系。黑色和黄色角落代表贴图坐标从`0,0`插值到`1,1`。如果没有更好的选择，在shader编程里，这种方式等价于通用编程里的`print`。

*combined image sampler*描述符在shader里通过sampler uniform来代表。在片元shader里通过如下方式引用。

```GLSL
layout(binding = 1) uniform sampler2D texSampler;
```

最后使用内置函数`texture`来获取贴图的颜色值，使用`sampler`和贴图坐标系当做参数。sampler在背后自动执行过滤和转换。

```GLSL
void main() {
    outColor = texture(texSampler, fragTexCoord);
}
```


[返回](../README.md)
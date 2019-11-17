# Presentation
## 窗口表面(Window surface)
Vulkan是平台无关的API，因此它不能直接和所在平台的窗口系统直接交互。为了将渲染好的结果呈现到窗口，需要Vulkan和窗口系统建立连接，需要使用**WSI**(Windows System Integration)扩展。首先说的扩展是`VK_KHR_surface`，其抛出`VkSurfaceKHR`对象代表表面抽象类型来呈现渲染后的图像。程序中的表面将在我们使用GLFW创建的窗口上。

`VK_KHR_surface`是instance级别的扩展，在前面使用`glfwGetRequiredInstanceExtensions`已经获取了此扩展，并且激活了此扩展。窗口表面需要在instance创建之后创建，因为它影响到物理设备的选择，为啥选择在物理设备选择之后介绍此概念，是因为这是大块内容，如果和基础设定糅合到一起，会使问题更加复杂，单独抽出来说更加合理。Vulkan的表面是可选项，可以选择不创建，所以不需要像OpenGL那样创建不可见窗口。

### 窗口表面创建
尽管`VkSurfaceKHR`是平台无关的，但其创建和平台相关。不同的平台扩展不同，传递的参数也不同。但是GLFW已经将平台相关的代码封装过了，我们只需使用GLFW的函数创建窗口表面即可。
```C++
void createSurface() {
    if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS) {
        throw std::runtime_error("failed to create window surface!");
    }
}
```

### 查询是否支持呈现(presentation)
尽管Vulkan的实现可能支持窗口系统的嵌入，但不意味着系统里的每个显卡设备都支持。因此需要扩展`isDeviceSuitable`来保证有一个设备可以将渲染贴图呈现到窗口表面。因为呈现是队列相关功能，因此需要查找相应队列家族支持将贴图呈现到窗口表面。

很有可能执行渲染命令的队列家族和呈现贴图到窗口的队列家族是不同的，因此我们需要分开指定呈现贴图的队列索引。

```C++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
    std::optional<uint32_t> presentFamily;

    bool isComplete() {
        return graphicsFamily.has_value() && presentFamily.has_value();
    }
};
```

然后在修改`findQueueFamilies`函数来找到合适的队列家族来呈现贴图到窗口表面。检查是否合适的函数为`vkGetPhysicalDeviceSurfaceSupportKHR`。

```C++
VkBool32 presentSupport = false;
// 是否支持呈现贴图到窗口表面
// 输入参数：
// 物理设备
// 队列家族索引
// 窗口表面
// 是否支持布尔值地址
vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
if (presentSupport) {
    indices.presentFamily = i;
}
```

很有可能图形绘制队列家族和呈现家族是同一个队列家族，但是为了健壮性，满足更多不同硬件的特性，这样写兼容性更强。但是如果想提高性能，让同一个队列来做这两件事，也可以通过算法实现。

### 创建呈现队列
创建呈现队列和图形渲染队列方法一样，只是指定的队列家族序列号不一定一样，所以将两个队列家族索引号加入`std::set<uint32_t>`来产生多个队列创建信息。

```C++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamilies = {indices.graphicsFamily.value(), indices.presentFamily.value()};

float queuePriority = 1.0f;
for (uint32_t queueFamily : uniqueQueueFamilies) {
    VkDeviceQueueCreateInfo queueCreateInfo = {};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamily;
    queueCreateInfo.queueCount = 1;
    queueCreateInfo.pQueuePriorities = &queuePriority;
    queueCreateInfos.push_back(queueCreateInfo);
}

createInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
createInfo.pQueueCreateInfos = queueCreateInfos.data();
```

最后再获取呈现队列：

```C+++
vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);
```

## 交换链 (Swap Chain)
Vulkan没有”默认framebuffer“的概念，因此需要一个拥有缓存的框架在图像显示到屏幕之前来供我们渲染。这个框架称作*swap chain*并且必须手动创建。*swapchain*本质上是队列里等待显示到屏幕的图像。我们的应用程序做的事情就是先获取队列里的图片，渲染后在提交到队列。队列如何运行以及从队列里的贴图呈现到屏幕的条件都依赖*swapchain*的设定，但是*swapchain*的通用目标是按照显示器的刷新频率呈现图片。

### 检查 *swapchain*的支持
不是所有的显卡可以将图片直接呈现到显示器，这里有多种原因。比如，安装在服务器上，没有显示设备用来输出。第二个原因是将图片呈现到显示器重度依赖于窗口系统，而*surface*又关联在窗口系统上，*swapchain*不包含在Vulkan的核心框架里。在查询设备支持`VK_KHR_swapchain`这个扩展后，可以开启这个扩展。

首先获得`VkPhysicalDevice`所支持的扩展，然后检查是否包含`VK_KHR_swapchain`，Vulkan的头文件包含了宏`VK_KHR_SWAPCHAIN_EXTENSION_NAME`定义了此扩展。是用宏可以防止拼写错误，也不需要记扩展的具体名字。

### 开启设备扩展
在创建logicdevice的时候指定扩展数量和扩展数据即可开启扩展。

```C++
createInfo.enabledExtensionCount = static_cast<uint32_t>(deviceExtensions.size());
createInfo.ppEnabledExtensionNames = deviceExtensions.data();
```

### swap chain 支持的细节
创建`swapchain`需要考虑以下三种属性：

* 表面基础特性(在swapchain里的最大/最小图像数量，图像的最大/最小高度和宽度)
* 表面格式(像素格式，颜色空间)
* 存在的呈现模式

存储查询到数据，定义数据结构类型如下：

```C++
struct SwapChainSupportDetails {
    VkSurfaceCapabilitiesKHR capabilities;
    std::vector<VkSurfaceFormatKHR> formats;
    std::vector<VkPresentModeKHR> presentModes;
};
```

查询表面特性使用如下函数：

```C++
VKAPI_ATTR VkResult VKAPI_CALL vkGetPhysicalDeviceSurfaceCapabilitiesKHR(
    VkPhysicalDevice                            physicalDevice,
    VkSurfaceKHR                                surface,
    VkSurfaceCapabilitiesKHR*                   pSurfaceCapabilities);
```

接下来查询支持的表面格式，依然是先查询数量，在获取数据：
```C++
VKAPI_ATTR VkResult VKAPI_CALL vkGetPhysicalDeviceSurfaceFormatsKHR(
    VkPhysicalDevice                            physicalDevice,
    VkSurfaceKHR                                surface,
    uint32_t*                                   pSurfaceFormatCount,
    VkSurfaceFormatKHR*                         pSurfaceFormats);
```

最后查询呈现模式，还是先查询数量，在查询数据：

```C++
VKAPI_ATTR VkResult VKAPI_CALL vkGetPhysicalDeviceSurfacePresentModesKHR(
    VkPhysicalDevice                            physicalDevice,
    VkSurfaceKHR                                surface,
    uint32_t*                                   pPresentModeCount,
    VkPresentModeKHR*                           pPresentModes);
```

综合是否支持swapchain扩展，以及是否查询到表面格式数据，以及呈现数据，得到是否可以使用swapchain。

### 为swapchain选择正确的配置
查询到了swapchain支持的特性，还需要选择合适特性。有以下三个选择方面：

* 表面格式 (color depth)
* 呈现模式 (交换图像到屏幕上的条件)
* Swap 尺寸 (在swapchain里的图像分辨率)

#### 表面格式
选择图像颜色频道构成，比如`r8g8b8`,`r8g8b8A8`，其中数字代表每个颜色通道占用位数。这就是图像格式。

图像格式选好了，还需要选择颜色空间。颜色空间(color space)主要用来告诉shader程序如何运算转换颜色数值。

```C++
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats) {
    for (const auto& availableFormat : availableFormats) {
        if (availableFormat.format == VK_FORMAT_B8G8R8A8_UNORM && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR) {
            return availableFormat;
        }
    }

    return availableFormats[0];
}
```

#### 呈现模式
呈现模式就是swapchain叫图像显示到屏幕的方式，Vulkan里有以下四种方式：

* `VK_PRESENT_MODE_IMMEDIATE_KHR`：应用提交的图像立马传输到屏幕，这会导致*撕裂*
* `VK_PRESENT_MODE_FIFO_KHR`：swapchain是一个队列，当显示器刷新时，从队列最前端获取一张图像然后应用程序向队列尾部插入一张新的图像。如果队列满了，应用需要等待。这和现代游戏里的垂直同步很类似，显示器刷新的那一刻被称作"vertical blank"
* `VK_PRESENT_MODE_FIFO_RELAXED_KHR`：这种模式和第二种模式的区别是，当队列为空时，不是等待下一个刷新时刻，而是直接刷新，这时候如果有图像进入队列，会被立即刷新到屏幕，这样就会造成撕裂
* `VK_PRESENT_MODE_MAILBOX_KHR`：这种模式是第二种模式的变种，当队列满的时候，应用不会被阻止提交，而是将队列里的一个图像替换，这种模式可以用来实现三重缓存，这种模式相对双重缓存显著的提升了由于延时导致的同步问题

这样看来三重缓存模式是最佳模式：

```C++
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes) {
    for (const auto& availablePresentMode : availablePresentModes) {
        if (availablePresentMode == VK_PRESENT_MODE_MAILBOX_KHR) {
            return availablePresentMode;
        }
    }

    return VK_PRESENT_MODE_FIFO_KHR;
}
```

#### swap 尺寸
swap 尺寸是swap chain里图像的分辨率，通常和要渲染的目标窗口尺寸一致。可设定的分辨率范围定义在结构体`VkSurfaceCapabilitiesKHR`。Vulkan通过设定成员`currentExtent`的宽度和高度来匹配窗口的分辨率。但是，一些窗口管理器允许自定义swap 尺寸，通过判断`currentExtent`的宽度和高度是否等于`uint32_t`的最大值来得知是否可以返回自定义尺寸。这种情况下，在`minImageExtent`和`maxImageExtent`的边界范围内取得合适的swap分辨率。

```C++
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {
    if (capabilities.currentExtent.width != UINT32_MAX) {
        return capabilities.currentExtent;
    } else {
        VkExtent2D actualExtent = {WIDTH, HEIGHT};

        actualExtent.width = std::max(capabilities.minImageExtent.width, std::min(capabilities.maxImageExtent.width, actualExtent.width));
        actualExtent.height = std::max(capabilities.minImageExtent.height, std::min(capabilities.maxImageExtent.height, actualExtent.height));

        return actualExtent;
    }
}
```

### 创建 swapchain
创建 swapchain的接口如下：

```C++
VKAPI_ATTR VkResult VKAPI_CALL vkCreateSwapchainKHR(
    VkDevice                                    device,
    const VkSwapchainCreateInfoKHR*             pCreateInfo,
    const VkAllocationCallbacks*                pAllocator,
    VkSwapchainKHR*                             pSwapchain);
```

需要`VkSwapchainCreateInfoKHR`对象，结构体如下：

```C++
typedef struct VkSwapchainCreateInfoKHR {
    VkStructureType                  sType;
    const void*                      pNext;
    VkSwapchainCreateFlagsKHR        flags;
    VkSurfaceKHR                     surface;
    uint32_t                         minImageCount;
    VkFormat                         imageFormat;
    VkColorSpaceKHR                  imageColorSpace;
    VkExtent2D                       imageExtent;
    uint32_t                         imageArrayLayers;
    VkImageUsageFlags                imageUsage;
    VkSharingMode                    imageSharingMode;
    uint32_t                         queueFamilyIndexCount;
    const uint32_t*                  pQueueFamilyIndices;
    VkSurfaceTransformFlagBitsKHR    preTransform;
    VkCompositeAlphaFlagBitsKHR      compositeAlpha;
    VkPresentModeKHR                 presentMode;
    VkBool32                         clipped;
    VkSwapchainKHR                   oldSwapchain;
} VkSwapchainCreateInfoKHR;
```

`minImageCount`是指定swapchain里的图像个数。通过查询到swapchain特性可以得知最小支持的图像数量，为了提交效率可以在最小的基础上在加1，但也不能超过最大支持的图像数量。因此指定图像数量代码如下：

```C++
uint32_t imageCount = swapChainSupport.capabilities.minImageCount + 1;
if (swapChainSupport.capabilities.maxImageCount > 0 && imageCount > swapChainSupport.capabilities.maxImageCount) {
    imageCount = swapChainSupport.capabilities.maxImageCount;
}
```

swapchain 创建信息如下：

```C++
VkSwapchainCreateInfoKHR createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
createInfo.surface = surface;
createInfo.minImageCount = imageCount;
createInfo.imageFormat = surfaceFormat.format;
createInfo.imageColorSpace = surfaceFormat.colorSpace;
createInfo.imageExtent = extent;
createInfo.imageArrayLayers = 1;
createInfo.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;
```

`imageArrayLayers`指定每张图像包含的层级。通常是1，除非开发3d立体应用。`imageUsage`位指定我们使用swapchain里的图像做何种操作。这里将swapchain里的图像当做颜色依附(color attachment)。也有可能将渲染图像首先做图像后处理。这种情况下将要使用`VK_IMAGE_USAGE_TRANSFER_DST_BIT`位然后使用内存操作将渲染图像转移到swapchain 图像里。

接下来是根据队列家族选择合适变量。

```C++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);
uint32_t queueFamilyIndices[] = {indices.graphicsFamily.value(), indices.presentFamily.value()};

if (indices.graphicsFamily != indices.presentFamily) {
    createInfo.imageSharingMode = VK_SHARING_MODE_CONCURRENT;
    createInfo.queueFamilyIndexCount = 2;
    createInfo.pQueueFamilyIndices = queueFamilyIndices;
} else {
    createInfo.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;
    createInfo.queueFamilyIndexCount = 0; // Optional
    createInfo.pQueueFamilyIndices = nullptr; // Optional
}
```

当渲染队列家族和呈现队列家族不是同一个队列，需要设定不同队列之间如何处理swapchain的图像。这里有两种处理方式：

* `VK_SHARING_MODE_EXCLUSIVE`：每个图像一次只被一个队列家族所拥有，如果别的家族队列在使用之前需要将权限手动转移过去。这个选项提供最好的性能。
* `VK_SHARING_MODE_CONCURRENT`：图像可以被多个队列家族使用，不需要手动转移权限

可以指定swapchain里的图像变换操作，如果swapchain支持的话(`supportedTransforms` in`capabilities`)，比如90度顺时针旋转或者横向翻转。这里不想翻转的话，指定当前变换即可。

`compositeAlpha`变量指定是否和其它窗口系统的透明通道是否融合。这里选择忽视其它窗口的透明通道。

`clipped`变量指定是否忽略混合其它窗口的像素值，设置为`VK_TRUE`为只使用自身像素值。

`oldSwapchain`变量指定上一个swapchain，当窗口改变大小的时候需要重建swapchain，这时候需要指定之前的swapchain。

```C++
createInfo.preTransform = swapChainSupport.capabilities.currentTransform;
createInfo.compositeAlpha = VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR;
createInfo.presentMode = presentMode;
createInfo.clipped = VK_TRUE;
createInfo.oldSwapchain = VK_NULL_HANDLE;
```

### 获取swapchain 图像
去swapchain里获取`VkImage`，还是同样的方式，先获取数量，在获取内容。

```C++
std::vector<VkImage> swapChainImages;
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, nullptr);
swapChainImages.resize(imageCount);
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, swapChainImages.data());
```

## 贴图视窗 (Image View)
所有`VkImage`，包括swapchain里，都必须封装在`VkImageView`对象里使用。图像视窗描述如何访问图像以及访问图像的那部分，比如将2d贴图当做没有mipmapping 等级的深度图。

这里通过创建贴图视窗将swapchain里的贴图封装为色彩渲染目标。

Image view的创建结构体如下：

```C++
typedef struct VkImageViewCreateInfo {
    VkStructureType            sType;
    const void*                pNext;
    VkImageViewCreateFlags     flags;
    VkImage                    image;
    VkImageViewType            viewType;
    VkFormat                   format;
    VkComponentMapping         components;
    VkImageSubresourceRange    subresourceRange;
} VkImageViewCreateInfo;
```

`viewType`指定图像类型，比如1d贴图，2d贴图，3d贴图，或cube maps。

`components`来指定颜色通道值。比如可以将所有通道指向红色通道，也可以指定`0`或`1`。

`subresourceRange`描述图像如何使用，以及图像的那部分可以访问。

```C++
createInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
createInfo.format = swapChainImageFormat;

createInfo.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;

createInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
createInfo.subresourceRange.baseMipLevel = 0;
createInfo.subresourceRange.levelCount = 1;
createInfo.subresourceRange.baseArrayLayer = 0;
createInfo.subresourceRange.layerCount = 1;
```
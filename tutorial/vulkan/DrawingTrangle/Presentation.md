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

## 贴图视窗 (Image View)
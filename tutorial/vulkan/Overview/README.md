# 总览 
## Vulkan 的起源
Vulkan 被设计为GPU上层的跨平台抽象层。过去的大多数API被设计用来配置GPU的固定功能。程序员不得不按照厂商提供的参数，来输入标准数据，设置相关参数，最后得到渲染结果。

随着显卡架构成熟，他们提供越来越多的可编程功能。这些新功能不得不和已经存在API混在一起，这不仅仅是统一抽象层的问题，更多的是程序员花费大量的精力去猜测显卡驱动层。另外一个问题是不同API，平台之间的shader语法不一致。同时旧的API在多线程方面的限制。Vulkan设计就是来解决这些问题。

## 绘制一个三角形需要哪些步骤
这里通过简单描述，首先让读者有个整体理解。

### Step1 实例和物理设备选择
Vulkan应用通过`VKInstance`设置Vulkan API来启动。实例通过描述应用和你将使用的API扩展来创建。创建好实例后，可以查询Vulkan支持的硬件选择一个或多个`VKPhysicalDevices`来操作。你可以查询类似显存大小，以及设备属性选择合适的硬件。

### Step2 逻辑设备和队列家族
选择正确的硬件设备后，你需要创建`VKDevice`(logic device)，这里你需要更加明确的指出使用那些`VKPhysicalDeviceFeatures`特性，比如多视窗渲染和64bit浮点数。你还要选择使用那个队列家族。大多数的Vulkan操作，比如渲染命令(draw command)以及内存操作(memory operation)，都是提交到`VKQueue`然后被异步执行。队列分配自队列家族，每个队列家族的队列支持执行系列的操作。例如，这里有图形，计算，内存操作的独立队列家族。已有的队列家族是挑选物理硬件的一项指标。每个硬件设备不一定支持所有的Vulkan功能，但是支持Vulkan的显卡都能满足我们感兴趣的队列操作。

### Step3 窗口和交换链
除非你只感兴趣画外(offscreen)的渲染，否则你需要创建窗口来展示渲染图片。通过当前平台接口或[GLFW](http://www.glfw.org/)和[SDL](https://www.libsdl.org/)这样的库来创建窗口。本教程将使用GLFW。

我们将使用两个以上的组件来渲染到窗口：窗口表面(VkSurfaceKHR)以及交换链(VKSwapchainKHR)。**KHR**前缀代表这些对象是Vulkan的扩展对象。Vulkan接口自身对平台完全无知的(agnostic)，这就是为什么我们需要使用标准WSI(Window System Interface)扩展来和窗口管理(Window Manager)交互。表面是跨平台的用来渲染的抽象窗口并且需要提供本地窗口句柄当参数来实例化，例如在windows上`HWND`，幸运的是 GLFW库有内置的函数解决平台相关的细节。

交换链(swap chain)是渲染目标的集合。它的基本目标是我们当前渲染的图像和屏幕上的图像不是同一个。这保证了只有渲染完毕的图像被显示。每当我们要渲染一帧时不得不让交换链提供一张贴图来用作渲染。当我们完成渲染时，贴图返回到交换链在某个时间点呈现到屏幕。渲染目标数量以及呈现完成的贴图到屏幕的条件依赖呈现模式。通用的呈现模式是双缓存(double buffering)或三缓存(triple buffering)。

一些平台允许你使用`VK_KHR_display`和`VK_KHR_display_swapchain`直接渲染显示器而不需要和任何窗口管理。这允许你创建展示整个屏幕的表面以及用来实现你自己的窗口管理。

### Step4 贴图(image view)和帧缓存(framebuffers)
渲染从交换链请求到的贴图，我们需要将其封装为`VKImageView`和`VKFramebuffer`。image view应用到贴图制定部分，framebuffer引用到image views用来当做颜色(color)，深度(depth)，模板(stencil)目标。因为交换链里有许多不同的贴图，我们将为每个贴图创建图像视图(image view)和帧缓存(framebuffer)，并在渲染时选择合适的一个。

### Step5 渲染批次(render passes)
Vulkan的渲染批次描述了在渲染操作中使用的贴图类型，以及如何使用和如何处理它们的内容。在我们的初级三角形渲染程序，我们将告知Vukan，将使用单个贴图当做颜色(color)目标，并且在渲染前使用固定的颜色来清除。这里渲染批次只描述贴图类型，`VKFramebuffer`绑定制定的贴图到插槽。

### Step6 图形管线
Vulkan的图形管线通过创建`VkPipeline`对象来设置。它描述了显卡可配置状态，例如视窗尺寸(viewport size)以及深度缓存操作以及使用`VkShaderModule`对象的可编程状态。`VkShaderModule`对象创建于shader的字节码。驱动也需要知道那个渲染目标将用在图形管线里，我们通过引用渲染批次来指定。

Vulkan和显存的接口最大的特性区别是：所有的图形管线配置需要预先设置。这就意味着如果想选择不同的shader或者轻微改变顶点布局，那就需要重新创建图形管线。这就意味着为了不同组合的渲染操作你需要提前创建多个不同的`VkPipeline`对象。只有一些基础配置，类似于视窗大小和清理颜色，可以动态改变。所有的状态需要显示描述，例如没有默认颜色融合状态。

好消息是因为你做了等价于提前编译而不是及时编译，对于驱动将有更多的优化机会以及运行时的性能更加可预估，因为类似于转换图形管线这样大的状态变化非常明确。

### Step7 命令池(command pool)和命令缓存(command buffers)
许多我们想在Vulkan里执行的操作需要提交到队里执行，例如绘画操作。这些操作在提交到队里之前需要记录在`VkCommandBuffer`。这些命令缓存从`VkCommandPool`里分配，命令池关联与特定的队列家族。为了渲染一个三角形，我们需要命令缓存录制如下操作：

* 开始渲染批次
* 绑定图形管线
* 绘制三个顶点
* 结束渲染批次

因为在帧缓存(framebuffer)里的贴图依赖交换链里给出的制定贴图，我们需要为每个可能的贴图录制命令缓存并且在渲染时选择一个正确的。这导致每一帧需要重新录制命令缓存，这不是高效的。

### Step8 主循环
当前绘制命令被封装在命令缓存中，那主循环就很明显了。我们首先从通过`vkAcquireNextImageKHR`从交换链请求一张贴图。然后我们为这张贴图选择合适的命令缓存通过执行`vkQueueSubmit`。最后我们将贴图返回到交换链然后通过`vkQueuePresentKHR`呈现到屏幕。

提交到队里的操作被异步执行。因此我们将使用同步对象来保证正确的执行顺序，例如`semaphores`对象。执行绘制命令缓存必须设置等待贴图请求完成，否则将发生我们开始渲染的图任然被读取来呈现到屏幕。`vkQueuePresentKHR`调用需要等待渲染完成，为此我们需要使用第二个`semaphore`来在渲染完毕时发射信号。

### 总结
简短的教程给予了对绘制一个三角形需要做的工作基础了解。真实的程序需要更多的步骤，比如分配顶点缓存，创建联合缓存，以及上传贴图，这些都将在子章节里细聊，但是由于学习Vulkan有陡峭的曲线，所以从简单开始。长话短说，绘制一个三角形需要如下步骤：

* 创建 `VkInstance`
* 选择一个支持的显卡(VkPhysicalDevice)
* 创建`VkDevice` 和 `VkQueue`用来绘制和呈现
* 创建 窗口，窗口表面以及交换链
* 封装交换链贴图到`VkImageView`
* 创建一个渲染批次指定渲染目标和如何使用
* 为渲染批次创建帧缓存
* 设置图形管线
* 分配命令缓存，然后为交换链里的每张贴图使用绘制命令来录制命令缓存
* 通过请求贴图，然后提交绘制命令缓存，最终将绘制好的贴图提交到交换链
## API的概念
### 编码规范
所有的Vulkan函数，结构体以及枚举定义在`vulkan.h`文件中，包含在 Vulkan SDK中。

函数的前缀为`vk`，枚举以及结构体类型前缀为`Vk`，枚举值使用`VK`为前缀。API重度使用结构体类型当做函数参数。例如，对象的创建遵循如下模式：

```C++
VkXXXCreateInfo createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_XXX_CREATE_INFO;
createInfo.pNext = nullptr;
createInfo.foo = ...;
createInfo.bar = ...;

VkXXX object;
if (vkCreateXXX(&createInfo, nullptr, &object) != VK_SUCCESS) {
    std::cerr << "failed to create object" << std::endl;
    return false;
}
```

Vulkan里许多需要你通过成员变量`sType`来显示的指定结构体类型。`pNext`成员可以指定扩展结构体，在本教程里总是为`nullptr   `。创建或销毁对象的函数总是有一个`VkAllocateCallbacks`参数允许你使用通用分配器来分配驱动内存，本教程里也为`nullptr`。

几乎所有函数返回`VkResult`类型变量，变量结果为`VK_SUCCESS`或其他错误码。Vulkan说明文档里有详细说明。

### 检测层
Vulkan被设计为高性能和低的驱动消耗。因此默认包含有限的错误检查和调试功能。驱动程序在遇到错误是直接崩溃而不是返回错误码，或者更糟糕的是在你的显卡可以正常运行，在其他显卡运行失败。

Vulkan通过称作*validation layers*开开启扩展检查。检查层是可以插入接口层和驱动层之间的代码来做类似于：对函数参数进行额外的检查以及内存分配问题。特色是在开发调试阶段开启检查，在发布版本关闭这项功能，从而没有额外的性能负担。每个人可以写自己的检查层，但是LunarG开发的Vulkan SDK提供了标准检测层套件将用在本教程。同时需要添加回调函数来接收检查层发来的消息。


[返回](../README.md)
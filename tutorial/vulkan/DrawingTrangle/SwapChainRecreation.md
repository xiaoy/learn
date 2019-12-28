# Swap chain recreation

## 简介
当窗口尺寸修改时，由于swapchain没有伴随修改为相匹配尺寸，从而导致渲染结果错误。因此需要在窗口尺寸修改时，重建交换链。

## 重建swapchain
当重建swapchain时，其他对象的创建依赖swapchain和窗口尺寸，所以需要重建这些对象。

```C++
void recreateSwapChain() {
    // 需要等待上一帧渲染完成，不能修改正在使用的资源，从而导致整个渲染状态错误
    vkDeviceWaitIdle(device);

    // 重建swapchain
    createSwapChain();

    // image views 依赖 swapchain，所以需要重建
    createImageViews();

    // 渲染批次依赖swapchain 贴图格式，虽然贴图格式修改很罕见，依然需要处理这种情况
    createRenderPass();

    // 视窗和裁剪区域在管线创建时指定，所以需要重建，如果使用动态状态来制定视窗和裁剪区域，则不需要重建管线
    createGraphicsPipeline();

    // 帧缓存和命令缓存都依赖swapchain的贴图属性，所以需要重建
    createFramebuffers();
    createCommandBuffers();
}
```

因为在重建之前，需要把旧的对象清理掉，这时涉及到清理函数`cleanup`中一些代码和此处需求重合，所以需要单独抽象出一个函数。

```C++
void cleanupSwapChain() {
    for (size_t i = 0; i < swapChainFramebuffers.size(); i++) {
        vkDestroyFramebuffer(device, swapChainFramebuffers[i], nullptr);
    }

    // 这里只需清除command buffers，不需要重建
    vkFreeCommandBuffers(device, commandPool, static_cast<uint32_t>(commandBuffers.size()), commandBuffers.data());

    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);

    for (size_t i = 0; i < swapChainImageViews.size(); i++) {
        vkDestroyImageView(device, swapChainImageViews[i], nullptr);
    }

    vkDestroySwapchainKHR(device, swapChain, nullptr);
}

void cleanup() {
    cleanupSwapChain();

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    vkDestroyCommandPool(device, commandPool, nullptr);

    vkDestroyDevice(device, nullptr);

    if (enableValidationLayers) {
        DestroyDebugUtilsMessengerEXT(instance, debugMessenger, nullptr);
    }

    vkDestroySurfaceKHR(instance, surface, nullptr);
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

为了正确处理窗口尺寸修改，需要查询当前framebuffer的尺寸来创建匹配的swap chain 图像。修改`chooseSwapExtent`如下：

```C++
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {
    if (capabilities.currentExtent.width != UINT32_MAX) {
        return capabilities.currentExtent;
    } else {
        int width, height;
        glfwGetFramebufferSize(window, &width, &height);

        VkExtent2D actualExtent = {
            static_cast<uint32_t>(width),
            static_cast<uint32_t>(height)
        };

        ...
    }
}
```

这就是重建swap chain 所有步骤！缺点是在重新创建swap chain之前必须停止所有绘制。更佳的处理方式是：在创建新的swap chain时，同时继续让旧的swap chain图像上执行的绘制命令完成。通过设置`VkSwapchainCreateInfoKHR`结构体的参数`oldSwapChain`字段来实现，当旧的swap chain绘制命令完成时，立刻销毁。

## 不是最优的或过时的swapchain
现在需要知道什么时刻需要重建swap chain。Vulkan在呈现阶段会通过返回的参数告知窗口大小和swap chain 图像尺寸不一致。函数`VkSwapchainCreateInfoKHR` 和 `VkSwapchainCreateInfoKHR` 返回如下枚举来说明：

* `VkSwapchainCreateInfoKHR`：swap chain 和表面尺寸不再匹配，不能再用来渲染。通常当窗口改变尺寸后返回此参数
* `VK_SUBOPTIMAL_KHR`：swap chain 依然可以正确呈现到表面，但和表面的尺寸不匹配

```C++
VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

if (result == VK_ERROR_OUT_OF_DATE_KHR) {
    recreateSwapChain();
    return;
} else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
    throw std::runtime_error("failed to acquire swap chain image!");
}
```

如果获取图像时，返回swap chain已过时，则不能再用来呈现。所以需要重建swap chain，并跳转到下一帧渲染。

## 明确处理窗口尺寸修改
在窗口尺寸改变后，尽管许多驱动和平台会自动触发`VK_ERROR_OUT_OF_DATE_KHR`，但是不保证100%触发。这就是为什么需要额外特殊处理窗口尺寸变化。

GLFW 在窗口尺寸修改时会有回调调用，使用`glfwSetFramebufferSizeCallback`来设置回调函数。

```C++
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
    glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
}

static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {

}
```

因为GLFW没有回调C++ 成员函数的方式，所以只能调用静态函数，但同时也可以设置`this`指针，然后在回调函数传递的`GLFWwindow`指针获取`this`指针。

```C++
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
glfwSetWindowUserPointer(window, this);
glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
```

获取指针使用`glfwGetWindowUserPointer`。

```C++
static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {
    auto app = reinterpret_cast<HelloTriangleApplication*>(glfwGetWindowUserPointer(window));
    app->framebufferResized = true;
}
```

最终在`drawFrame`函数来处理窗口修改。

```C++
if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR || framebufferResized) {
    framebufferResized = false;
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    ...
}
```

## 处理窗口最小化
当窗口尺寸变化时，需要处理一种特殊情况，当窗口最小化，这时framebuffer的宽度和高度都为零，所以在`recreateSwapchain`需要特殊处理。

```C++
void recreateSwapChain() {
    int width = 0, height = 0;
    // 获取framebuffer 尺寸
    glfwGetFramebufferSize(window, &width, &height);
    while (width == 0 || height == 0) {
        glfwGetFramebufferSize(window, &width, &height);
        // 休眠cpu，等待特定时间再查询
        glfwWaitEvents();
    }

    vkDeviceWaitIdle(device);

    ...
}
```
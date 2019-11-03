# Setup
## BaseCode
### 通用结构
整个Vulkan程序运行的通用结果，其实也适用于游戏。

```C++
#include <vulkan/vulkan.h>

#include <iostream>
#include <stdexcept>
#include <functional>
#include <cstdlib>

class HelloTriangleApplication {
public:
    void run() {
        initVulkan();
        mainLoop();
        cleanup();
    }

private:
    void initVulkan() {

    }

    void mainLoop() {

    }

    void cleanup() {

    }
};

int main() {
    HelloTriangleApplication app;

    try {
        app.run();
    } catch (const std::exception& e) {
        std::cerr << e.what() << std::endl;
        return EXIT_FAILURE;
    }

    return EXIT_SUCCESS;
}
```

基本框架结构如下：
* 资源，变量初始化
* 主循环渲染，直到窗口关闭，退出主循环
* 清理资源，退出游戏
* 整体运行结果放入异常捕捉器里，如果发生异常，抛出异常原因

### 资源管理
Vulkan 创建的对象在我们不需要的时候都需要显示的销毁，使用C++的`<memory>`里的方法集合可以实现自动回收资源。但通过显示的释放资源，一是符合Vulkan的教条，任何操作透明，防止错误。二是通过学习控制Vulkan对象的生命周期是学习Vulkan的很好方式。

Vulkan资源管理有两种类型：
* 直接创建对象，使用 `vkCreateXXX` 和 `vkDestroyXXX` 组
* 从别的对象分配，使用 `vkAllocateXXX` 和 `vkFreeXXX` 组

### 集成GLFW
首先是使用GLFW自己头文件代替Vulkan的头文件，因为GLFW自己通过宏可以智能包含Vulkan头文件。

```C+++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
```

在初始化Vulkan 前先创建窗口。

```C++
void run() {
    initWindow();
    initVulkan();
    mainLoop();
    cleanup();
}

private:
void initWindow() {
    glfwInit();                                                                 // 初始化 GLFW

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);                              // 不创建OpenGL环境 
    glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);                                // 不对重设窗口大小相应

    // -----------------------------
    // 创建窗口函数
    // arg1->窗口宽度
    // arg2->窗口高度
    // arg3->窗口标题
    // arg4->指定显示器
    // arg5->OpenGL相关
    // 返回参数为 GLFWwindow* 
    // -----------------------------
    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);      // 创建窗口
}
```

主循环如下：
```C++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }
}
```

一直处理事件，直到窗口关闭，或发生错误。

最后当窗口退出，需要清理资源。
```C++
void cleanup() {
    glfwDestroyWindow(window);

    glfwTerminate();
}
```

## Instance
### 创建 Instance
通过创建Instance来初始化Vulkan库。Instance是应用程序和Vulkan之间的桥梁。它的创建是将应用的信息告知驱动。

```C++
void createInstance() {
    VkApplicationInfo appInfo = {};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pApplicationName = "Hello Triangle";
    appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.pEngineName = "No Engine";
    appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.apiVersion = VK_API_VERSION_1_0;
    
    VkInstanceCreateInfo createInfo = {};
    createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
    createInfo.pApplicationInfo = &appInfo;
    
    // Vulkan 是平台无关的，所以需要窗口扩展来和Vulkan交互
    uint32_t glfwExtensionCount = 0;
    const char** glfwExtensions;
    glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

    createInfo.enabledExtensionCount = glfwExtensionCount;
    createInfo.ppEnabledExtensionNames = glfwExtensions;;

    // 后面介绍
    createInfo.enabledLayerCount = 0;

    if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS) {
        throw std::runtime_error("failed to create instance!");
    }
}
```

Vulkan创建对象的通用规则如下：

* 创建信息结构体的指针
* 对象分配器回调
* 创建对象的指针

创建函数调用后返回创建结果，类型为 `VkResult`，内容是 `VK_SUCESS` 或其它错误代号。

### 检查扩展支持
使用Vulkan函数即可获取设备支持的扩展列表。

```C++
// 获取支持数量
uint32_t extensionCount = 0;
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

// 分配对应数量内存
std::vector<VkExtensionProperties> extensions(extensionCount);

// 设置扩展属性内容
vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, extensions.data());
```

上面三个步骤也是Vulkan里查询设置属性的通用规则。

### 清理
创建了`VkInstance`，所以在程序退出前需要清理。清理函数的第二个参数是清理后的回调，暂时不需要。

```C++
void cleanup() {
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

## Validation layers
### 什么是验证层
Vulkan的极简设计，错误检查非常有限。所以这些函数调用使用的参数都需要用户自己负责。但Vulkan也提供了非常优雅的系统*validation layers*。验证层是默认嵌入到Vulkan函数调用里的组件用来额外的操作。验证层通用操作如下：

* 检查参数的误用
* 追踪对象的创建和销毁来查找资源泄露
* 通过追溯线程调用源头来检查线程安全
* 将每次调用和参数的日志输出的标准输出
* 为性能分析和重放调用追踪Vulkan的调用

以下是检测层大概实现方法示例代码：

```C++
VkResult vkCreateInstance(
    const VkInstanceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkInstance* instance) {

    if (pCreateInfo == nullptr || instance == nullptr) {
        log("Null pointer passed to required parameter!");
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    return real_vkCreateInstance(pCreateInfo, pAllocator, instance);
}
```

检测层可以在调试阶段打开，在发布阶段关闭。

Vulkan自身不带检测层，但是 LunarG Vulkan SDK提供检测层合集来检查通用错误。Vulkan有两种类型的检测层，**instance** 和**device specific**。**instance** 负责检查全局Vulkan函数的调用，**device specific** 检查指定GPU驱动层的调用。

### 使用检测层
和扩展层相似，检测层需要指定名字来开启。所有用的标准检测被打包到称作`VK_LAYER_KHRONOS_validation`的层包含在SKD中。

当然还是需要查询当前设备以及Vulkan支持的检测层，在判断是否包含我们要用到的检测层。如果检测支持，将我们要使用的检测层设置到`instance`。

### 消息回调
检测层默认是将日志输出到标准输出，我们可以提供自己的回调函数来处理日志。回调函数需要绑定到一个信使对象上，创建此对象需要扩展的支持。所以先将扩展`VK_EXT_debug_utils`加入到instance的扩展里。后续才可以创建对应的信使。

回调函数必须满足相应的要求。需要对应类型的参数，并且是静态函数。

```C++
static VKAPI_ATTR VkBool32 VKAPI_CALL debugCallback(
    VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
    VkDebugUtilsMessageTypeFlagsEXT messageType,
    const VkDebugUtilsMessengerCallbackDataEXT* pCallbackData,
    void* pUserData) {

    std::cerr << "validation layer: " << pCallbackData->pMessage << std::endl;

    return VK_FALSE;
}
```

首个参数为消息级别，说明如下：

* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT`：诊断消息
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT`：信息，比如创建资源
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT`：不是错误信息，但是可能是对Vulkan的错误使用
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT`：程序运行异常，可能导致程序崩溃

第二个参数`messageType` 说明如下：

* `VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT`：有事件发生，但与性能和标准无关
* `VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT`：可能违反规范或可能有错误发生的事件
* `VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT`：很可能不是高性能使用Vulkan

第三个参数`pCallbackData`是结构体`VkDebugUtilsMessengerCallbackDataEXT`包含消息自身，有如下重要成员：

* `pMessage`：以 null 结束的调试消息字符串
* `pObjects`：和消息相关的对象列表
* `objectCount`：对象数量

最后一个参数`pUserData`为设置回调函数时设置的参数，用来用户自己传递所需要的消息。

回调函数需要返回`VK_FALSE`，如果是`VK_TRUE` 将抛出`VK_ERROR_VALIDATION_FAILED_EXT`错误。

然后创建对象来传递回调函数。

```C++
VkDebugUtilsMessengerCreateInfoEXT createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
createInfo.pfnUserCallback = debugCallback;
createInfo.pUserData = nullptr; // Optional
```

创建对象时和回调函数参数一致，可以指定对应的消息级别，消息类型，以及回调函数。

因为这是扩展对象，所以创建对象的函数也是扩展函数，需要首先加载，方可使用。

```C++
VkResult CreateDebugUtilsMessengerEXT(VkInstance instance, const VkDebugUtilsMessengerCreateInfoEXT* pCreateInfo, const VkAllocationCallbacks* pAllocator, VkDebugUtilsMessengerEXT* pDebugMessenger) {
    auto func = (PFN_vkCreateDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkCreateDebugUtilsMessengerEXT");
    if (func != nullptr) {
        return func(instance, pCreateInfo, pAllocator, pDebugMessenger);
    } else {
        return VK_ERROR_EXTENSION_NOT_PRESENT;
    }
}
```

最后创建对象：

```C++
// 倒数第二个参数为分配器调用回调，所以可以传空
if (CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &debugMessenger) != VK_SUCCESS) {
    throw std::runtime_error("failed to set up debug messenger!");
}
```

最后需要清理对象：

```C++
void DestroyDebugUtilsMessengerEXT(VkInstance instance, VkDebugUtilsMessengerEXT debugMessenger, const VkAllocationCallbacks* pAllocator) {
    auto func = (PFN_vkDestroyDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkDestroyDebugUtilsMessengerEXT");
    if (func != nullptr) {
        func(instance, debugMessenger, pAllocator);
    }
}
```

### 调试 instance 的创建和销毁
在创建信使对象的时候需要instance作为参数，销毁信使对象在销毁instance之前，所以无法检验instance的创建和销毁。在Vulkan的扩展文档里得知通过在创建instance时，可以在将`VKInstanceCreateInfo`的`pNext`指向`VkDebugUtilsMessagerCreateInfoEXT`结构体。这样实现检验instance的创建和销毁。

### 配置
检验层还有许多指定标志，具体查看Vulkan SDK下的`Config`目录，`vk_layer_setting.txt`解释如何配置检验层。


# 安装环境
具体不同平台的安装vulkan的详细步骤这里没有记录的必要，这里只介绍使用到的三个库。

## SDK
Vulkan SDK 里包含头文件，标准检测层，调试工具以及Vulkan 函数的加载器。在运行时，加载器去驱动层查找函数，类似于GLFW对于Opengl。

## GLFW
Vulkan是一个和平台无关的API，因此不包含工具用来创建渲染结果窗口。为了享用Vulkan跨平台的优势，但同时省去各个平台(Windows, Linux, MacOs)复杂的创建窗口流程，选择GLFW非常合适。

## GLM
和DirectX12 不同，Vulkan自身不包含线性代数库，所以GLM是最好选择。


[返回](../README.md)
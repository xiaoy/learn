# Graphic pipeline basics
## 简介
图形渲染管线是将网格(mesh)上的顶点和贴图进行一些列的操作后最终表现到渲染目标的像素上。渲染流程图如下：

![graphic pipeline](../res/vulkan_simplified_pipeline.svg)

*input assembler* 从指定的缓存里收集顶点数据，有些为了节省空间，使用额外的顶点索引缓存来指定顶点数据。

*vertex shader* 每个顶点都要执行的程序，通常是将顶点从模型空间转换到屏幕空间。并且将转换后的顶点数据传到管线下一流程。

*tessellation shader* 用来通过一定的规则来细分几何体，从而来增加网格(mesh)质量。通常用在砖块，楼梯这些有突出来结构的物件上，当摄像机靠近时不显得太平坦。

*geometry shader* 运行在每个基础几何体上(三角形，线段，点)，此shader可以让输入的几何体降级为更简单的几何体，或升级为更复杂的几何体。这个*tessellation shader*有些相似，但更加灵活。但当前大多数显卡运行此shader效率不高，除了intel的集成显卡。

*rasterization* 阶段将几何体离散为片元(fragments)。它们就是充满framebuffer的像素。所有在framebuffer外的像素被丢弃，*vertex shader*输出的顶点属性插值到每个片元上。因为深度检测，在片元后方的其它片元被丢弃。

*fragment shader* 每个存活的片元都要经过此shader处理并且需要判断使用对应的颜色和深度值写入相应的framebuffer。在这个阶段可以使用*vertex shader*传递来的贴图坐标和法线变量，以及其它变量。

*color blending*阶段将映射到同一像素的不同片元混合到一起，片元之间可以简单的互相覆盖，或者叠加，或者根据透明度来混合。

颜色为绿色阶段(stages)称作*固定管线阶段(fixed-funciton stage)*。本阶段只能设置参数，运行流程是预定义的，不可改变。

颜色为橙色阶段(stages)称作*可编程阶段*，这些阶段可以上传自己的shader代码，来实现自己想完成的操作。这些程序并行运行在GPU的多核上。

像OpenGL，Directx3D旧的APIs，修改管线只能通过类似于`glBlendFunc`和`OMSetBlendState`这样的接口设置参数来实现。在Vulkan里的图像管线几乎全部都是可修改的，因此当你想修改shader，绑定不同的framebuffers或修改融合函数时，都需要重头新建管线。缺点是你需要根据不同的组合创建不同的管线来实现自定义的渲染操作。但是这种高度透明的自定义管线可以让驱动更好的优化。

一些可编程的阶段可以根据需要是否开启。比如如果只是渲染简单的几何体，可以选择关闭*tessellation shader*和*geometry shader*。再者比如只是想渲染*shadow map*，可以关闭*fragment shader*。

## Shader 模块
相比早期的APIs使用`GLSL`和`HLSL`语法来添加可读性很高的shader代码，Vulkan的shader代码转换为字节码格式。这种字节码的格式称作`SPIR-V`被设计用在Vulkan和OpenCL。这种格式用在图形和计算shader上。

优点就是GPU厂商的编译器将字节码格式的程序转换为运行时程序更加简单。过去使用类似`GLSL`写的shader代码，一些GPU厂商对标准的解释过于灵活，很有可能在不同GPU上编译结果不一致，使用字节码可以有效的避免类似问题。

但这并不是说要手写字节码，Khronos发布了和厂商无关的编译器，将`GLSL`编译为`SPIR-V`。编译器被设计为完全符合标准输出字节码后用在程序里。也可以将编译器嵌入到程序里，在运行时将`GLSL`编译为`SPIR-V`供程序使用。我们可以直接使用Khronos提供的`glslangValidator.exe`，但我们选择使用google提供的`glslc.exe`。优点是`glslc`使用的参数格式和著名的编译器GCC，CLang一致，并且还类似于`include`的功能。

GLSL是一种C语法风格的shading语言。程序里包含`main`函数被每个对象调用。除了使用函数参数当做输入，返回值当做输出，GLSL使用许多全局变量来处理输入输出。本语言包括很多特性来辅助图形编程，比如内置的基础`vector`和`matrix`，还包括函数操作，比如点积(cross-product)，矩阵和向量乘法(matrix-vector product)，围绕向量的反射(reflections around a vector)。向量类型使用`vec`以及后缀数字来标识元素数量。

### Vetex shader

## Fixed 函数

## 渲染通道

## 结论
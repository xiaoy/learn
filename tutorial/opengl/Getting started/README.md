# Geting started
## OpenGL
OpenGL是提供一系列函数接口来操作graphic和images的API(Application Programming Interface)。OpenGL其实是[Khronos Group](http://www.khronos.org/)开发维护的说明文档。每家显卡平台根据说明文档开发相应的库来实现这个接口。

### Core-profile vs Immediate Mode
Immediate mode也称作固定管线，大多数OpenGL的操作隐藏在库里，只能通过少数的函数接口来操作显卡。这满足不了很多开发需求，同时效率低下。从OpenGL 3.2开始引入了Core-profile，提供了更多接口，开放了更多的接口。OpenGL从此变的灵活且高效。

### Extensions
OpenGL有非常棒的特性是支持扩展，当显卡厂商提出新的技术或重大优化，通常这些特性包含在显卡驱动里。这时使用OpenGL的扩展功能来使用这些特性，而不需要等待OpenGL的更新。

```C++
if(GL_ARB_extension_name)
{
    // Do cool new and modern stuff supported by hardware
}
else
{
    // Extension not supported: do it the old way
}
```

### State mechine
OpenGL自身是一个大的状态机：一系列的变量定义了OpenGL当前的操作。OpenGl的状态通常称作OpenGL的**context**。最终的绘制渲染操作就是通过修改OpenGL的操作来实现。

### Objects
OpenGL的核心使用C语言实现。使用了多种抽象方式，其中一种就是**objects**。object是一系列选项的集合，来表示OpenGL状态子集。

OpenGL的状态Context看做一个大的结构体，对象子状态通过object来操作：

```C++
// The State of OpenGL
struct OpenGL_Context {
  	...
  	object_name* object_Window_Target;
  	...  	
};
```

以下为object的生命周期：

```C++
// create object
unsigned int objectId = 0;
glGenObject(1, &objectId);
// bind/assign object to context
glBindObject(GL_WINDOW_TARGET, objectId);
// set options of object currently bound to GL_WINDOW_TARGET
glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_WIDTH,  800);
glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_HEIGHT, 600);
// set context target back to default
glBindObject(GL_WINDOW_TARGET, 0);
```

## Creating Window
创建惊艳的图形第一步是创建OpengGL context以及一个窗口来绘制。每个操作系统的实现方式不一致。

### GLFW
GLFW使用C语言实现，专门用于OpenGL。它可以创建OpenGL context，定义窗口参数，处理用户输入。

### GLAD
OpenGL只是一个标准，每家的厂商实现不一致。因此这里有许多版本的OpenGL驱动，大多数函数只能运行时获取。所以在使用函数时，需要通过定义函数指针，然后获取函数，在调用函数。GLAD用来处理这个麻烦事。

```C++
// define the function's prototype
typedef void (*GL_GENBUFFERS) (GLsizei, GLuint*);
// find the function and assign it to a function pointer
GL_GENBUFFERS glGenBuffers  = (GL_GENBUFFERS)wglGetProcAddress("glGenBuffers");
// function can now be called as normal
unsigned int buffer;
glGenBuffers(1, &buffer);
```

使用GLAD即可简化上面的流程，只要调用函数即可。

### 创建流程

* 初始化GLFW
* 创建窗口
* 加载GLAD
* 设置viewport
* 设置窗口尺寸大小修改回调
* 处理事件回调
* 交换buffer
* 退出程序

## Hello Triangle
在OpenGL里所有物件都在3D空间里，但屏幕或窗口是一个像素的2D数组，因此OpenGL的大部分工作是将3D坐标系转换为屏幕上显示的2D像素。从3D坐标系转换为2D像素的流程通过OpenGL的图形管线来管理。图形管线分为两个部分，前一部分将3D坐标转换为2D坐标，后一部分将2D坐标系转换为着色后的像素。

渲染管线分为几个步骤，每个步骤需要前一个步骤的输入。这些步骤都是互相独立的，因此可以并行运行。在管线的每个步骤都会运行相应的程序来处理数据，这些程序称作shaders。

一部分shaders允许我们使用自己的shaders来替换。给与了我们合适的对管线控制的粒度。shader使用*OpenGL Shading Language(GLSL)*来编写。

下图为图形管线的每个阶段的抽象，蓝色阶段代表我们可以使用自己的shader替换：

![](../res/pipeline.png)


---
title: WebGL 编程指南
date: 2022-12-07 18:26
tags: [前端, css]
---

## 一、什么是 WebGL

[WebGL](https://www.khronos.org/webgl/) 是一个 JavaScript API，可以基于 [OpenGL ES](https://www.khronos.org/opengles/) 2.0 的 API 在 canvas 中进行 2D 和 3D 渲染。[THREE.js](https://threejs.org/)和[BABYLON.js](https://www.babylonjs.com/)等很多框架封装了 WebGL，提供了各个平台之间的兼容性。使用这些框架而非原生的 WebGL 可以更容易地开发 3D 应用和游戏。了解了 WebGL 后能更深刻的理解这些框架的实现。**虽然说我们开发中用不上这么底层的 API，但是了解了 WebGL 后能更深刻的理解这些框架的实现。**

<!--more-->

WebGL 程序包括用**JavaScript 写的控制代码**，和在图形处理单元（GPU）中执行的**着色代码（GLSL）**。

## 二、最简的 WebGL 程序

WebGL 在浏览器上是通过 Canvas 画布渲染所以我们**需要一个 Canvas**。

通过 getWebGLContext 获得**WebGL 的绘图上下文**。其实是我们自定义的方法，因为 WebGL 获取上下文的方式有些许繁琐所以我们直接引入一个工具库来帮我获取 WebGL 的上下文。

通过**clearColor 设置清空颜色**，其中颜色的设置是 rgba 但是值是从 0.0 到 1.0。

通过**clear 执行清空颜色缓冲区**。除了颜色缓冲区还能清空：

- gl.DEPTH_BUFFER_BIT 深度缓冲区
- gl.STENCIL_BUFFER_BIT 模板缓冲区

[code](https://codepen.io/WindStormrage/pen/qBoRGPV) 后续都可以在这个基础上改 js 里的代码

```js
<canvas id="mycanvas" height="500" width="500"></canvas>;
const canvas = document.getElementById("mycanvas");
// 获取上下文
const gl = getWebGLContext(canvas);
// 设置清空的颜色
gl.clearColor(0, 0, 0, 0.1);
// 清空
gl.clear(gl.COLOR_BUFFER_BIT);
```

这就是最简的的 WebGL 程序，可能你会说为什么没有着色器呀，别急我们下面来带你在 WebGL 中绘制一个点。

## 三、绘制一个点

### 3.1 使用着色器绘制一个点

想要使用 WebGL 作图就必须要使用到着色器（shader）。着色器包括：

- **顶点着色器**（Vertex shader）用来描述顶点的特征（位置、颜色、大小等），顶点指的就是空间中的某一个点。
- **片元着色器**（Fragment shader）处理每一个片元，片元可以理解成像素点。

比如说我们想画一个三角形首先要知道三角形的三个点在哪，这就需要我们通过顶点着色器来描述点的位置，然后如果要把这个三角形画出来就需要填充颜色，三角形上每个像素点需要什么颜色就可以使用片元着色器来决定。

我们先从画一个点开始[code](https://codepen.io/WindStormrage/pen/GRxNZLJ?editors=0010)：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9c5ccb9f6c84f52875874d08be64747~tplv-k3u1fbpfcp-zoom-1.image)

```js
// 顶点着色器
const VSHADER_SOURCE = `
  void main() {
   // 点的位置
   gl_Position = vec4(0.0, 0.0, 0.0, 1.0);
   // 点的大小
   gl_PointSize = 10.0;
 }
 `;
// 片元着色器
const FSHADER_SOURCE = `
  void main() {
   // 设置颜色
   gl_FragColor = vec4(1.0, 0.0, 0.0, 1.0);
 }
`;
// 初始化着色器
initShaders(gl, VSHADER_SOURCE, FSHADER_SOURCE);
// 绘制一个点
gl.drawArrays(gl.POINTS, 0, 1);
```

着色器用的是**glsl 来编写**，glsl 是以 c 语言为基础的强类型语言，在**js 中储存为字符串**然后通过辅助函数**initShaders**就可以进行读取。

其中顶点着色器**gl_Position**和**gl_PointSize**是**内置变量**分别表示顶点的位置和尺寸。因为是强类型语言赋值类型必须相同，gl_Position 类型是 vec4，gl_PointSize 类型是 float，vec4 是 glsl 特有的类型，由四个浮点数组成的矢量，相同意思的类型还有 vec2，vec3。位置使用了四个分量是因为顶点位置是齐次坐标，只要知道第四个分量是 1.0 就可以了，其他分别表示 x，y，z 三个分量。

片元着色器最后的颜色输出需要**把颜色值赋值给 gl_FragColor**，同样也是 vec4 类型，分别对应 RGBA 的四个分量。

创建着色器后需要使用**gl.drawArrays 方法进行绘制**，第一个参数是绘制方式，第二个参数是从哪个顶点开始绘制，第三个参数是绘制多少个顶点，具体意思在后续会说明。

WebGL 使用的是**右手坐标系**，最后可以在画布中心看到我们绘制的一个点。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d26ef903c2642dbbc9577a1af920cfb~tplv-k3u1fbpfcp-zoom-1.image)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5bee3c63f3d54270af8c0b7ace8b6a4b~tplv-k3u1fbpfcp-zoom-1.image)

你可以会认为片元着色器是不是像油漆桶工具一样直接把我们画的点绘制成红色，为了证明片元着色器是逐像素执行每一个点的颜色，我们可以看看下一节。

### 3.2 一个点里的 shader 世界

```glsl
const VSHADER_SOURCE = `
  void main() {
   gl_Position = vec4(0.0, 0.0, 0.0, 1.0);
   gl_PointSize = 90.0;
 }
 `;
const FSHADER_SOURCE = `
  // 精度限定词
  precision mediump float;
  void main() {
    // 当前片元坐标
    vec2 uv = gl_FragCoord.xy;
    float color = 0.0;
    if (uv.x > 50.0)
      color = 1.0;
   gl_FragColor = vec4(color, color, color, 1.0);
 }
`;
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54fb789223f941f1b83a68e21762292c~tplv-k3u1fbpfcp-zoom-1.image)

我们先看看上面这段 shader，顶点着色器只是把点的大小变大了，方便我们观察。

片元着色器首先我们定义了 float 类型的精度，因为 webgl 中只有 float 类型没有默认的精度所以使用 float 类型必须需要手动指定。

**gl_FragCoord**是片元着色器的内置变量，gl_FragCoord 的 x 和 y 元素是当前片段的窗口空间坐标。它们的起始处是窗口的左下角。通过[变量].xy 就可以获得变量的前两个分量。然后下面的代码意思是当前片元 x 大于 50 就会设置 color 为 1.0 也就是白色，否则就是 0.0 黑色。

```glsl
const FSHADER_SOURCE = `
  precision mediump float;
  void main() {
    vec2 uv = gl_FragCoord.xy / vec2(100.0);
    // step：第二个参数大于第一个参数返回1 否自返回0
    // length：返回参数到左下角的距离
    vec3 color = vec3(step(0.3, length(uv - 0.5)));
    gl_FragColor = vec4(color, 1.0);
 }
`;
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b1a237b0f5240b19b236ce26630b5cd~tplv-k3u1fbpfcp-zoom-1.image)

通过一些内置的计算方法和聪明的头脑就可以用 shader 玩出各种花样具体可以看看[shader 高级应用](https://www.shadertoy.com/)，我们这里主要介绍 webgl 的使用所以 shader 内容只是点到为止。

### 3.3 从 js 传入点的信息

回到正题，一般我们的数据都是通过 js 拿到的，所以**js 和 shader 是怎么实现交互**的呢。

在顶点着色器中有两种变量可以从 js 中获得数据：

**attribute 变量**：传输与顶点相关的数据，每个顶点获得的数据不一样。只能在顶点着色器使用。

**uniform 变量**：传输与顶点无关的数据，每个顶点获得相同的数据。

```js
const VSHADER_SOURCE = `
  // 声明变量
  attribute vec4 a_Position;
  attribute float a_PointSize;
  void main() {
   gl_Position = a_Position;
   gl_PointSize = a_PointSize;
 }

 `;
const FSHADER_SOURCE = `
  precision mediump float;
  // 声明变量
  uniform vec4 u_FragColor;
  void main() {
    gl_FragColor = u_FragColor;
 }
`;

// 获得变量的储存位置
const a_Position = gl.getAttribLocation(gl.program, "a_Position");
const a_PointSize = gl.getAttribLocation(gl.program, "a_PointSize");
const u_FragColor = gl.getUniformLocation(gl.program, "u_FragColor");

// 传值
gl.vertexAttrib2f(a_Position, 0.5, 0.5);
gl.vertexAttrib1f(a_PointSize, 10.0);
gl.uniform4f(u_FragColor, 1.0, 0.0, 0.0, 1.0);
```

通过**getAttribLocation**和**getUniformLocation**就可以获得对应**变量的储存地址**，获得地址后通过**vertexAttrib2f**、**vertexAttrib1f**、**uniform4f 向变量赋值。**

你可能注意到了方法后面的 1f、2f、4f，vertexAttrib[1234]f 分别为你想要传入的浮点数的数量，然后我们为什么只给 a_Position 传入了两个浮点数呢，如果定义的是 vec4 然后只传了两个浮点数的话 webgl 会自动补充没有传的位数，并不会报错。

## 四、绘制三角形

### 4.1 创建缓冲区

之前都是对一个顶点进行绘制，一般的图形，三角形，立方体或建模同学导出的模型数据都是由多个顶点构成的，如果想要**一次绘制多个点**需要怎么传递数据呢。我们可以通过**缓冲区对象**(BufferObject)一次性向着色器传入多个顶点数据。[code](https://codepen.io/WindStormrage/pen/WNzoxrE?editors=0010)

```js
// 绘制方式; 开始顶点; 总顶点数;
gl.drawArrays(gl.TRIANGLES, 0, 3);

function initVertexBuffers() {
 const vertices = new Float32Array([-0.5, 0.5, 0.5, 0.5, 0, -0.5]);
 // 创建缓冲区对象
 const vertexBuffer = gl.createBuffer();
 // 将缓冲区对象绑定到目标
 // gl.ARRAY_BUFFER  缓冲区对象中包含顶点数据
 gl.bindBuffer(gl.ARRAY_BUFFER, vertexBuffer);
 // 向缓冲区对象写数据
 // 第二个参数的数据写入绑定了第一个参数的缓冲区对象
 gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);
 const a_Position = gl.getAttribLocation(gl.program, "a_Position");
 // 将缓冲区对象分配给变量
 // 参数分别为, 位置;大小(每一个顶点的分量数,两个就是x和y);类型
 gl.vertexAttribPointer(a_Position, 2, gl.FLOAT, false, 0, 0);
 // 连接变量和缓冲区对象
 gl.enableVertexAttribArray(a_Position);
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db700648cbd240f4b5dbc4c1fe53dce0~tplv-k3u1fbpfcp-zoom-1.image)

使用缓冲区对象向顶点着色器传递数据**不需要调整着色器代码**，也是通过 attribute 变量接收数据。

在绘制的时候需要在**drawArrays 时传递一下顶点的数量**，这样着色器才能知道需要绘制多少个顶点，从缓冲区拿多少数据。

然后我们看看创建缓冲区的方法，首先需要定义数据，这里使用的是 Float32Array，是因为 WebGL 只能**从缓冲区读取二进制数据**，使用类型化数组可以操作二进制缓冲区的数据。数据有了后可以**开始连接着色器**了：

**1、创建缓冲区对象：**

可以通过 createBuffer 创建一个 WebGL 的缓冲区对象。

**2、绑定缓冲区对象**

使用 bindBuffer 把创建的缓冲区对象绑定给**gl.ARRAY_BUFFER**，gl.ARRAY_BUFFER 表示缓冲区中包含的是顶点数据，也可以绑定 gl.ELEMENT_ARRAY_BUFFER 表示数据是顶点索引。说句题外话：在 js 中 ArrayBuffer 不是某种东西的数组，而是一段二进制数据缓冲区，需要使用类型化数组操作，具体可[看看这](https://zh.javascript.info/arraybuffer-binary-arrays)。

**3、将数据写入缓冲区对象**

缓冲区处理好了后就可以向缓冲区写入数据了，通过[bufferData](https://developer.mozilla.org/zh-CN/docs/Web/API/WebGLRenderingContext/bufferData)，向绑定了 ARRAY_BUFFER 的缓冲区写入数据，gl.STATIC_DRAW 表示只会想缓冲区写入一次数据。

**4、将缓冲区对象分配给 attribute 变量**

先获得 attribute 变量的位置，然后通过**vertexAttribPointer**把整个缓冲区分配给变量。参数分别为，位置、每一个顶点数据大小，数据类型， 是否进行归一化处理，后面两个参数后续再说明。

**5、开启 attribute 变量**

使用**enableVertexAttribArray**开启。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f33f52bbd3948318e55fae30c786e7d~tplv-k3u1fbpfcp-zoom-1.image)

### 4.2 不同的绘制方式

对于 drawArrays 的第一个参数可以传递以下七种绘制方式：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd06e63c17a84dd3bdefec5e67f2b452~tplv-k3u1fbpfcp-zoom-1.image)

### 4.3 传递多组数据

上述例子只说了传递多个顶点信息，如果需要同时传递多个顶点大小或者顶点颜色信息，可以多建立几个缓冲区每个缓冲区传递给一个 attribute 变量，其实还有更加方便的方法，就是使用 vetrexAttribPointer 方法的后两个参数：

```js
function initVertexBuffers() {
 const vertices = new Float32Array([
  -0.5, 0.5, 1.0, 1.0, 0.0, 0.5, 0.5, 1.0, 0.0, 1.0, 0, -0.5, 0.0, 1.0, 1.0,
 ]);
 // 每个元素占字节数
 const FSIZE = vertices.BYTES_PER_ELEMENT;

 const vertexBuffer = gl.createBuffer();
 gl.bindBuffer(gl.ARRAY_BUFFER, vertexBuffer);
 gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);

 const a_Position = gl.getAttribLocation(gl.program, "a_Position");
 gl.vertexAttribPointer(a_Position, 2, gl.FLOAT, false, FSIZE * 5, 0);
 gl.enableVertexAttribArray(a_Position);

 const a_Color = gl.getAttribLocation(gl.program, "a_Color");
 gl.vertexAttribPointer(a_Color, 3, gl.FLOAT, false, FSIZE * 5, FSIZE * 2);
 gl.enableVertexAttribArray(a_Color);
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/22d1812aebe049e6bfdd280edd08e41f~tplv-k3u1fbpfcp-zoom-1.image)

和之前的代码区别是：

- 传入的数据变多了，每一行除了传两个坐标信息，还带有三个颜色信息。
- 增加了一个 a_Color 的 attribute 变量来接受颜色数据。
- vertexAttribPointer 调整了两个参数的传入，分别是**每两次顶点之间间隔的字节数**，**当前变量取值的偏移字节数**，因为是传字节数所以需要通过 BYTES_PER_ELEMENT 获取数组每一项占用字节大小。新数据是每间隔 5 个数据为一个顶点，位置从第 0 个开始取值，颜色从第 2 个开始取值。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c953e3eb2f314a329fa0b5aaac879712~tplv-k3u1fbpfcp-zoom-1.image)

其实上面传递的是颜色变量，但是到得到的三角形是没颜色的，**attribute 变量是不能传递到片元着色器的**，那能不能通过 uniform 变量向片元着色器传值呢，uniform 变量传的是所有顶点都一样的数据，下一章来讲讲怎么向片元着色器传递多种颜色。

## 五、绘制彩色三角形

### 5.1 着色器之间数据传递

介绍一种新变量，**varying变量**能够从顶点着色器向片元着色器传递数据。具体使用见代码：[code](https://codepen.io/WindStormrage/pen/zYWoBOJ?editors=0010)

``` glsl
const VSHADER_SOURCE = `
  attribute vec4 a_Position;
  attribute vec4 a_Color;
  varying vec4 v_Color;
  void main() {
   gl_Position = a_Position;
        v_Color = a_Color;
  }
 `;
const FSHADER_SOURCE = `
  precision mediump float;
  varying vec4 v_Color;
  void main() {
    gl_FragColor = v_Color;
  }
`;
```

在顶点着色器中**声明了attribute变量a_Color**接收缓冲区的数据。**声明varying变量v_Color**，把**a_Color的值传递给v_Color**。

片元着色器中只要声明了相同名字的varying变量就可以接收顶点着色器传递的数据了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ad12e7955fc49678aedce6105eaa1c0~tplv-k3u1fbpfcp-zoom-1.image)

上图就是数据传递的所有过程，通过传参的片元着色器得到的是一个五彩缤纷的三角形：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c8bcec9ec2ab40c1b79781027eab3235~tplv-k3u1fbpfcp-zoom-1.image)

### 5.2 图形装饰和光栅化

为什么会得到这样的结果呢，再来理解一下两个着色器，我是这么理解的，**顶点着色器会在处理每个顶点时执行一次**，**片元着色器是每个像素都会执行一次**，片元着色器不会属于一个顶点所以不能使用attribute变量。在顶点和片元之间还会存在下面两个步骤：

- **图形装配过程**：把孤立的点通过drawArrays的参数装配成几何图形，drawArrays的绘制次数的参数说明执行多少次片元着色器
- **光栅化过程**：把装配好的几何图形转换成片元。举个不恰当的例子，就是类似于把svg转换成png，svg是每个点线的描述文件，png是像素信息的文件。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/44a3a9c85f704f13aa14f6281481cf67~tplv-k3u1fbpfcp-zoom-1.image)

颜色会被写入颜色缓冲区最后渲染在浏览器上面。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/528487ca986d44e8904c45c0e9744002~tplv-k3u1fbpfcp-zoom-1.image)

### 5.3 varying变量的内插

我们只对每个顶点设置了颜色，每一个片元的颜色需要怎么确认呢，顶点着色器的v_Color传递到片元着色器的v_Color之间进行了**内插的过程**，如果是两个顶点，就会对两个点进行插值获取到中间的颜色，所以片元着色器会因为位置不同而渲染不同的颜色。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25136c9468f34bccb3762858314d4af8~tplv-k3u1fbpfcp-zoom-1.image)

从数据到最后渲染的全部流程就在下面：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a5cffe1bcf54c6095c13e074d72de4a~tplv-k3u1fbpfcp-zoom-1.image)

### 5.4 gpu来加速运行

大家可能认为一个模型这么多顶点，一个页面这么多片元需要处理，WebGL岂不是非常的慢？第一章我们轻描淡写了一句“着色器语言在GPU中运行”，**WebGL的渲染依赖底层GPU的渲染能力**。大家也许见过这个图，通过上面对WebGL全部流程的了解就知道为什么WebGL有这么多独立的计算任务。CPU单个核心的运算速度非常快但是对于大量任务也只能排队执行，**而GPU（图形处理器: Graphic Processor Unit)可以看成有非常多的小管道，每一个管道执行一个着色器，整体的执行速度就会非常的迅速了。**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29c2541b7a844416a872b4692a86fafd~tplv-k3u1fbpfcp-zoom-1.image)![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38ecda67f2664b4a8baa980df82d7445~tplv-k3u1fbpfcp-zoom-1.image)

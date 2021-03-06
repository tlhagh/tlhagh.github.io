---
layout: post
title:  "快速构建VR原型"
date:   2017-08-24
categories: VR
author: 张翔
location: ShangHai, China
cover: "https://meetsup.oss-cn-hangzhou.aliyuncs.com/blog-images/article6/vrlogo.jpg"
description: 我的翻译计划第三篇，这篇主要介绍通过three.js和2D软件快速构建VR原型。
---
---
与桌面端、移动端VR一样，WebVR的设计还处于一无所有的状态。每一个过程和设计原则都必须反复斟酌，包括我们是如何将灵感构建成原型。然而，通过简单的圆柱体和精确的测量，就能在我们最喜欢的2D设计应用程序和我们的VR设备的虚拟画布之间快速移动。


经过很多年的在Photoshop 和Keynote之间来回摸索，很高兴最终选定Illustrator作为我们的主要开发工具。我很喜欢3D应用，比如4D电影。但是对于我们来说，最头疼就是排版和界面布局等。所以当设计一个WebVR 导航界面的时候，我想要一个能让我从Illustrator中创建的模型快速迭代到WebVR测试场景中的工作流模式。

## 简单来说

1. 用2D设计应用创建布局，并将其导出为bitmap位图。
2. 用three.js创建一个圆柱体网格，它的圆周/高度比与位图的宽/高比相同。
3. 将bitmap位图作为纹理贴图(texture)应用到圆柱上并翻转圆柱面。
4. 在VR设备中预览。

我们用首选的2D设计应用开始，对我来说，就是Illustrator。先创建一个**360cm×90cm**的画布。然后当我们在Rift上面预览时，这个画布将围绕着我们，映射到以我们（准确来说是WebGL相机）为中心的圆柱体上。

![mockup1.png](https://meetsup.oss-cn-hangzhou.aliyuncs.com/blog-images/article6/mockup1.png)

<font color="gray">这个360厘米×90厘米Illustrator布局将被映射到一个360厘米的周长和90厘米高的WebGL圆柱体上。</font>

使用现实世界的单位是很重要的，因为比例感是虚拟现实的一部分，用户能感觉到的比例主要取决于你场景中元素的大小相对于用户眼睛之间的距离。这个距离通过实际测量（准确来说，米）去定义。在圆柱体中使用现实世界的单位确保我们不会遇到任何奇怪的惊喜，例如突然出现10层高的文本块（除非你想要）。

当我们创建布局时，同样重要的是知道圆柱体上的元素最终会出现在哪里。这就是为什么宽360厘米很方便：我们的图上的每厘米在3D圆柱的圆周上就正好相等于1°。我们的布局（180厘米/180度）的中心将直接显示在观看者的前面，左右边缘会展现在后面。

![mockup2.png](https://meetsup.oss-cn-hangzhou.aliyuncs.com/blog-images/article6/mockup2.png)

但是我们需要知道布局对用户的可见性是多少？毕竟，人的视野是有限的（https://xkcd.com/1080/）。 我们不想让用户转90°才能阅读到重要的状态指示。以下来自Extron的图表给了我们一些灵感。

![human-visual-field.jpg](https://meetsup.oss-cn-hangzhou.aliyuncs.com/blog-images/article6/human-visual-field.jpg)

<font color="gray">人类的视野水平约为180度，但是我们读取文本的能力仅限于中心10°，我们符号识别的能力扩展到中心60°。</font>

为了帮助我们跟踪用户看到的内容，在布局模板中设置指示传达这些值会很有帮助。最重要的是，我们视野的中心位置60°，30°和10°分别可以看到颜色，形状和文字。

![mockup3.png](https://meetsup.oss-cn-hangzhou.aliyuncs.com/blog-images/article6/mockup3.png)

我们还需要记住，目前的VR设备的视野是相当有限的。例如，DK2具有大约90°的有效水平视野。下面是我们可以在设备中看到的东西：

![visual-field-DK2.png](https://meetsup.oss-cn-hangzhou.aliyuncs.com/blog-images/article6/visual-field-DK2.png)

一旦我们准备好布局，我们可以将其作为bitmap位图导出。只要我们不改变相同布局的宽高比，我们可以根据需要延长导出时间。

## 在VR中预览布局
好消息是我们无需大量的JS代码来创建我们的WebGL VR场景。以下场景是建立在MozVR three.js WebVR模板上的，可以从`vr-web-examples repo`（https://github.com/MozVR/vr-web-examples） 上获取代码。它使用three.js和另外两个库来处理VR设备的检测，通信和渲染。
要预览我们的模型，我们只需将我们保存的bitmap位图复制到`/images`目录并重命名为`mockup.png`，覆盖现有文件。然后，我们将`index.html`加载到启用VR的浏览器中（例如，使用VR的Firefox），然后按F或双击进入VR模式。在我们的VR设备中，浏览器将会重新渲染，我们会看到一个围绕着我们的圆柱体。

## 代码
我们来看看`index.html`是如何工作的。大部分代码是VR支持的three.js场景的标准模板。为了添加布局，我们需要做以下事情：

1.创建一个圆周/高度比与bitmap的宽/高比匹配圆柱体。
2.创建一个能加载我们模型作为纹理的材料。
3.翻转圆柱体确保模型能正确显示（朝内）。
4.从我们的圆柱体和材料中创建网格，并将其添加到场景中来呈现。

首先我们需要给圆柱体设置变量

```javascript
/*设置圆柱体需要展示在模型上的关键数据，将这些测量数据与图像的大小相匹配是很重要的，否则圆柱体的表面区域将会出现挤压或拉伸，我们从圆柱的圆周开始，将其设置为匹配图像的宽度。请记住，VR场景的标准测量单位是米。例如，如果我们的模型画布宽36厘米，我们将圆周值设置为3.6厘米（360度/ 100）。
*/
var circumference = 3.6;
/*设置圆柱体的半径。我们可以从圆周推导半径。*/
var radius = circumference / 3.14 / 2;
/*设置圆柱的高度。与周长一样，我们将该值与我们的模型的高度相匹配，并转换为米（90厘米转换为0.9米）。*/
var height = 0.9;
```

然后，我们使用变量创建一个圆柱体实例，然后将它的表面翻过来：

```javascript
/*
创建我们的模型的圆柱体对象。构造函数有如下参数:
`CylinderGeometry(radiusTop, radiusBottom, height, radiusSegments, heightSegments, openEnded)`.我们设置`radiusSegments`为60使圆柱体平滑，保持顶部和底部为`openEnded`
*/
var geometry = new THREE.CylinderGeometry( radius, radius, height, 60, 1, true );
/*
反转X轴上尺度。这会翻转圆柱表面，使它面向内侧。我们预期的模型显示效果：面向内侧和朝着正确的方向。尝试删除此行来查看在不翻转的情况下会发生什么。
*/
geometry.applyMatrix( new THREE.Matrix4().makeScale( -1, 1, 1 ) );
```

然后，我们为网格创建一种材质：

```javascript
/*
创建我们将加载模型并应用于圆柱对象的材料。设置`transparent`为`true`，能让我们使用Alpha通道的模型。设置`side`为`THREE.DoubleSide`,这会使我们的材料会朝内和朝外（相对于圆柱体面的方向）。默认情况下，材料和three.js的网格面朝外，从相反的方向看不见。设置`THREE.DoubleSide`可确保圆柱体及其材料可见，无论我们正在查看哪个方向（内部或外部）。这个步骤并不是绝对必要的，因为我们实际上会在稍后的步骤中将对象的面向内翻转，但是要注意`side`属性以及如何定义它。然后我们将模型加载成一种纹理。
*/
var material = new THREE.MeshBasicMaterial({
     transparent: true,
     side: THREE.DoubleSide,
     map: THREE.ImageUtils.loadTexture( 'images/mockup.png' )
});
```

接下来，我们创建网格并将其添加到我们的场景中：

```javascript
/*
从几何(geometry)和材料(material)创建我们的圆柱体对象的网格。
*/
var mesh = new THREE.Mesh( geometry, material );
/*
将我们的圆柱对象添加到场景中。添加到three.js场景中的元素的默认位置为（0,0,0），这也是我们相机在场景中的默认位置。所以我们的相机在圆柱体中间。
*/
scene.add( mesh );
```

圆柱体现在应该在场景中渲染，我们位于圆柱的中心。

## 进一步试验
在布局加载之后，还有更多我们可以选择去做的。

## 1.改变圆柱体的半径
这个能改变模型与用户之间的距离。默认情况下，周长3.6米的圆柱体的半径为0.573米（22.5in）。这是大多数人查看电脑显示器的平均距离。使用VR 设备，你可以调整网格的比例来感知你的布局。确保X、Y和Z轴设置成相同的值，否则这个圆柱体就会被拉伸。

```javascript
/*
为了调整我们的模型和用户之间的距离，我们可以选择扩展我们的网格。如果我们将X，Y，Z设置为0.5，那么半径将会缩小一半，并且模型更加接近我们的眼睛。因为我们按比例缩放（在X，Y，Z上相等），所以模型不会更大，但VR耳机的立体声效果告诉我们它们更接近。使用此设置来找到您喜欢的值。
*/
mesh.scale.set( 0.5, 0.5, 0.5 );
```

实验时，还需要要考虑场景中其他对象出现在布局和用户之间的可能性。如果我设计一个0.5米半径平视显示（HUD）风格的导航界面，VR中的头像就会镶嵌在墙上，墙面上的几何体会比界面离我们更近，这样就会挡住界面。我们的加载指示就会突然消失。

Oculus最佳实现指南（需要阅读所有VR内容的创建者）有如下建议：

> 使UI更接近用户（例如，20cm）可以帮助防止阻塞（VR中的对象比HUD对象更接近用户）。但这要求使用者“...只要在检查HUD时，就得将焦点转移到近距离HUD和远距离的场景之间，眼睛收敛和调节（眼睛焦点）的这些变化会很快导致疲劳和眼睛疲劳。”
>

## 2.添加背景图片
默认情况下场景的背景是黑色的，但是添加背景图片很容易。

```javascript
/*
要选择将背景图像添加到场景中，就需要创建一个大球体并应用到bitmap位图。首先，创建球的几何体。“SphereGeometry”构造函数需要几个参数，但是我们只需要基本的三个参数：“radius”，“widthSegments”和“heightSegments”。我们将“半径”设置为大到5000米，因此球体不太可能堵塞我们场景中的其他对象。我们将宽度和高度分别设置为64和32，使得球体表面光滑。然后，我们使用`THREE.Matrix4().makeScale()`在X轴上翻转几何面，使其朝内，就像我们的模型圆柱体一样。
*/
var geometry = new THREE.SphereGeometry( 5000, 64, 32 );
geometry.applyMatrix( new THREE.Matrix4().makeScale( -1, 1, 1 ) );
/*
创建加载背景图的介质
*/
var metarial = new THREE.MeshBasicMaterial({
     map:THREE.ImageUtils.loadTexture('images/background.ong')
});
/*
用geometry和material创建我们背景的网格，并将其添加到场景中。
*/
var mesh = new THREE.Mesh(geometry,meterial);
scene.add(mesh);
```

就是这样！当我们戴上VR设备并加载场景时，我们应该站在我们的模型布局中间，被很远的背景图像包裹在一起。使用不同的背景图来找到你想要的对比度。我倾向于使用近似3D场景的图像，以便我们能判断颜色，易读性。为了获得最佳效果，建议使用等角格式的全景图（见下文）。可以完美的映射到（不变形）WebGL球体上。

![puydesancy.jpg](https://meetsup.oss-cn-hangzhou.aliyuncs.com/blog-images/article6/puydesancy.jpg)

<font color="gray">由Alexandre Duret-Lutz拍摄的一个等角图像的例子。在Flickr上查找更多Alexandre的全景相片。</font>

Flickr的Equirectangular Pool是3D图片的绝佳来源（只需要检查许可证）。你还可以使用3D应用程序将3D场景渲染为等角格式。例如，我使用Cinema 4D Vray来创建本教程中使用的模糊的全景图。或者你只需要一个简单的渐变色或纯色，可以使用你喜欢的图像编辑应用来填充宽高比2：1的画布。

## 3.创建多个图层
深度是虚拟现实设计的基本要素。通过VR设备两个独立的眼睛，我们可以感知元素之间在z轴上的细微差异。例如，我们可以看到一个发光的按钮悬停在其父对话表面上方0.5cm处，或者UI水平拉伸。VR中的深度自然会在2D布局中进行缩影：在层叠之间创建视觉对比度。

很容易在我们的场景中添加额外的图层。我们创建额外的网格，并在其材料中加载不同的bitmap位图。

* 复制粘贴上面的代码，不包括周长`circumference`，半径`radius`和高度`height`的变量（它们只需要指定一次）。
* 在新材料中，指定不同的bitmap位图（例如，`THREE.ImageUtils.loadTexture（'images / mockup-background.png'）`）。
* 根据需要调整网格比例改变布局之间的距离。这会在各层之间产生分隔。

我创建MozVR.com布局的策略是首先在单个Illustrator图层中构建布局，而不必考虑太多关于3D构图的问题，然后再将元素分到新的层，使得每一层代表不同的深度。然后将每个层单独保存为透明的bitmap位图。然后将每一层都加载到自己的圆柱体的网格中，然后通过调整比例找到所需要的分隔。当我们和DK2的3D相机结合使用时，我发现这个深度特别壮观，当我们身处虚拟世界，很容易感知层之间的视觉差异效果。

## 玩得开心！
这种技术使我们能够将我们所熟悉的工作流与虚拟现实世界相结合。这是一个简单的快速迭代方式。Start hacking - and have fun!
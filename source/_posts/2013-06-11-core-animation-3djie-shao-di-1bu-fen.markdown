---
layout: post
title: "Core Animation 3D介绍 第1部分"
date: 2013-06-11 13:21
comments: true
published: true
categories: 动画
tag: [动画,CoreAnimation,翻译]
---

本教程将向你介绍Core Animation中用于绘制3D图形的一些技术。

好消息是：我们不必直接使用OpenGL，仅用Core Animation就可以很容易实现一些3D效果。但是，“用Core Animation来制作一个复杂的3D游戏”也并不是一个好主意。

这个教程将分为两部分。第一部分，我们先简单介绍一点3D原理知识，并运用这些概念来创建一些简单的3D场景。在第二部分中，我们将使用Core Animation来制作一个类似于旋转木马的3D场景特效。

最终app的预览效果如下：(译注: 原文中这里的视频被墙，因此这里只简单的提供一个图片预览，你可以直接下载例子代码运行即可以看到最终app的效果)

![](/images/posts/2013_06_11_core_animation_3d_app_preview.png)

准备好了吗？开始编码吧！
<!--more-->

首先，下载文章后面的代码。如果你要自己创建项目，要记得添加QuartzCore框架。

### 少量3D和矩阵相关的数学知识

要在一个3D空间中绘图，除了标准2D坐标(X和Y)概念外，我们还需要引入一个深度(depth)的概念，也就是要加入第三个坐标轴--Z轴。

这样，在空间中，我们只需要简单的改变物体的X、Y和Z坐标，就可以在垂直方向、水平方向或深度方向上移动物体。

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/handed.png)

在2D或3D空间中，对一个物体执行平移、缩放或旋转这些操作时需要使用矩阵运算。

你可以将矩阵想象成一个多维数组。比如，在3D空间中我们使用一个像下面这种格式的4X4矩阵：

``` objc
[X][0][0][0]
[0][Y][0][0]
[0][0][Z][0]
[0][0][0][1]
```

把这个矩阵和物体的每一个坐标点(又称顶点)相乘，我们可以得到物体的一个变换(transformation)效果。

严格点讲，前面这个矩阵是用来执行缩放操作的。其中的X、Y、Z值代表每个轴上的缩放值。

如果你要进行其他的变换操作，比如旋转或者平移，你需要将矩阵换成其他矩阵格式(scheme)。

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/Screen-Shot-2013-03-20-at-11.02.03-AM.png)

不要紧张！你不需要知道其他更多原理知识，而且也不必直接进行这些操作。Core Animation会为你完成这些操作，虽然你不知道它是如何做到的，但是它确实会为你完成这些操作，所以不必害怕。

当然，就我个人来说，如果我知道它背后是如何工作的(至少知道一些它的基本原理)，我会对我的代码更有自信。因此，如果你想了解更多矩阵相关的知识，建议你读一读这篇[文章](http://www.matrix44.net/cms/notes/opengl-3d-graphics/basic-3d-math-matrices)。


### 3D变换(Transformations)

现在基本上，你已经知道矩阵的作用了，也知道了3D空间是如何构成的。来，我们用Core Animation做一些3D的东西吧。

打开TB_3DIntro->viewController.m文件。

我在里面列出了6个分别以A、B、C、D、E、F开头的函数。每一个函数对应了一种不同的3D场景效果。

我们先来看看由`A_singlePlan`函数创造的场景吧。

用这个函数，我们画了一个平面，平面绕Y轴旋转了45度。

首先，我们创建了一个CALayer，用它来作容器层(当然，并不是一定要这样做，只是我更喜欢不直接在self.view的layer上进行工作)。

``` objc 
- (void)A_singlePlane{
    //Create the container
    CALayer *container = [CALayer layer];
    container.frame = CGRectMake(0, 0, 640, 300);
    [self.view.layer addSublayer:container];
```

然后我们创建了另一个CALayer，用它代表一个平面。

``` objc
	//Create a Plane
    CALayer *purplePlane = [self addPlaneToLayer:container
                                            size:CGSizeMake(100, 100)
                                        position:CGPointMake(250, 150)
                                           color:[UIColor purpleColor]];
```

我写了一个简单的辅助函数，用它将平面直接添加到容器层上，然后返回这个新建的平面层。代码非常简单：

``` objc
- (CALayer*)addPlaneToLayer:(CALayer*)container size:(CGSize)size position:(CGPoint)point color:(UIColor*)color{
    //Initialize the layer
    CALayer *plane = [CALayer layer];
    //Define position,size and colors
    plane.backgroundColor = [color CGColor];
    plane.opacity = 0.6;
    plane.frame = CGRectMake(0, 0, size.width, size.height);
    plane.position = point;
    plane.anchorPoint = CGPointMake(0.5, 0.5);
    plane.borderColor = [[UIColor colorWithWhite:1.0 alpha:0.5]CGColor];
    plane.borderWidth = 3;
    plane.cornerRadius = 10;
    //Add the layer to the container layer
    [container addSublayer:plane];
    
    return plane;
}
```

最后，我们用CATransform3D来添加变换。

啊？？？你一定又想问CATransform3D是什么东西？按住Cmd键点击这个类型名称，你可以发现它是一个结构体，使用了一种很“火星”的语法来表示一个矩阵。![](http://www.thinkandbuild.it/wp-includes/images/smilies/icon_razz.gif)

``` objc
struct CATransform3D
{
  CGFloat m11, m12, m13, m14;
  CGFloat m21, m22, m23, m24;
  CGFloat m31, m32, m33, m34;
  CGFloat m41, m42, m43, m44;
};

typedef struct CATransform3D CATransform3D;
```

旋转变换部分的代码也相当简单：

``` objc
    //Apply transform to the PLANE
    CATransform3D t = CATransform3DIdentity;
    t = CATransform3DRotate(t, 45.00f * M_PI / 180.0f, 0, 1, 0);
    purplePlane.transform = t;
}
```

先用单位矩阵`CATransform3DIdentity`来初始化一个变换(transformation)，然后用`CATransform3DRotate`函数给变换乘上一个旋转矩阵。

CATransform3DRotate这个函数的参数分别表示矩阵，旋转的角度(以弧度为单位)和3个坐标轴上的变换系数。这个例子中，X和Z轴都没受到影响，只对Y轴有影响，物体会在Y轴上旋转45度。

下图是运行结果。

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/pa1.png)

呃......你可以看到，它还不是3D的！我们只是在X轴方向上将一个正方形压扁了。

这是由于我们还没有设置视点的值(perspective value)。通常，在绘制3D场景的时候会将场景进行正射投影(Orthographic projection)处理，由此会产生一个扁平的场景。换句话说，在正射投影后你无法看到Z轴上的深度感。

要给我们的场景加上空间深度感，我们必须修改变换矩阵的`m34`参数。这个参数决定了视点的值。

现在，来看看`B_singlePlanePerspective`这个函数，这函数展示了视点的作用。

这个函数和前一个函数只有变换部分的代码有所不同：

```objc
    //Apply transform to the PLANE
    CATransform3D t = CATransform3DIdentity;
    //Add perspective!!!
    t.m34 = 1.0/ -500;
    t = CATransform3DRotate(t, degToRad(45.0), 0, 1, 0);
    purplePlane.transform = t;
}
```

你可以看到，我们直接给矩阵的m34属性赋了一个值。在这里，我不会深入的从数学上解释这个值是如何起作用的。但是我可以告诉你的是，这个值越接近0，视点就越深。

下面是两种不同视点值的效果：

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/pa_2.png)

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/pa_21.png)


### 3D变换的顺序问题(Transformations chain)

我们可以将多个矩阵相乘从而将多种变换应用到一个物体上。比如，如果我们想将平移和旋转变换应用到一个物体上，我们可以直接将两个变换矩阵相乘：

```objc 
TransformMatrix = TranslateMtx * RotateMtx
```

数学上，一般情况下乘法都可以使用交换律：

```objc 
TranslateMtx * RotateMxt = RotateMtx * TranslateMtx 
```

但是**矩阵乘法不适用于交换律**。AxB的结果可能和BxA的结果不一样。要记住这一点！

接下来`C_transformationsChain`这个例子中，我们会将2个变换效果按不同的先后顺序应用到2个不同的物体上。

以下是主要代码：

```objc
//Apply transformation to the PLANES
CATransform3D t = CATransform3DIdentity;
 
//Purple plane: Perform a rotation and then a translation
t = CATransform3DRotate(t, 45.0f * M_PI / 180.0f, 0, 0, 1);
t = CATransform3DTranslate(t, 100, 0, 0);
purplePlane.transform = t;
 
 
//reset the transform matrix
t = CATransform3DIdentity;
 
//Red plane: Perform translation first and then the rotation
t = CATransform3DTranslate(t, 100, 0, 0);
t = CATransform3DRotate(t, 45.0f * M_PI / 180.0f, 0, 0, 1);
redPlane.transform = t;
```

运行结果如下：

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/Screen-Shot-2013-03-20-at-11.47.15-AM.png)

看到了吗，不同的变换顺序完全导致了不同的效果。

我们重点下看一下紫色的那块。旋转变换改变了它的坐标轴方向，然后我们又将它沿X轴进行了平移，此时它的X轴方向已经和红色平面的X轴方向不一致了。

变换步骤示意图如下：

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/p_r.png)

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/p_r2.png)

### 图层层次(layer hierarchies)

到目前为止，我们都是将变换直接应用在这些平面上。在3D场景中，经常需要创建一些相互之间有层次结构的物体。这个时候，只需要将变换应用到根层次上，就可以使整个层次结构中的物体整体具有这个变换效果。这种做法非常有用。

接下来，我们来看看`D_multiplePlanes`这个例子。

我们在容器层上添加了4个平面。

当没有任何变换效果时看起来是这样的：

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/p5.png)

如果我们给每一个平面都加上一个Y轴上的旋转变换，我们会得到4个独立的旋转效果：

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/p5_2.png)

但是，如果我们只是在容器层上应用旋转变换，我们会得到一个完全不同的场景效果：

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/p5_1.png)

这种效果，是相机(camera)位置发生改变带来的结果。我们没有移动每个平面，只是改变了视点的位置。

此函数主要的变换部分代码如下，包含了分别应用于各个平面和应用于容器层的两种变换效果的代码：

```objc
//Transformation 
CATransform3D t = CATransform3DIdentity;
 
BOOL applyToContainer = NO;
 
//Apply the transformation to each PLANE
if(!applyToContainer){
    t.m34 = 1.0 / -500.0;
    t = CATransform3DRotate(t, degToRad(60.0), 0, 1, 0);
    purplePlane.transform = t;
     
    t = CATransform3DIdentity;
    t.m34 = 1.0 / -500.0;
    t = CATransform3DRotate(t, degToRad(60.0), 0, 1, 0);
    redPlane.transform = t;
     
    t = CATransform3DIdentity;
    t.m34 = 1.0 / -500.0;
    t = CATransform3DRotate(t, degToRad(60.0), 0, 1, 0);
    orangePlane.transform = t;
     
    t = CATransform3DIdentity;
    t.m34 = 1.0 / -500.0;
    t = CATransform3DRotate(t, degToRad(60.0), 0, 1, 0);
    yellowPlane.transform = t;
}
 
//Apply the transformation to the CONTAINER
else{    
    CATransform3D t = CATransform3DIdentity;
    t.m34 = 1.0/-500;
    t = CATransform3DRotate(t, degToRad(60.0), 0, 1, 0);
    container.transform = t;
}
```

### CATransformLayer

到目前为止我们所见到的代码都能正常工作，但说实话，作为3D层次结构的根，CALayer不是正确的选择。

函数`E_multiplePlanesZAxis`展示了为什么。

这个例子中，我们创建4个XY坐标相同只有Z坐标不同的平面。紫色平面最近，黄色平面最远。

```objc
//Apply transforms to the PLANES
t = CATransform3DIdentity;
t = CATransform3DTranslate(t, 0, 0, -10);
purplePlane.transform = t;
 
t = CATransform3DIdentity;
t = CATransform3DTranslate(t, 0, 0, -50);
redPlane.transform = t;
 
t = CATransform3DIdentity;
t = CATransform3DTranslate(t, 0, 0, -90);
orangePlane.transform = t;
 
t = CATransform3DIdentity;
t = CATransform3DTranslate(t, 0, 0, -130);
yellowPlane.transform = t;
```

在旋转这些平面前，我们先让容器层执行了一个旋转变换。

```objc
//Apply transform to the CONTAINER
CATransform3D t = CATransform3DIdentity;
t.m34 = 1.0/-500;
t = CATransform3DRotate(t, 80.0f * M_PI / 180.0f, 0, 1, 0);
container.transform = t;
```

你也许希望看到下面这种结果：

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/p6_1.png)

但是，实际上会得到这种结果：

![](http://www.thinkandbuild.it/wp-content/uploads/2013/03/p6.png)

这是因为CALayer不能处理3D层次结构的深度，它只能将场景处理成相同的Z层次。

为了修正这个问题，我们需要用一个`CATransformLayers`来做根层对象。

函数`F_multiplePlanesZAxis`修正了这个问题：

```objc
//Create the container as a CATransformLayer
CATransformLayer *container = [CATransformLayer layer];
container.frame = CGRectMake(0, 0, 640, 300);
[self.view.layer addSublayer:container];
```

CATransformLayer是一个特殊的layer。与CALayer的不同之处在于，CATransformLayer本身不会被渲染到屏幕上，只有它的子图层才会被渲染到屏幕上。所以它的一些属性，比如backgroundColor、contents、border等等都没有什么用。要记住这点。

到这里，这个教程的第一部分就完了。建议你实际运行一下这些函数，也可以试试我没有讲到的`CATransform3DScale`，试试用它做一个缩放变换看。

如果你有任何问题，可以在twitter上找到我([@bitwaker](http://www.twitter.com/bitwaker))。

[本教程代码下载](https://github.com/ariok/TB_3DCoreAnimation)

译自：[Think & Build](http://www.thinkandbuild.it/introduction-to-3d-drawing-in-core-animation-part-1/)
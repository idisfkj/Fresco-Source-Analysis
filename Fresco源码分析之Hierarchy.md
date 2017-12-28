# Fresco源码分析之Hierarchy
---
上篇文章我们分析了**Fresco**中的**DraweeView**，对其中的一些原理以及方法进行了解析。在这过程中我们了解到，**DraweeView**中是通过**DraweeHolder**来统一管理的。而**DraweeHolder**又是用来统一管理相关的**Hierarchy**与**Controller**，如果想了解**DraweeView**相关的知识，可以先看下我的前一篇文章[Fresco源码分析之DraweeView](https://github.com/idisfkj/Fresco-Source-Analysis/blob/master/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8BDraweeView.md)。今天这里进一步来分析**Fresco**中的**Hierarchy**。

# GenericDraweeHierarchyBuilder
在**GenericDraweeView**的构造方法中会调用*inflateHierarchy(context, atts)*方法来创建一个**GenericDraweeHierarchyBuilder**对象，通过调用该对象的**build**方法来生成一个**Hierarchy**。

>如果你看了我上一篇文章，相信对这个方法你会感到很亲切。

所以这个类的主要作用是用来创建一个**Hierarchy**，通过**builder**模式来实例化包含相关信息的**Hierarchy**。如果你一步步深入了解**Fresco**的话，相信你对**builder**模式也将习以为常，因为后面你将经常与它碰面。这也是**Fresco**开源项目的主要设计模式。在该**builder**类中主要构建的信息有:

* **Drawable**相关：**placeholderImage**、**retryImage**、**failureImage**、**progressBarImage**、**background**、**overlays**与**pressedStateOverlay**
* **ScaleType**相关：与**Drawable**相对应的**placeholderImageScaleType**、**retryImageScaleType**等等。
* 其它：**fadeDuration**渐变过渡时间与**RoundingParams**圆角相关信息等等

# GenericDraweeHierarchy
通过上面的**GenericDraweeHierarchyBuilder**的**build**方法会创建一个**GenericDraweeHierarchy**对象。这就是我们今天需要主要分析的类，也是**Fresco**的一个核心类。

## SettableDraweeHierarchy
首先我们来分析一下它的类结构，它会实现**SettableDraweeHierarchy**接口，我们进入该接口发现一共有6个接口方法，它们分别为：

1. *void reset();* 重新初始化**Hierarchy**
2. *void setImage(Drawable drawable, float progress, boolean immediate);* 设置实际需要展示的图片，其中**progress**表示图片的加载质量进度(在渐进式中会使用到)
3. *void setProgress(float progress, boolean immediate);* 更新图片加载进度
4. *void setFailure(Throwable throwable);* 图片加载失败时调用，可以设置**failureImage**
5. *void setRetry(Throwable throwable);* 当图片加载失败时重新进行加载，可以设置**retryImage**
6. *void setControllerOverlay(Drawable drawable);* 用来设置图层覆盖

这些方法在**GenericDraweeHierarchy**中都会做出相应的实现，同时最终都会在对应的**DraweeView**中的**Controller**来调用。

>至于这些方法中的实现细节，由于代码比较多，这里就不一一列举出来，大家可以自行查看**GenericDraweeHierarchy**的源码。

## DraweeHierarchy
上面的**SettableDraweeHierarchy**还有一个父类接口为**DraweeHierarchy**，在这个接口中只有一个接口方法为*Drawable getTopLevelDrawable();*。是不是对这个方法也有点熟悉呢？（路人甲：嗯，好像上篇文章提及过！）它是用来获取视图树的最顶层视图，其实说白了就是显示出来的**Drawable**。它主要在**DraweeHolder**中调用，最终也会由*void setHierarchy(DH hierarchy)* 与 *void setController(@Nullable DraweeController draweeController)* 来调用

```
super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());
```
从而显示需要展示的图片。

下面我们回到之前的**GenericDraweeHierarchy**类。在开始分析它之前，我们先来了解一下它的另一个感念-`层级树或者说图层树`，我这里就把它说成图层树吧。那么我们来看下**Fresco**的图层树是什么样的：

```
 *  o RootDrawable (top level drawable)
 *  |
 *  +--o FadeDrawable
 *     |
 *     +--o ScaleTypeDrawable (placeholder branch, optional)
 *     |  |
 *     |  +--o Drawable (placeholder image)
 *     |
 *     +--o ScaleTypeDrawable (actual image branch)
 *     |  |
 *     |  +--o ForwardingDrawable (actual image wrapper)
 *     |     |
 *     |     +--o Drawable (actual image)
 *     |
 *     +--o null (progress bar branch, optional)
 *     |
 *     +--o Drawable (retry image branch, optional)
 *     |
 *     +--o ScaleTypeDrawable (failure image branch, optional)
 *        |
 *        +--o Drawable (failure image
```
根据上面所展示的层次结构，我们可以发现最底层是由**RootDrawable**来构成，它有一个直接子分支为**FadeDrawable**。而在**FadeDrawable**中又有5个直接子分支，分别为**placeholder branch**、**actual image branch**、**progressBar image branch**、**retry image branch**与**failure image branch**。至于这些**image**的作用相信不用我再多做说明了，这些**image**除了**actual image**是必须要明确指定的，其它的都是可选择的配置。因此**RootDrawable**与**FadeDrawable**是一定存在的。虽然其它的都是可选的配置，但无论你是否选择了，它们的层级结构都会保留在图层树中。还有每一个层级都有自己独立的**scale type**，当然**rounding**(圆角)也是支持的。

>其实除了这5个分支，与它们同一层次的还有**background image**与**overlay image**，它们分别位于**placeholder image**之前与**failure image**之后。**background**相信都知道，因为图片都支持**background**与**src**，**overlay**为图层覆盖。至于为什么没有在上面的结构中显示，我这里也不得而知，猜测可能是这两个并不是**Fresco**主要常用的特性。

那么这些图层结构是通过**layers**数组来体现的，可以来看下**GenericDraweeHierarchy**的源码

```
  GenericDraweeHierarchy(GenericDraweeHierarchyBuilder builder) {
    mResources = builder.getResources();
    mRoundingParams = builder.getRoundingParams();
 
    mActualImageWrapper = new ForwardingDrawable(mEmptyActualImageDrawable);
 
    int numOverlays = (builder.getOverlays() != null) ? builder.getOverlays().size() : 1;
    numOverlays += (builder.getPressedStateOverlay() != null) ? 1 : 0;
 
    // layer indices and count
    int numLayers = OVERLAY_IMAGES_INDEX + numOverlays;
 
    // array of layers
    Drawable[] layers = new Drawable[numLayers];
    layers[BACKGROUND_IMAGE_INDEX] = buildBranch(builder.getBackground(), null);
    layers[PLACEHOLDER_IMAGE_INDEX] = buildBranch(
        builder.getPlaceholderImage(),
        builder.getPlaceholderImageScaleType());
    layers[ACTUAL_IMAGE_INDEX] = buildActualImageBranch(
        mActualImageWrapper,
        builder.getActualImageScaleType(),
        builder.getActualImageFocusPoint(),
        builder.getActualImageColorFilter());
    layers[PROGRESS_BAR_IMAGE_INDEX] = buildBranch(
        builder.getProgressBarImage(),
        builder.getProgressBarImageScaleType());
    layers[RETRY_IMAGE_INDEX] = buildBranch(
        builder.getRetryImage(),
        builder.getRetryImageScaleType());
    layers[FAILURE_IMAGE_INDEX] = buildBranch(
        builder.getFailureImage(),
        builder.getFailureImageScaleType());
    if (numOverlays > 0) {
      int index = 0;
      if (builder.getOverlays() != null) {
        for (Drawable overlay : builder.getOverlays()) {
          layers[OVERLAY_IMAGES_INDEX + index++] = buildBranch(overlay, null);
        }
      } else {
        index = 1; // reserve space for one overlay
      }
      if (builder.getPressedStateOverlay() != null) {
        layers[OVERLAY_IMAGES_INDEX + index] = buildBranch(builder.getPressedStateOverlay(), null);
      }
    }
 
    // fade drawable composed of layers
    mFadeDrawable = new FadeDrawable(layers);
     mFadeDrawable.setTransitionDuration(builder.getFadeDuration());
 
    // rounded corners drawable (optional)
    Drawable maybeRoundedDrawable =
        WrappingUtils.maybeWrapWithRoundedOverlayColor(mFadeDrawable, mRoundingParams);
 
    // top-level drawable
    mTopLevelDrawable = new RootDrawable(maybeRoundedDrawable);
    mTopLevelDrawable.mutate();
 
    resetFade();
  }
```
根据源码可以明显的看出不管**overlay**是否存在，它都会保留图层层次，当然如果存在的话，就根据实际情况在**OVERLAY_IMAGES_INDEX**之后增加图层层次。**layers**是一个**Drawable**数组，这里通过*Drawable buildBranch(@Nullable Drawable drawable, @Nullable ScaleType scaleType)* 方法来创建对应的**Drawable**。在最后，创建了**FadeDrawable**并将**layers**传递给它，在这里**FadeDrawable**的特性应该有点眉目了吧，其实它内部做的就是对**Drawable**数组进行操作。之后**RootDrawable**也出现了(关于**Drawable**文章后面会统一分析)，这样之前所提到的层级结构就形成了。

既然**actual image**是一定存在的，那么在它真正展示之前**image**中的显示用什么来控制的呢？其实就是我们之前所提到的**SettableDraweeHierarchy**来控制。在真实的图片展示出来之前，它可以用来展示**placeholder image**等相关图层。具体的我们可以来看一下它里面实现的一些方法。

## setImage
这里就拿*void setImage(Drawable drawable, float progress, boolean immediate)* 来进行深入分析，那么下面来看下它的源码：

```
  @Override
  public void setImage(Drawable drawable, float progress, boolean immediate) {
    drawable = WrappingUtils.maybeApplyLeafRounding(drawable, mRoundingParams, mResources);
    drawable.mutate();
    mActualImageWrapper.setDrawable(drawable);
    mFadeDrawable.beginBatchMode();
    fadeOutBranches();
    fadeInLayer(ACTUAL_IMAGE_INDEX);
    setProgress(progress);
    if (immediate) {
      mFadeDrawable.finishTransitionImmediately();
    }
    mFadeDrawable.endBatchMode();
  }
```
首先该方法会使用**WrappingUtils**工具类并且结合**RoundingParams**参数来重新生成一个可以附带圆角的**Drawable**。然后将新的**Drawable**交由**mActualImageWrapper**，结合上面的层次结果分析，会很容易知道这就是需要真实显示的图层。它是一个**ForwardingDrawable**类型，该类主要是对传入的目标**Drawable**进行相应的原生方法操作。下一步调用**FadeDrawable**的*beginBatchMode()*方法，该方法的作用主要为了图层批处理做标记，防止在批处理时进行**invalidate**操作。直到最后的*endBatchMode()*方法调用之后才标识着图层批处理操作结束。该批处理操作的目的是将**actual image**图层显示出来，所以首先调用*fadeOutBranches()*方法：

```
  private void fadeOutBranches() {
    fadeOutLayer(PLACEHOLDER_IMAGE_INDEX);
    fadeOutLayer(ACTUAL_IMAGE_INDEX);
    fadeOutLayer(PROGRESS_BAR_IMAGE_INDEX);
    fadeOutLayer(RETRY_IMAGE_INDEX);
    fadeOutLayer(FAILURE_IMAGE_INDEX);
  }
```
这里对上述提到的所有图层进行*fadeOutLayer()*操作，继续进入*fadeOutLayer()*方法

```
  private void fadeOutLayer(int index) {
    if (index >= 0) {
      mFadeDrawable.fadeOutLayer(index);
    }
  }
```
发现也很简单，无非就是调用了**FadeDrawable**的*fadeOutLayer()*方法。

```
  public void fadeOutLayer(int index) {
    mTransitionState = TRANSITION_STARTING;
    mIsLayerOn[index] = false;
    invalidateSelf();
  }
```
在之前已经提到过**FadeDrawable**本质上可以理解为时一个**Drawable**数组，内部都是围绕着数组整体进行操作。对应的还有*fadeInLayer()*

```
  private void fadeInLayer(int index) {
    if (index >= 0) {
      mFadeDrawable.fadeInLayer(index);
    }
  }
```
在**FadeDrawable**中除了用来保存相应的**Drawable**数组**mLayers**，还有与其相对应的**mIsLayerOn**布尔数组，该数组用来标识各个**Hierarchy**中的图层是否需要展示。所以*fadeOutLayer()*是对**index**处的图层进行隐藏标识。最终的显隐操作都会转化为在*void draw(Canvas canvas)*方法中进行**alpha**操作。

那么再回到之前的*setImage()*方法中，*fadeOutBranches()*是对相关的图层进行隐藏标识，然后再通过*fadeInLayer(ACTUAL_IMAGE_INDEX)*方法改变**actual image**图层的标识，将它改变成显示状态。最后如果有**progressBar image**图层的话，也将会由*setProgress(progress)*方法来体现。**immediate**是用来判断是否之后的显隐操作立马实现或者渐变过渡实现(内部就是对alpha进行百分比操作，内部有个mDurationMs，该时间值也是文章开头提到的builder中的fadeDuration)。这样整个的实际图片展示流程我们已经分析完毕，所以**Hierarchy**中最重要的还是对图层概念的理解。下面再对**GenericDraweeHierarchy**中的一些其它方法进行简要的说明：

* *Drawable buildActualImageBranch(Drawable drawable, @Nullable ScaleType scaleType, @Nullable PointF focusPoint, @Nullable ColorFilter colorFilter)* 构建实际展示图片的图层分支，内部对于**Drawable**的创建还是借助**WrappingUtils**工具类
* *Drawable buildBranch(@Nullable Drawable drawable, @Nullable ScaleType scaleType)* 构建除实际展示图片之外的其它图层分支，**Drawable**的创建也是借助**WrappingUtils**工具类
* *void resetFade()* 初始化图层结构

>主要的就这几个吧，其它的都已经在上面分析流程中详细说明了。

# Drawable
最后再整理一下**Fresco**中的一些相关的自定义的**Drawable**子类

1. **ArrayDrawable**：**Drawable**数组的集合体，通过**layers**数组来管理**Drawable**，内部的都是对数组集合中的每一个**Drawable**进行操作，类似与**Android**原生的**LayerDrawable**，只是它并不支持**add**与**remove**操作
2. **ForwardingDrawable**：对传入的目标**Drawable**即操作对象进行封装处理，该新类的方法可以调用目标对象对应的方法，同时保留目标**Drawable**的各个状态，不依赖与目标类的细节实现，提高新类的稳定性，该方式可以称之为复合。**Fresco**中绝大多数自定义**Drawable**都是它的子类。
3. **AutoRotateDrawable**：它继承于**ForwardingDrawable**，实现的是对**Drawable**的旋转操作
4. **FadeDrawable**：它继承于**ArrayDrawable**，之前也详细提到过，**Hierarchy**中的主要图层集合体。主要是通过**mLayers**与**mIsLayerOn**数组来控制数组中各个**Drawable**的**alpha**值，即显隐
5. **MatrixDrawable**：它继承于**ForwardingDrawable**，顾名思义通过矩阵来改变**Drawable**状态。
6. **OrientedDrawable**：它也继承于**ForwardingDrawable**，它不同于**AutoRotateDrawable**的是，它只支持90度的倍数角度旋转。
7. **ProgressBarDrawable**：进度条**Drawable**，支持横竖方向。
8. **RoundedBitmapDrawable**：继承于**BitmapDrawable**，根据**Bitmap**来创建有关圆角的**Drawable**，主要在**WrappingUtils**类中使用，用例构建全新的圆角**Drawable**
9. **RoundedColorDrawable**：继承于**Drawable**，根据**Color**来创建圆角**Drawable**，主要在**WrappingUtils**类中使用，用例构建全新的圆角**Drawable**
10. **RoundedCornersDrawable**：继承于**ForwardingDrawable**，用于设计**overLay**覆盖图片圆角，主要在**WrappingUtils**类中使用
11. **ScaleTypeDrawable**：继承于**ForwardingDrawable**，用于缩放类型的**Drawable**

# End
本篇文章主要是分析**Fresco**中有关**Hierarchy**相关的实现与原理，通过分析发现**Hierarchy**中都是对**Drawable**图层进行处理，并没有其它的缓存、请求之类的逻辑。所以如果你使用**Fresco**的时候只使用**Hierarchy**的话，就与别的**ImageView**没有多大的区别，真正的图层操作与缓存控制都在**Controller**中，所以下篇文章将进入**Controller**解析，来详细了解它的实现细节。
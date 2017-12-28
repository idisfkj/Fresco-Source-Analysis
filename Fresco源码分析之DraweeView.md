---
title: Fresco源码分析之DraweeView
date: 2017-11-28 21:28:05
tags: 源码分析
category: android
---

在`Android`中图片加载的框架很多，例如：`Fresco`、`Picasso`、`Glide`与`Imageloader`。它们都有各自的优点，但总的来说，使用起来方便简单、可配置性高与提供良好的缓存机制。由于平常主要用的还是`Fresco`，所以这里有必要对`Fresco`的原理进行深入研究。这样对于以后的使用与理解将会得到巨大的帮助。

`Fresco`是专注于对图片加载而设计的框架，所以对于以图片为主的`App`强烈推荐使用。`Fresco`对于图片的展示支持多种情况：`backgroud image`(背景图)、`placeholder image`(占位图)、`actual image`(加载的图片)、`progress bar image`(进度条)、`retry image`(重新加载的图片)、`failure image`(失败图片)与`overlay image`(叠加图)。`Fresco`既然支持这么多图片展示情况，那么它对这次图层的管理模式又是怎么样的呢？`Fresco`对于这些图层的管理都交给了`Hierarchy`，而这些图层的数据都通过`Controller`来设置。先不分析这些，这些后续文章会详细分析，今天先从`Fresco`的基本组件开始。

## SimpleDraweeView
> 首先这是对`Fresco`的源码分析，所以在看这篇文章之前你应该要有使用`Fresco`的基础，如果没有的强烈推荐看下[Fresco官方文档](https://www.fresco-cn.org/docs/getting-started.html)。

我们使用`Fresco`进行图片加载，使用最多的还是已经封装好的`SimpleDraweeView`，而在`SimpleDraweeView`的构造方法中会调用`init()`方法，它的源码如下：

```
private void init(Context context, @Nullable AttributeSet attrs) {
    if (isInEditMode()) {
      return;
    }
    Preconditions.checkNotNull(
        sDraweeControllerBuilderSupplier,
        "SimpleDraweeView was not initialized!");
    mSimpleDraweeControllerBuilder = sDraweeControllerBuilderSupplier.get();
 
    if (attrs != null) {
      TypedArray gdhAttrs = context.obtainStyledAttributes(
          attrs,
          R.styleable.SimpleDraweeView);
      try {
        if (gdhAttrs.hasValue(R.styleable.SimpleDraweeView_actualImageUri)) {
          setImageURI(
              Uri.parse(gdhAttrs.getString(R.styleable.SimpleDraweeView_actualImageUri)),
              null);
        } else if (gdhAttrs.hasValue((R.styleable.SimpleDraweeView_actualImageResource))) {
          int resId = gdhAttrs.getResourceId(
              R.styleable.SimpleDraweeView_actualImageResource,
              NO_ID);
          if (resId != NO_ID) {
            setActualImageResource(resId);
          }
        }
      } finally {
        gdhAttrs.recycle();
      }
    }
  }
```
这个方法做的事情很简单，但我们要注意的是它会对`sDraweeControllerBuilderSupplier`进行`null`判断，如果为`null`将会抛出异常。`sDraweeControllerBuilderSupplier `是供应类，通过它的`get`方法来获取`DraweeControllerBuilder`，这个是`controller`构造器，这个以后的章节会详细说明。空判断的目的就是在使用`SimpleDraweeView`之前必须初始化`sDraweeControllerBuilderSupplier`。在`SimpleDraweeView`中我们能找到它的初始化方法

```
public static void initialize(
      Supplier<? extends SimpleDraweeControllerBuilder> draweeControllerBuilderSupplier) {
    sDraweeControllerBuilderSupplier = draweeControllerBuilderSupplier;
  }
```
它是一个`static`方法，在`SimpleDraweeView`初始化之前加载一次即可。这是`SimpleDraweeView`的关键。它还有一个关键方法

```
public void setImageURI(Uri uri, @Nullable Object callerContext) {
    //通过controller 来保存Uri 等相关信息
    DraweeController controller = mSimpleDraweeControllerBuilder
        .setCallerContext(callerContext)
        .setUri(uri)
        .setOldController(getController())
        .build();
    setController(controller);
  }
```
使用`mSimpleDraweeControllerBuilder`来构建一个`Controller`，而`Controller`是由`build`模式所创建，这里我们能看到`uri`也交由`Controller`管理。其实最终`uri`会封装成一个`ImageRequest`，`Controller`真正持有的是`uri`的封装体`ImageRequest`。在`SimpleDraweeView`中它会重写`setImageURI`方法，最终也就是将`ImageView`中的原生方法给覆盖掉。还有其它的类似的`setImageResource`与`setImageBitmap`等，`Fresco`都在这些方法上加了`@Deprecated`，意思就是说不推荐使用，如果使用的话就跟直接使用`ImageView`没什么区别，这就无法体验到`Fresco`的强大的特性。在`Fresco`中统一由`setController`来替代。

>关于`Controller`后续文章会详细分析。

## Fresco
如果我们根据[Fresco官方文档](https://www.fresco-cn.org/docs/getting-started.html)的正常步骤来使用的话就无需担心这一步，因为在我们在使用`Fresco`之前都要先调用`Fresco.initialize(context)`

```
public static void initialize(
      Context context,
      @Nullable ImagePipelineConfig imagePipelineConfig,
      @Nullable DraweeConfig draweeConfig) {
    if (sIsInitialized) {
      FLog.w(
          TAG,
          "Fresco has already been initialized! `Fresco.initialize(...)` should only be called " +
            "1 single time to avoid memory leaks!");
    } else {
      sIsInitialized = true;
    }
    // we should always use the application context to avoid memory leaks
    context = context.getApplicationContext();
    if (imagePipelineConfig == null) {
      //初始化ImagePipeline工厂，包含ImagePipelineConfig 相关初始化配置信息
      // (三级缓存、图片解码/编码、转化、渐变、bitmap配置、四种executor 分别为 io、decode、background、lightweight background)等
      ImagePipelineFactory.initialize(context);
    } else {
       ImagePipelineFactory.initialize(imagePipelineConfig);
    }
    //初始化Drawee相关配置信息
    initializeDrawee(context, draweeConfig);
  }
```
除了初始化`ImagePipeline`之外，最后还会调用`initializeDrawee (context, draweeConfig)`，我们来看下`initializeDrawee`做了什么

```
private static void initializeDrawee(
      Context context,
      @Nullable DraweeConfig draweeConfig) {
    //构建PipelineDraweeControllerBuilderSupplier,
    //其中ImagePipeline、PipelineDraweeControllerFactory、ControllerListener set集合
    sDraweeControllerBuilderSupplier =
        new PipelineDraweeControllerBuilderSupplier(context, draweeConfig);
    //初始化SimpleDrawee
    //初始化时通过调用PipelineDraweeControllerBuilderSupplier实现的Supplier的get()方法
    // 返回配置信息的封装体PipelineDraweeControllerBuilder implements SimpleDraweeControllerBuilder
     SimpleDraweeView.initialize(sDraweeControllerBuilderSupplier);
  }
```
在这里我们会看到之前所提到的`sDraweeControllerBuilderSupplier`与`SimpleDraweeView`中必须优先调用的`initialize`方法。相信现在应该明白的为什么在使用`Fresco`之前必须调用它的`initialize`方法了。因为它必须要初始化一些必要的配置信息，其中就包括使用的控件`SimpleDraweeView`的配置信息。

## GenericDraweeView
上面所说的`SimpleDraweeView`的父类是`GenericDraweeView`，它做的事情很简单，处理`xml`相关的属性。它会通过`inflateHierarchy`方法进行初始化。

```
protected void inflateHierarchy(Context context, @Nullable AttributeSet attrs) {
    GenericDraweeHierarchyBuilder builder =
        GenericDraweeHierarchyInflater.inflateBuilder(context, attrs);
    setAspectRatio(builder.getDesiredAspectRatio());
    setHierarchy(builder.build());
  }
```
它是由`GenericDraweeHierarchyBuilder`来统一封装这些属性。最终通过`build`方法来构建`GenericDraweeHierarchy`，这就是`Fresco`的图层。然后通过`setHierarchy`将图层传递给`DraweeHolder`。`DraweeHolder`是用来管理`Hierarchy`与`Controller`的。而`DraweeHolder`是在最底层的`DraweeView`中，这也是`GenericDraweeView`的父类。下面我们进入`DraweeView `，来看看它到底做了什么。

## DraweeView
`DraweeView`是`Fresco`最底层的控件，也是我们使用它展示图片的基础，它继承于`ImageView`，所以它能做的事也无非于在原生`ImageView`上做扩展或者方法重写，从而来实现自己的一套逻辑。先看下它的构造方法

```
public DraweeView(Context context) {
    super(context);
    init(context);
  }
```
没什么特别的逻辑，就一个`init`方法，那么就进入`init`看看

```
/** This method is idempotent so it only has effect the first time it's called */
  private void init(Context context) {
    if (mInitialised) {
      return;
    }
    mInitialised = true;
    mDraweeHolder = DraweeHolder.create(null, context);
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
      ColorStateList imageTintList = getImageTintList();
      if (imageTintList == null) {
        return;
      }
      setColorFilter(imageTintList.getDefaultColor());
    }
    // In Android N and above, visibility handling for Drawables has been changed, which breaks
    // activity transitions with DraweeViews.
    mLegacyVisibilityHandlingEnabled = sGlobalLegacyVisibilityHandlingEnabled &&
        context.getApplicationInfo().targetSdkVersion >= 24; //Build.VERSION_CODES.N
  }
```
我们可以看到它会通过`mInitialised`来判断是否需要初始化，源码注释也说明的该方法只会调用一次。这是因为创建`Hierarchy`的代价太大，所以只会创建一次，以后都会使用同一个`mDraweeHolder`中的`Hierarchy`，所以会看到这里就必须初始化一个`mDraweeHolder`。在`DraweeView`中还有以下几个主要方法：

* `void setHierarchy(DH hierarchy)`设置`Hierarchy`,同时会将`Hierarchy`交由`mDraweeHolder`管理，最后还会调用`super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());`来将`Hierarchy`中的图层树显示出来。
* `Drawable getTopLevelDrawable()`会通过`mDraweeHolder`中的`Hierarchy`来获取图层树。
* `void setController(@Nullable DraweeController draweeController)`设置`Controller`，同时也会将`Hierarchy`中的图层树显示出来。
* `void onAttachedToWindow()`、`void onDetachedFromWindow()`、`void onStartTemporaryDetach()`与`void onFinishTemporaryDetach()`是来控制图层的显示与隐藏、绑定与解绑的回调函数，它们都分别会调用`mDraweeHolder`的`onAttach()`与`onDetach()`。其实最终调用的还是`Controller`中的`onAttach()`与`onDetach()`。
* `boolean onTouchEvent(MotionEvent event)`控制控件的触摸，如果`Controller`有效的话会调用`Controller`中的`onTouchEvent`。
* `void setAspectRatio(float aspectRatio)`用来设置`DraweeView`显示的宽高比例。
* `setImageDrawable`、`setImageBitmap`、`setImageResource`与`setImageURI`这些方法都是原生`ImageView`的方法，但在`DraweeView`中这些方法都被加上了`@Deprecated`标记。标明不推荐使用，如果一定使用的话，那么`DraweeView`将会退化成一个普通的`ImageView`。因为在`DraweeView`中都是通过`Controller`来体现它的缓存、加载机制等特性。

上面这些就是`DraweeView`的主要涉及到的方法与特性，不过在`DraweeView`中基本上每一个方法都涉及到了`DraweeHolder`，那它到底是干什么的呢？别急下面就轮到它了。

## DraweeHolder

```
A holder class for Drawee controller and hierarchy.
```
上面的是官方注释，说明`DraweeHolder`是用来管理`Hierarchy`与`Controller`的，同时也是它们之间的联系的桥梁。`DraweeView`以及它的子类都是通过它来间接操作`Controller`与`Hierarchy`。

```
public static <DH extends DraweeHierarchy> DraweeHolder<DH> create(
      @Nullable DH hierarchy,
      Context context) {
    DraweeHolder<DH> holder = new DraweeHolder<DH>(hierarchy);
    holder.registerWithContext(context);
    return holder;
  }
```
它是通过公有的静态方法来创建自身实例的。在上面的`DraweeView`的`init`方法中会调用。在其内部的`DraweeEventTracker`，是用来记录事件的传递，方便`dubug`的调试。如果不需要的话，可以在`Fresco.initialize()`之前调用`DraweeEventTracker.disable()`。那么剩下的方法其实基本上在`DraweeView`中都说过。

* `onAttach()`与`onDetach()`，都会调用`attachOrDetachController()`，根据情况分别调用`attachController()`与`detachController()`,最终调用的就是`Controller`的`onAttach()`与`onDetach()`
* `Drawable getTopLevelDrawable()`调用`mHierarchy.getTopLevelDrawable()`获取图层树。
* `void setController(@Nullable DraweeController draweeController)`设置`Controller`，在设置之前会先判断是否已经`wasAttached`，如果是的话就先调用`detachController()`，然后清除老的`Controller`，再将`Hierarchy`设置到新的`Controller`中。最后再`attachController()`进行绑定显示图层。
* `void setHierarchy(DH hierarchy)`设置`Hierarchy`，如果`Controller`有效的话就与`Hierarchy`建立链接，将`Hierarchy`设置到`Controller`中。

以上就是`DraweeHolder`的主要方法，都跟`Controller`与`Hierarchy`相关。而`DraweeHolder`又与`DraweeView`相连，所以最终还是要回到`Controller`与`Hierarchy`中。

## End
这次主要是分析了`Fresco`中的基本组件`DraweeView`与它的子类。如果你还想进一步了解`Hierarchy`与`Controller`的原理，下篇文章将会详细分析相关的原理，敬请期待！
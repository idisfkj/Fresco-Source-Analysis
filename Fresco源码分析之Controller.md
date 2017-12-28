---
title: Fresco源码分析之Controller
date: 2017-12-14 08:32:26
tags: 源码分析
category: android
---
如果你是第一次看我的**Fresco**的源码分析系列文章，这里强烈推荐你先阅读我的前面两篇文章[Fresco源码分析之DraweeView](https://idisfkj.github.io/2017/11/28/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8BDraweeView/)与[Fresco源码分析之Hierarchy](https://idisfkj.github.io/2017/12/07/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8BHierarchy/)。好了，下面进入正题。在上篇文章中我们提到，在**Fresco**中关于图片的缓存、请求与显示逻辑处理都在**Controller**中。那么**Controller**到底是如何贯穿这些功能的呢？我们先从它的出生开始。

# Suppiler
**PipelineDraweeControllerBuilderSupplier**是一个供应商，主要实现了`Supplier<T>`接口，它只有一个方法*T get()*，用来获取相关的提供实现。因此该供应类提供的就是**PipelineDraweeControllerBuilder**实例。

```
  @Override
  public PipelineDraweeControllerBuilder get() {
    return new PipelineDraweeControllerBuilder(
        mContext,
        mPipelineDraweeControllerFactory,
        mImagePipeline,
        mBoundControllerListeners);
  }
```
在生成的**builder**中有4个参数，第一个是**Context**再熟悉不过了；第二个是**Controller**的工厂；第三个是数据管道**ImagePipeline**；第四个是**listener**的**set**集合，主要用在图片请求之后的监听回调。下面详细说明后面三个参数内容与作用。

## PipelineDraweeControllerFactory
在这个类中主要就两个方法，分别为**internalCreateController**与**newController**，对外的方法就一个**newController**。这个两个方法都是用来创建**PipelineDraweeController**对象。其中**newController**内部就是调用了**internalCreateController**来进行创建**PipelineDraweeController**实例。

```
  protected PipelineDraweeController internalCreateController(
      Resources resources,
      DeferredReleaser deferredReleaser,
      DrawableFactory animatedDrawableFactory,
      Executor uiThreadExecutor,
      MemoryCache<CacheKey, CloseableImage> memoryCache,
      @Nullable ImmutableList<DrawableFactory> globalDrawableFactories,
      @Nullable ImmutableList<DrawableFactory> customDrawableFactories,
       Supplier<DataSource<CloseableReference<CloseableImage>>> dataSourceSupplier,
      String id,
      CacheKey cacheKey,
      Object callerContext) {
    PipelineDraweeController controller = new PipelineDraweeController(
        resources,
        deferredReleaser,
        animatedDrawableFactory,
        uiThreadExecutor,
        memoryCache,
        dataSourceSupplier,
        id,
        cacheKey,
        callerContext,
        globalDrawableFactories);
     controller.setCustomDrawableFactories(customDrawableFactories);
    return controller;
  }
```
其中**DeferredReleaser**是用来管理推迟资源释放的。我们在之前的文章已经提到，在**onAttach**中会进行加载资源，而**onDetach**中又会释放资源。因为在**Fresco**中往往会在**onDetach**与**onAttach**之间频繁切换(view的显隐、绘制与Controller的设置都会调用)，并且它们都处于在同一个**looper**(其实就是主进程的looper)中。如果在**onDetach**时马上释放资源的话，这样会造成资源的滥用，导致不必要的资源加载与释放回收。所以就用了这个资源推迟释放的机制(内部原理是使用了set集合的唯一性的特性)。

**dataSourceSupplier**是**DataSource**的供应商，用来提供**DataSource**实例。而**DataSource**是用来获取与存储请求结果的，相当与图片数据源。这些都会在后续的**Controller**中使用到。

## ImagePipeline
既然它是数据管道，自然是与网络请求与缓存数据有关。其实我们可以把它理解为多个管道的集合，最终显示的图片资源就是来自于它们中的其中一个。下面介绍其中的主要方法：

* *fetchDecodedImage()* 发送请求，返回**decode image**的数据源。
* *fetchEncodedImage()* 发送请求，返回**encoded image**的数据源。
* *prefetchToBitmapCache()* 发送预处理请求，获取预处理的**bitmap**缓存数据。
* *prefetchToDiskCache()* 发送预处理请求，获取预处理的磁盘缓存数据。
* *submitFetchRequest()* 发送请求，获取相应类型的数据源。
* *submitPrefetchRequest()* 发送预处理请求，获取相应类型的缓存数据。

这里用的最多的还是*fetchDecodedImage()*

```
  public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
      ImageRequest imageRequest,
      Object callerContext,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit) {
    try {
      Producer<CloseableReference<CloseableImage>> producerSequence =
           mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);
      return submitFetchRequest(
          producerSequence,
          imageRequest,
          lowestPermittedRequestLevelOnSubmit,
          callerContext);
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    }
  }
```
这里主要涉及到**Producer**，这是一个生产者，内部只有一个公共接口方法*void produceResults(Consumer<T> consumer, ProducerContext context)*，用来获取数据源。其实我们会发现**submitFetchRequest**方法中的**producer**传入的其实是一个队列，因为内部会递归调用*produceResults()*来获取最终的数据源。

>关于**Producer**后续有时间的话会单独开篇文章详细分析。

## ControllerListener
如果你对**ControllerListener**不熟悉的话，那么**BaseControllerListener**应该或多或少使用过吧。它其实就是**ControllerListener**的空实现。既然是监听回调，那么来看下它提供的回调方法与调用时机。

* *void onSubmit(String id, Object callerContext)* 在发送请求的时候回调
* *void onFinalImageSet(String id, @Nullable INFO imageInfo, @Nullable Animatable animatable)* 在最终设置**image**图片时回调，其中**imageInfo**包含图片的相关基本信息（width、height与quality）
* *void onIntermediateImageSet(String id, @Nullable INFO imageInfo)* 在发送请求与最终图片设置的过程中回调
* *void onIntermediateImageFailed(String id, Throwable throwable)* 在发送请求与最终失败的过程中回调
* *void onFailure(String id, Throwable throwable)* 发送请求失败时回调
* *void onRelease(String id)* 资源释放时回调

# ControllerBuilder
既然是**builder**模式，最终的目的自然就是用来创建**Controller**，所以我们可以直接奔着它的目的来分析。在这里**Controller**的**builder**类是**PipelineDraweeControllerBuilder**。我们找到它的*build()*发现在它的父类**AbstractDraweeControllerBuilder**中。但最终的创建实例方法还是调用了*obtainController()*抽象方法。所以经过反转还是回到了**PipelineDraweeControllerBuilder**，那么我们直接来看下它创建方式。

```
  @Override
  protected PipelineDraweeController obtainController() {
    DraweeController oldController = getOldController();
    PipelineDraweeController controller;
    if (oldController instanceof PipelineDraweeController) {
      controller = (PipelineDraweeController) oldController;
      controller.initialize(
          obtainDataSourceSupplier(),
          generateUniqueControllerId(),
          getCacheKey(),
          getCallerContext(),
          mCustomDrawableFactories);
    } else {
      controller = mPipelineDraweeControllerFactory.newController(
          obtainDataSourceSupplier(),
          generateUniqueControllerId(),
          getCacheKey(),
          getCallerContext(),
          mCustomDrawableFactories);
    }
    return controller;
  }
```
通过上面的代码，逻辑已经很明显了。首先判断是否已经存在**Controller**，如果存在的话就无需创建新的实例，只需调用*initialize()*方法进行重写初始化；如果不存在，那么就调用我们文章之前分析的**PipelineDraweeControllerFactory**中的*newController()*来创建新的实例。这里主要的参数还是*obtainDataSourceSupplier()*，之前也简单提到了，它是**DataSource**的供应者。那么我们来看下**Supplier**的创建

```
  protected Supplier<DataSource<IMAGE>> getDataSourceSupplierForRequest(
      final REQUEST imageRequest,
      final CacheLevel cacheLevel) {
    final Object callerContext = getCallerContext();
    return new Supplier<DataSource<IMAGE>>() {
      @Override
      public DataSource<IMAGE> get() {
        return getDataSourceForRequest(imageRequest, callerContext, cacheLevel);
      }
      @Override
      public String toString() {
        return Objects.toStringHelper(this)
            .add("request", imageRequest.toString())
            .toString();
      }
    };
  }
```
在这个方法中，我们一眼就看到了**Supplier**的创建，之前也提到它只有一个*get()*方法，就是用来提供所以需要的**DataSource**。在这里也是如此，这里它调用了*getDataSourceForRequest()*方法，该方法是一个抽象方法，细节实现由它的子类实现，所以我们可以再次回到**getDataSourceForRequest**，在其中就能够搜索到*getDataSourceForRequest()*方法

```
  @Override
  protected DataSource<CloseableReference<CloseableImage>> getDataSourceForRequest(
      ImageRequest imageRequest,
      Object callerContext,
      CacheLevel cacheLevel) {
    return mImagePipeline.fetchDecodedImage(
        imageRequest,
        callerContext,
        convertCacheLevelToRequestLevel(cacheLevel));
  }
```
看到上面的方法实现方法是否眼熟呢？这也是我们上面所提到的**ImagePipleline**中的方法，这里就不在多做分析了。这样**Controller**就与获取数据的通道建立了联系。那么下面我们就转战到**Controller**中，看看它到底做了什么。

# Controller
**PipelineDraweeController**继承于**AbstractDraweeController**，在**PipelineDraweeController**中主要的方法有三个

* *Drawable createDrawable(CloseableImage closeableImage)* 这是内部类**DrawableFactory**中的方法，是一个工厂，不言而喻它是用来创建**Drawable**的，在数据源返回的时候回调，进而显示到**Hierarchy**层。
* *getDataSource()* 获取数据源通道，与其建立联系。
* *void setHierarchy(@Nullable DraweeHierarchy hierarchy)* 设置**Hierarchy**图层，内部持有的其实是**SettableDraweeHierarchy**接口对象。所以内部调用的也就是它的6个接口方法。之前的文章也有提及，用来控制图片加载过程中的显示逻辑。

其余的逻辑处理都在它的父类**AbstractDraweeController**中。在之前我们多次提及到**onAttach**与**onDetach**方法，它们分别是处理数据加载与释放。

## onAttach

```
  @Override
  public void onAttach() {
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(
          TAG,
          "controller %x %s: onAttach: %s",
          System.identityHashCode(this),
          mId,
          mIsRequestSubmitted ? "request already submitted" : "request needs submit");
    }
//事件记录器   mEventTracker.recordEvent(Event.ON_ATTACH_CONTROLLER);
     Preconditions.checkNotNull(mSettableDraweeHierarchy);
    //取消资源推迟释放机制，防止资源被释放
    mDeferredReleaser.cancelDeferredRelease(this);
    mIsAttached = true;
    if (!mIsRequestSubmitted) {
      submitRequest();
    }
  }
```
在这个方法中**mEventTracker**是事件记录器，默认是开启的，如果要关闭则需要在*Fresco.initialize()*之前调用*DraweeEventTracker.disable()*关闭；然后就是将其从资源推迟释放机制中取消；最后就是调用*submitRequest()*发送数据源请求。

```
  protected void submitRequest() {
    final T closeableImage = getCachedImage();
    //1.判断内存缓存中是否存在
    if (closeableImage != null) {
      mDataSource = null;
      mIsRequestSubmitted = true;
      mHasFetchFailed = false;
       mEventTracker.recordEvent(Event.ON_SUBMIT_CACHE_HIT);
      //1.1数据获取中通知回调
      getControllerListener().onSubmit(mId, mCallerContext);
      //1.2数据处理
      onNewResultInternal(mId, mDataSource, closeableImage, 1.0f, true, true);
      return;
    }
     mEventTracker.recordEvent(Event.ON_DATASOURCE_SUBMIT);
    //2.通过DataSource获取数据源
    //2.1数据获取中通知回调
    getControllerListener().onSubmit(mId, mCallerContext);
    mSettableDraweeHierarchy.setProgress(0, true);
    mIsRequestSubmitted = true;
    mHasFetchFailed = false;
    mDataSource = getDataSource();
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(
          TAG,
          "controller %x %s: submitRequest: dataSource: %x",
          System.identityHashCode(this),
          mId,
          System.identityHashCode(mDataSource));
    }
    final String id = mId;
    final boolean wasImmediate = mDataSource.hasResult();
    //内部请求数据回调，当数据源返回时回调
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            // isFinished must be obtained before image, otherwise we might set intermediate result
            // as final image.
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            T image = dataSource.getResult();
            //2.2数据处理
            if (image != null) {
              //成功处理
              onNewResultInternal(id, dataSource, image, progress, isFinished, wasImmediate);
            } else if (isFinished) {
              //失败处理
              onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
            }
          }
          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            //失败处理
            onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
          }
          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            //数据进度处理
            onProgressUpdateInternal(id, dataSource, progress, isFinished);
          }
        };
    //数据源订阅回调注册
    mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
  }
```
逻辑方面的处理，上面代码中已经详细注释了，总的来说就是先从内存中获取如果存在就直接拿来用，否则就通过**DataSource**从网络或者是本地资源中获取。使用**DataSource**方式会使用到**DataSubscriber**，即订阅方式。当数据源已经获取到时，发送通知给订阅者，因此分别回调订阅者的方法。上述两种方式只要成功了都会交由*onNewResultInternal()*处理，而失败则由*onFailureInternal()*处理，同时请求进度处理由*onProgressUpdateInternal()*处理。

```
  private void onNewResultInternal(
      String id,
      DataSource<T> dataSource,
      @Nullable T image,
      float progress,
      boolean isFinished,
      boolean wasImmediate) {
    // ignore late callbacks (data source that returned the new result is not the one we expected)
    if (!isExpectedDataSource(id, dataSource)) {
      logMessageAndImage("ignore_old_datasource @ onNewResult", image);
      releaseImage(image);
      dataSource.close();
      return;
    }
    mEventTracker.recordEvent(
        isFinished ? Event.ON_DATASOURCE_RESULT : Event.ON_DATASOURCE_RESULT_INT);
    // create drawable
    Drawable drawable;
    try {
      drawable = createDrawable(image);
    } catch (Exception exception) {
      logMessageAndImage("drawable_failed @ onNewResult", image);
      releaseImage(image);
      onFailureInternal(id, dataSource, exception, isFinished);
      return;
    }
    T previousImage = mFetchedImage;
    Drawable previousDrawable = mDrawable;
    mFetchedImage = image;
    mDrawable = drawable;
    try {
      // set the new image
      if (isFinished) {
        logMessageAndImage("set_final_result @ onNewResult", image);
        mDataSource = null;
        //通过hierarchy（GenericDraweeHierarchy）来设置image
        mSettableDraweeHierarchy.setImage(drawable, 1f, wasImmediate);
        getControllerListener().onFinalImageSet(id, getImageInfo(image), getAnimatable());
        // IMPORTANT: do not execute any instance-specific code after this point
      } else {
        logMessageAndImage("set_intermediate_result @ onNewResult", image);
        mSettableDraweeHierarchy.setImage(drawable, progress, wasImmediate);
         getControllerListener().onIntermediateImageSet(id, getImageInfo(image));
        // IMPORTANT: do not execute any instance-specific code after this point
      }
    } finally {
      if (previousDrawable != null && previousDrawable != drawable) {
        releaseDrawable(previousDrawable);
      }
      if (previousImage != null && previousImage != image) {
        logMessageAndImage("release_previous_result @ onNewResult", previousImage);
        releaseImage(previousImage);
      }
    }
  }
```
这里我们主要就看里面的两个**try**。  
第一个使用*createDrawable(image)*将拿到的数据源转变成**Drawable**，这个方法的具体实现是在子类中实现(上面也有提及)。  
第二个分为两种情况，一方面如果数据源已经全部获取完，则直接调用**SettableDraweeHierarchy**接口的*setImage()*方法将图片设置到**Hierarchy**图层上，同时调用**Listener**的回调方法*onFinalImageSet()*;另一方面如果数据源还在获取中，也是调用**SettableDraweeHierarchy**接口的*setImage()*方法，只是其中的参数**progress**根据进度来设置而已，由于还处于资源获取中所以调用*onIntermediateImageSet()*回调。  
这样**Controller**就与**Hierarchy**联系起来了，将需要的图片设置到显示的图片中。

>对于**SettableDraweeHierarchy**中的这些方法如果不理解的可以回过头去看我之前的这篇文章[Fresco源码分析之Hierarchy](https://idisfkj.github.io/2017/12/07/Fresco%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8BHierarchy/)

由于源码太多，对于*onFailureInternal()*与*onProgressUpdateInternal()*这里就不贴出源码来进行分析了。原理与调用的方法基本类似，如果想看源码的可以点[这里](https://github.com/facebook/fresco/blob/master/drawee/src/main/java/com/facebook/drawee/controller/AbstractDraweeController.java)

## onDetach

```
  @Override
  public void onDetach() {
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(TAG, "controller %x %s: onDetach", System.identityHashCode(this), mId);
    }
    mEventTracker.recordEvent(Event.ON_DETACH_CONTROLLER);
    mIsAttached = false;
    mDeferredReleaser.scheduleDeferredRelease(this);
  }
```
相对于之前的分析*onDetach()*就简单多了，这里它只是对资源进行释放，释放的策略也是推迟释放策略[DeferredReleaser](https://github.com/facebook/fresco/blob/master/drawee/src/main/java/com/facebook/drawee/components/DeferredReleaser.java)。

# End
本篇文章主要分析了**Fresco**中的**Controller**相关处理逻辑，它控制着**Hierarchy**显示逻辑，同时它是数据源的获取桥梁通过**DataSource**来链接数据源的获取。那么问题又来了，**DataSource**又是如何产生的呢？同时它的内部逻辑又是如何的呢？这就涉及到**Producer**了，敬请关注下篇文章`Fresco源码分析之Producer`
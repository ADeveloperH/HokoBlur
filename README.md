## 动态模糊组件HokoBlur


### 1. 描述

- 组件主要提供以下功能：

	- 给图片添加模糊效果；
	- 动态模糊，对背景的实时模糊。

- 组件主要的特性：
	- 多种实现方案，包括RenderScript、OpenGL、Native和Java；
	- 多种算法，包括Box、Stack和Gaussian算法，满足不同的模糊效果；
	- 多核多线程，提升模糊效率，增加异步调用Api；
	- 类似iOS的动态背景模糊
	
	
### 2. 原理及性能分析 

可以参考[这里](doc/performance_analysis.md)。


### 3.使用姿势

#### 3.1 API调用

完整的api如下

```java
Blur.with(context)
    .scheme(Blur.SCHEME_NATIVE) //设置模糊实现方案，包括RenderScript、OpenGL、Native和Java实现，默认为Native方案
    .mode(Blur.MODE_STACK) //设置模糊算法，包括Gaussian、Stack和Box，默认并推荐选择Stack算法
    .radius(10) //设置模糊半径，内部最大限制为25，默认值5
    .sampleFactor(2.0f) // 设置scale因子，factor = 2时，内部将bitmap的宽高scale为原来的 1/2，默认值5
    .forceCopy(false) //对于scale因子为1.0f时，会直接修改传入的bitmap，如果你不希望修改原bitmap，设置forceCopy为true即可，默认值false
    .needUpscale(true) //设置模糊之后，是否upscale为原Bitmap的尺寸，默认值true
    .translateX(150)//可对部分区域进行模糊，这里设置x轴的偏移量
    .translateY(150)//可对部分区域进行模糊，这里设置y轴的偏移量
    .blurGenerator() //获得模糊实现类
    .blur(bitmap);	//模糊图片，方法是阻塞的，底层为多核并行实现，异步请使用doAsyncBlur

```
日常并不需要如此复杂的参数设置，如果单纯只是想添加模糊效果，可以这样调用：

```java
//doBlur()将返回模糊后的Bitmap
Bitmap outBitmap = Blur.with(context).blurGenerator().blur(bitmap);

```

对于尺寸很大的图，建议使用异步的方式调用


```java
Blur.with(this)
    .scheme(Blur.SCHEME_NATIVE)
    .mode(Blur.MODE_STACK)
    .radius(10)
    .sampleFactor(2.0f)
    .forceCopy(false)
    .needUpscale(true)
    .blurGenerator()
    .asyncBlur(bitmap, new AsyncBlurTask.CallBack() {
        @Override
        public void onBlurSuccess(Bitmap outBitmap) {
        	// do something...
        }

        @Override
        public void onBlurFailed() {

        }
    });

```

### 3.2 效果展示

#### 动画

<img src="doc/graphic/animation_blur_progress.gif" width = "370" height = "619" alt="动态模糊" />

#### 任意部位模糊

较高的模糊处理效率，可以实现任意部位的实时模糊。实际并不需要特别大尺寸的图只需要选取屏幕的一部分即可。

<img src="doc/graphic/dynamic_blur.gif" width = "370" height = "600" alt="动态模糊" />



### 4. 动态模糊

动态模糊提供了对View以及ViewGroup的实时背景模糊，并不是针对Bitmap的实现。组件将会对View所在区域进行模糊。

为View添加背景模糊，只需要将BlurDrawable设置为View背景即可。

```java
final BlurDrawable blurDrawable = new BlurDrawable();
View view = findViewById(R.id.test_view);
view.setBackgroundDrawable(blurDrawable);

```
模糊参数的调整，可以这样操作：

```java
BlurDrawable.mode(mode)
BlurDrawable.radius(radius)
BlurDrawable.sampleFactor(factor)

```

禁用/开启背景模糊

```java
BlurDrawable.disableBlur();
BlurDrawable.enableBlur();
```
组件已包含实现背景模糊的三种常用ViewGroup，包括BlurFrameLayout、BlurLinearLayout和BlurRelativeLayout。

使用示例：

```java
// 模糊动画
ValueAnimator animator = ValueAnimator.ofInt(0, 20);
animator.setDuration(2000);
animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
    @Override
    public void onAnimationUpdate(ValueAnimator animation) {
        mFrameLayout.getBlurDrawable().setBlurRadius((Integer) animation.getAnimatedValue());
    }
});

```
gif图较大，稍等片刻

<img src="doc/graphic/blur_drawable.gif" width = "370" alt="动态模糊" />



### 5. 配置
动态模糊正常工作，需要在混淆时加入下面的规则：

```java
-keep class com.mogujie.utils.blur.opengl.functor.** { *; }

```



### 6. 注意事项


1. 当未对Bitmap进行scale操作(```sampleFactor(1.0f)```)，传入的Bitmap将会被之后的操作直接修改。所以当函数返回某个bitmap的时候，可以被立刻使用到控件上面去。

2. **强烈建议使用在模糊操作之前，进行downScale操作，降低被模糊图片的大小，这将大幅提升模糊效率和效果。**

3. 请将模糊半径限制在25内（组件内部同样进行了限制），增加半径对模糊效果的提升远小于通过增加scale的缩放因子的方式，而且半径增加模糊效率也将降低；

4. RenderScript方案因为兼容性有待验证，因此暂时并未在该版本中，如果有需要更大计算量和更复杂模糊效果的场景，会考虑将RenderScript方案加回来。

5. 算法的选择
	- 如果你对模糊效果要求不高，同时希望较快完成图片的模糊，请选择Box算法；
	- 如果你对模糊效果要求较高，同时可以忍受较慢完成图片的模糊，请选择Gaussian算法；
	- Stack算法有非常接近Gaussian算法的模糊效果，同时提升了算法效率，一般情况下使用Stack算法即可；
6. BlurDrawable通过OpenGL实现，因此如果页面未开启硬件加速，背景模糊将无效。

6. 示例与用法
具体示例详见组件工程


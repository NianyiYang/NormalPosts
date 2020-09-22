# 慧投资 AndroidX 改造具体实践

## 思路

1. 按照 Android 官方网站的解决方案操作 

   [官方网站的 AndroidX 改造](https://developer.android.com/jetpack/androidx/migrate)

2. 将 `config.gradle` 中的依赖从 support 改写为 androidx

    [官方下载的映射表](../../../Downloads/androidx-artifact-mapping.csv) 

3. Glide，ButterKnife 等常用的第三方包已经适配了AndroidX，用最新的版本

4. 项目中一些使用反射获取属性的地方要修改



将

```groovy
support_multidex       : 'com.android.support:multidex:1.+',
support_v4             : 'com.android.support:support-v4:27.1.0',
support_design         : 'com.android.support:design:27.1.0',
support_appcompat      : 'com.android.support:appcompat-v7:27.1.0',
support_recyclerview   : 'com.android.support:recyclerview-v7:27.1.0',
support_percent        : 'com.android.support:percent:27.1.0',
support_constraint     : 'com.android.support.constraint:constraint-layout:1.0.2',
cardview               : 'com.android.support:cardview-v7:27.1.0',
gird_layout            : "com.android.support:gridlayout-v7:27.1.0",
```

改为

```groovy
support_multidex       : 'androidx.multidex:multidex:2.0.1',
support_v4             : 'androidx.legacy:legacy-support-v4:1.0.0',
support_design         : 'com.google.android.material:material:1.0.0',
support_appcompat      : 'androidx.appcompat:appcompat:1.0.0',
support_recyclerview   : 'androidx.recyclerview:recyclerview:1.1.0',
support_percent        : 'androidx.percentlayout:percentlayout:1.0.0',
support_constraint     : 'androidx.constraintlayout:constraintlayout:1.1.3',
cardview               : 'androidx.cardview:cardview:1.0.0',
gird_layout            : "androidx.gridlayout:gridlayout:1.0.0",
```



Robolectric 模块中的 pom 依赖需要修改

```
<groupId>androidx.legacy</groupId>
<artifactId>legacy-support-v4</artifactId>
<version>1.0.0</version>
```

依赖项中的 exclude 需要手工修改

`cangol.mobile:actionbar` 存疑



尝试修改 SmartRefreshLayout 版本，存在问题，回退版本



`FixedLoadMoreRecyclerView` 需要更新继承的方案

```java
private void createScroller() {
    try {
        // 升级 AndroidX 并兼容老版本
        if(this.getClass().getSuperclass() ==null) {
            return;
        }

        Field viewFlinger = this.getClass().getSuperclass().getDeclaredField("mViewFlinger");
        viewFlinger.setAccessible(true);

        Object viewFlingerObject = viewFlinger.get(this);

        Class<?> viewFlingerClazz = Class.forName("android.support.v7.widget.RecyclerView$ViewFlinger");
        Class<?> viewFlingerClazzAndroidX = Class.forName("androidx.recyclerview.widget.RecyclerView$ViewFlinger");

        Object scrollerCompatObject;
        Field mScrollerCompat;
        Object scroller = null;

        if (viewFlingerClazz.isInstance(viewFlingerObject)) {

            scrollerCompatObject = viewFlingerClazz.cast(viewFlingerObject);
            mScrollerCompat = viewFlingerClazz.getDeclaredField("mScroller");
            mScrollerCompat.setAccessible(true);

            scroller = mScrollerCompat.get(scrollerCompatObject);

        } else if (viewFlingerClazz.isInstance(viewFlingerClazzAndroidX)) {

            scrollerCompatObject = viewFlingerClazzAndroidX.cast(viewFlingerObject);
            mScrollerCompat = viewFlingerClazzAndroidX.getDeclaredField("mOverScroller");
            mScrollerCompat.setAccessible(true);

            scroller = mScrollerCompat.get(scrollerCompatObject);
        }

        if (scroller instanceof OverScroller) {
            overScroller = (OverScroller) scroller;
        } else if (scroller instanceof ScrollerCompat) {
            scrollerCompat = (ScrollerCompat) scroller;
        }

        Log.d(TAG, "release check ok");
    } catch (NoSuchFieldException e) {
        e.printStackTrace();
        Log.d(TAG, "release check failed->NoSuchFieldException:" + e.toString());
    } catch (IllegalAccessException e) {
        e.printStackTrace();
        Log.d(TAG, "release check failed->IllegalAccessException:" + e.toString());
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
        Log.d(TAG, "release check failed->ClassNotFoundException:" + e.toString());
    } catch (Throwable e) {
        e.printStackTrace();
    }
}
```



## 问题汇总

### 使用了第三方的工具类

> 拷贝到项目的基础库中，全局替换引用


### 找不到 `NotificationCompat.MediaStyle()`  

> 改为 `androidx.media.app.NotificationCompat.MediaStyle()`

### The given artifact contains a string literal with a package reference 'android.support.v4.content' that cannot be safely rewritten. Libraries using reflection such as annotation processors need to be updated manually to add support for androidx.

> 升级 ButterKnife 版本到 10.2.1

### 吸顶反射失效

> 全局搜索`Class.forName("android.support.v7.widget.RecyclerView$ViewFlinger")`
>
> 替换为 `Class.forName("androidx.recyclerview.widget.RecyclerView$ViewFlinger")`

`FixedLoadMoreRecyclerView`中不能替换，原因是这里的 RecyclerView 引用的还是老版本的库

> 项目中反射了`ViewFlinger`属性`mScroller`，经查询发现 AndroidX 中该属性改名为`mOverScroller`

> 依然失效，手动实验后发现升级 AndroidX ，`NestedScrollView$onNestedPreScroll()` 方法有两个实现（新版本增加了一个实现方法），加上这个实现即可
>
> ```kotlin
> @Override
> public void onNestedPreScroll(@NonNull View target, int dx, int dy, @NonNull int[] consumed, int type) {
>     if (target instanceof com.baidao.stock.chart.widget.FixedRecycleView
>             && !((com.baidao.stock.chart.widget.FixedRecycleView) target).isEnableDispatch()) {
>         return;
>     }
> 
>     //先parent消费
>     super.onNestedPreScroll(target, dx, dy, consumed, type);
>     //我再消费
>     if (dy > 0 && canScrollVertically(1)) {
>         scrollBy(0, dy - consumed[1]);//减去parent消费的距离
>         consumed[1] = dy;
>     } else {
>         if (target instanceof FixedJSWebView && !target.canScrollVertically(dy)) {
>             scrollBy(0, dy - consumed[1]);//减去parent消费的距离
>             consumed[1] = dy;
>         }
>     }
> }
> ```
>
> 


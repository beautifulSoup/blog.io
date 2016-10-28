---
layout: post
title: Android Drawable缓存问题 以及Resources源码分析
date: 2016-10-28
categories: blog
tags: [Development,Android]
description: 本篇博客主要追查了在用Resources获取Drawable时获取到同一个对象问题的追查以及对Resources部分源码的分析。
---


## Android Drawable缓存问题 以及Resources源码分析

### 起源
今天开发过程中遇到一个问题，定位到问题代码如下：

```java
		
    public static Drawable getColorFilteredDrawable(@DrawableRes int drawableRes, @ColorRes int colorRes){
        Context context = App.getContext();
        Drawable drawable = ContextCompat.getDrawable(context, drawableRes);
        drawable.setColorFilter(getColor(colorRes), PorterDuff.Mode.SRC_ATOP);
        return drawable;
    }
```
这段代码的期望是通过resId产生不同的Drawable，并改变颜色，产生不同颜色的Drawable对象。但是最后发现该方法返回相同resId的Drawable都是同一个颜色，所以颜色只与最后调用的setColorFilter方法有关。初步断定可能是Android系统缓存了Drawable对象。为了进一步确认这个问题，决定进到Android Sdk源码中看一下。

### 源码分析
跟踪代码到Resources类的loadDrawable方法。通过分析代码，发现其中有一段就是判断是否缓存了该resId和Theme所对应的Drawable，如果缓存了，就返回了缓存这的对象，这也是getResources.getDrawable()方法返回同一个Drawable对象的原因。

```java
    @Nullable
    Drawable loadDrawable(TypedValue value, int id, Theme theme) throws NotFoundException {
        ...

        // First, check whether we have a cached version of this drawable
        // that was inflated against the specified theme.
        if (!mPreloading) {
            final Drawable cachedDrawable = caches.getInstance(key, theme);
            if (cachedDrawable != null) {
                return cachedDrawable;
            }
        }
        ...
    }
```

### 解决方案

既然知道了原因，那么怎么做到获得相同resId的不同Drawable对象呢。通过查找代码，我们发现了Drawable有一个工厂类ConstantState。它保存了一些共享的常量，并且也可以作为工厂类来产生新的Drawable。但是这样产生的Drawable还是共享了这个ConstantState对象，所以为了让Drawable完全独立，还需要调用mutate()方法同时拷贝里面的ConstantState对象，可以理解为DeepCopy。

```java
 /**
     * This abstract class is used by {@link Drawable}s to store shared constant state and data
     * between Drawables. {@link BitmapDrawable}s created from the same resource will for instance
     * share a unique bitmap stored in their ConstantState.
     *
     * <p>
     * {@link #newDrawable(Resources)} can be used as a factory to create new Drawable instances
     * from this ConstantState.
     * </p>
     *
     * Use {@link Drawable#getConstantState()} to retrieve the ConstantState of a Drawable. Calling
     * {@link Drawable#mutate()} on a Drawable should typically create a new ConstantState for that
     * Drawable.
     */
    public static abstract class ConstantState {
    	...
    	 public abstract Drawable newDrawable();

     
   	     public Drawable newDrawable(Resources res) {
            return newDrawable();
         }

        public Drawable newDrawable(Resources res, Theme theme) {
            return newDrawable(null);
         }
        ...
    	
    }
```

所以最后通过下面这句代码就可以获得完全独立的Drawable对象，随便修改也不会影响其他地方了。

```java
Drawable drawable = ContextCompat.getDrawable(context, drawableRes).getConstantState().newDrawable().mutate();
```

### 扩展

与Drawable类似的，系统还缓存了ColorStateList，Animation，StateListAnimator，如果通过Resources取得这些资源的对象，又想对其进行修改的话就要注意了。



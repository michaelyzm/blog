---
layout: post
title: Ceaser的原理和基于安卓的实现
date: 2017-07-28
categories: 架构
tags: [架构, 分析, 实现]
---

### Ceaser的Ease有哪些
Ceaser是一个非常好用的CSS动画效果工具项目，通过选择各种已经实现好的效果，可以得到现成的CSS代码直接使用。
  
目前，这个项目有很多程序员为它做了扩展，在本文参考的Mathew Lein的项目(<https://matthewlein.com/ceaser/>)中，
一共包含30种实现，其中包括5种默认效果，24种预置的EaseIn和EaseOut的效果，以及1种自定义效果。
### Ceaser的Ease是用什么方法实现的
根据网页中生成的代码，我们可以发现所有的Ease效果都是用三阶贝塞尔曲线表示的，比如ease(default)方案，其代码如下

    -webkit-transition: all 500ms cubic-bezier(0.250, 0.100, 0.250, 1.000); 
       -moz-transition: all 500ms cubic-bezier(0.250, 0.100, 0.250, 1.000); 
         -o-transition: all 500ms cubic-bezier(0.250, 0.100, 0.250, 1.000); 
            transition: all 500ms cubic-bezier(0.250, 0.100, 0.250, 1.000); /* ease (default) */

    -webkit-transition-timing-function: cubic-bezier(0.250, 0.100, 0.250, 1.000); 
       -moz-transition-timing-function: cubic-bezier(0.250, 0.100, 0.250, 1.000); 
         -o-transition-timing-function: cubic-bezier(0.250, 0.100, 0.250, 1.000); 
            transition-timing-function: cubic-bezier(0.250, 0.100, 0.250, 1.000); /* ease (default) */
  
从代码中可以看出这些动画效果都是由三阶贝塞尔曲线表示的，唯一的不同是传入的四个参数。
  
### 贝塞尔曲线是什么
  
贝塞尔曲线，又叫贝兹曲线或者贝济埃曲线，是应用于二维图形应用程序的数学曲线，它是1962年由法国工程师皮埃尔·贝塞尔
所广泛发表的，起初主要用于汽车主体设计。
##### 一阶贝塞尔曲线的公式如下

$$B(t) = P_0 + (P_1 - P_0)t = (1 - t)P_0 + tP_1, t\in[0,1]$$

##### 二阶贝塞尔曲线的公式如下
  
$$B(t) = (1-t)^2P_0 + 2t(t-1)P_1 + t^2P_2, t\in[0,1]$$

##### 三阶贝塞尔曲线的公式如下
  
$$B(t) = P_0(1-t)^3 + 3P_1t(1-t)^2 + 3P_2t^2(1-t) + P_3t^3, t\in[0,1]$$

这里的\\(P_0\\)、\\(P_1\\)、\\(P_2\\)和\\(P_3\\)就是三阶贝塞尔曲线最重要的四个元素，其中\\(P_0\\)是贝塞尔曲线的起点，对于Ease来说，默认为(0, 0)。
\\(P_3\\)是贝塞尔曲线的终点，对于Ease来说默认为(1, 1)。 那么剩下来的\\(P_1\\)和\\(P_2\\)则是三阶贝塞尔曲线的两个控制点，分别假定为$(x_0, y_0)$和$(x_1, y_1)$，
那么cubic-bezier函数对应的四个参数就是$(x_0, y_0, x_1, y_1)$。
  
### 在安卓里怎么使用它
安卓里边的动画分为Animation和Animator两种，但是不管哪一种，都会用到插值器(Interpolator)，而Ease对应的就是插值器的功能。那么安卓里插值器是如何定义的呢？
  
	/**
	 * An interpolator defines the rate of change of an animation. This allows
	 * the basic animation effects (alpha, scale, translate, rotate) to be 
	 * accelerated, decelerated, repeated, etc.
	 */
	public interface Interpolator extends TimeInterpolator {
	    // A new interface, TimeInterpolator, was introduced for the new android.animation
	    // package. This older Interpolator interface extends TimeInterpolator so that users of
	    // the new Animator-based animations can use either the old Interpolator implementations or
	    // new classes that implement TimeInterpolator directly.
	}
这是Interpolator定义，可以看出它是一个interface，而且只是简单继承了TimeInterpolator，下面看一下TimeInterpolator的定义。
  
	/**
	 * A time interpolator defines the rate of change of an animation. This allows animations
	 * to have non-linear motion, such as acceleration and deceleration.
	 */
	public interface TimeInterpolator {

	    /**
	     * Maps a value representing the elapsed fraction of an animation to a value that represents
	     * the interpolated fraction. This interpolated value is then multiplied by the change in
	     * value of an animation to derive the animated value at the current elapsed animation time.
	     *
	     * @param input A value between 0 and 1.0 indicating our current point
	     *        in the animation where 0 represents the start and 1.0 represents
	     *        the end
	     * @return The interpolation value. This value can be more than 1.0 for
	     *         interpolators which overshoot their targets, or less than 0 for
	     *         interpolators that undershoot their targets.
	     */
	    float getInterpolation(float input);
	}
  
TimeInterpolator里有一个getInterpolation的函数，接受一个0到1之间的浮点数作为传入参数, 这个浮点数表示当前动画已播放的时间占总时间的百分比。
从这个函数的定义，我们就能发现，如果要构造一个三阶贝塞尔曲线的Interpolator，三阶贝塞尔曲线公式中的$t$实际上不是传入参数，$B(t)$所对应的点的$x_t$才是传入的input
，而$B(t)$对应的点的$y_t$则是函数所需要的输出。经过以上分析，我们就可以开始写我们自己的贝塞尔曲线Interpolator了。
  
首先，我们的已知条件是$x_t$，根据已知条件，我们可以得到简化公式
$$
x_t = 3x_0t(1-t)^2 + 3x_1t^2(1-t) + t^3, t\in[0,1]
$$
要根据$x_t$计算$t$比较麻烦，而且存在虚根的问题，作为程序员，我们当然要用数值计算中最经典的牛顿迭代法来解决这种解方程的问题啦。但是这个函数是否单调递增或者单调递减呢？
当t=0时x_t的取值为0，t=1时x_t的取值为1，所以这个函数如果单调的话，只能是单调递增。那么怎么保证它单调递增呢，这里不过多的讨论数学问题，只按照大家的结论，直接认定当$t\in[0,1]$时，
$x_t$的值是随t递增的，所以我们可以从小到大找到第一个大于等于input的$x_t$，从而得到对应的t，并用它来计算$y_t$，$y_t$的简化公式如下
$$
y_t = 3y_0t(1-t)^2 + 3y_1t^2(1-t) + t^3, t\in[0,1]
$$

首先我们要实现一个三阶贝塞尔曲线的计算公式的函数，即用t计算$x_t$和$y_t$的函数

    float cubicBezier(float t, float c0, float c1)
    {
        return 3*c0*t*(1-t)*(1-t) + 3*c1*t*t*(1-t) + t*t*t;
    }

函数的输入参数中，c0和c1分别表示控制点$P_1$和$P_2$的坐标，当计算$x_t$的时候用$x_0$和$x_1$，计算$y_t$的时候用$y_0$和$y_1$。然后再实现利用$x_t$估算t的函数:

    float estimateT(float x)
    {
        float result = 0;
        float step = 0.001f;
        for(int i = 0; i < 1000; ++i)
        {
            if(cubicBezier(result, x0, x1) >= x)
            {
                break;
            }
            else
            {
                result += step;
            }
        }
        return result;
    }

利用这两个函数，我们可以最后获得$y_t$：

    @Override
    public float getInterpolation(float input) {
        float t = estimateT(input);
        float yResult = cubicBezier(t, y0, y1);
        return yResult;
    }
  
经过测试，每千次调用所需时间为400ms,考虑到安卓每帧画面的绘制时间为16.67ms，运算速度对画面绘制不会构成明显影响，这个算法就暂时不需要改进了。
至此，我们已经实现了三阶贝塞尔曲线插值器的类，然后我们可以通过Ceaser上的各类Ease贝塞尔曲线来实现我们所需要的插值器，改善我们的动画效果了。
  
需要完整代码的可以到<https://github.com/michaelyzm/Interpolator>获取，在这个工程中我还提供了一个贝塞尔曲线预览和点击调整的小工具，可以用来设计自定义的动画，欢迎使用。

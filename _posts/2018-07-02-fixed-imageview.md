---
layout: post
title: ImageView 固定寬高比
description: Android ImageView 固定寬高比
categories: [android]
keywords: android, imageview, fixed-aspect-ratio
---

一个可自定义固定寬高比的 ImageView。
<!-- more -->
在图片尺寸已知的情况下，加载大量图片时使用固定的寬高比的 ImageView。
setMeasuredDimension 方法决定了 View 大小，在 onMeasure 重设 ImageView 的大小。

{% highlight kotlin %}

class FixedImageView(context: Context, attrs: AttributeSet) : AppCompatImageView(context, attrs) {

    private var widthWeight = 1
    private var heightWeight = 1

    init {
        val a = context.obtainStyledAttributes(attrs, R.styleable.FixedImageView)
        widthWeight = a.getInteger(R.styleable.FixedImageView_widthWeight, 1)
        heightWeight = a.getInteger(R.styleable.FixedImageView_heightWeight, 1)
        a.recycle()
    }

    fun setWidthAndHeightWeight(widthWeight: Int, heightWeight: Int) {
        this.widthWeight = widthWeight
        this.heightWeight = heightWeight
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec)
        val width = this.measuredWidth
        val height = width * heightWeight / widthWeight
        setMeasuredDimension(width + paddingLeft + paddingRight, height + paddingTop + paddingBottom)
    }
}

{% endhighlight %}


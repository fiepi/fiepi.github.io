---
layout: post
title: 使用 ConstraintLayout 固定 ImageView 寬高比
description: 使用 ConstraintLayout 固定 ImageView 寬高比
categories: [android]
keywords: android, imageview, fixed-aspect-ratio, constraintlayout
---

前一篇动态修改 ImageView 的寬高比，在 RecyclerView 瀑布流加载大量寬高比不同的图片时会出现抖动移位。
现改用 ConstraintLayout 实现可避免这种情况。
<!-- more -->
item 布局，初始比例 1:1

{% highlight xml %}

<android.support.constraint.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">
    <ImageView
        android:id="@+id/post_item"
        android:scaleType="centerCrop"
        android:contentDescription="@null"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        app:layout_constraintDimensionRatio="H, 1:1"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintTop_toTopOf="parent"/>
    <ImageView
        android:id="@+id/rate"
        android:src="@drawable/ic_action_star_border_24dp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="4dp"
        android:contentDescription="@null"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"/>
</android.support.constraint.ConstraintLayout>

{% endhighlight %}

加载图片前在 onBindViewHolder 方法修改 ImageView 寬高比

{% highlight kotlin %}

	val lp = holder.post.layoutParams as ConstraintLayout.LayoutParams
	lp.dimensionRatio = "H, ${posts[position].actual_preview_width}:${posts[position].actual_preview_height}"
        holder.post.layoutParams = lp
        GlideApp.with(context)
        	.load(MoeGlideUrl(posts[position].preview_url))
                .fitCenter()
                .placeholder(context.resources.getDrawable(placeHolderId, context.theme))
                .into(holder.post)

{% endhighlight %}


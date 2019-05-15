---
layout: post
title:  "从零开始：优化代码之MultiTypeAdapter"
time: 2019-05-18
categories: collect
tags: 从零开始 组件 优化 Android Adapter
---


Android 开发中经常会需要显示列表，目前 Google 官方推荐的显示方案是：`RecyclerView` + `Adapter`。  
RecyclerView 并没有太复杂的属性配置；所以，要实现列表的展示及相关逻辑，我们主要的工作内容是实现自己的 Adapter。  


下面我们就以一个类似朋友圈的列表，写一个常见的多类型 Adapter 实现：


```kotlin

class MyAdapter(private val datas: List<Any>): Adapter<ViewHolder>() {
    companion object {
        const val TYPE_TEXT = 1
        const val TYPE_IMAGE = 2
        const val TYPE_AD = 100
    }

    override fun getItemCount(): Int = datas.size

    override fun getItemViewType(position: Int): Int {
        val data = datas[position]
        if (data is PostMsg) {
            return data.type
        } else if (data is AdMsg) {
            return TYPE_AD
        } else {
            throw IllegalArgumentException("Unkown data: $data")
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val inflater = LayoutInflater.from(parent.context)
        if (viewType == TYPE_TEXT) {
            return PostTextViewHolder(inflater.inflate(R.layout.item_left, parent, false))
        } else if (viewType == TYPE_IMAGE) {
            return PostImageViewHolder(inflater.inflate(R.layout.item_left, parent, false))
        } else if (viewType == TYPE_AD) {
            return AdViewHolder(inflater.inflate(R.layout.item_right, parent, false))
        } else {
            throw IllegalArgumentException("Unkown viewType: $viewType")
        }
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        if (holder is PostTextViewHolder) {
            val postMsg = datas[position] as PostMsg
            holder.tvName.text = "postion: $position  ${postMsg.label}"
            holder.tvName.setOnClickListener {
                // tvName的点击事件
            }
        } else if (holder is PostImageViewHolder) {
            val postMsg = datas[position] as PostMsg
            holder.tvName.text = "postion: $position  ${postMsg.label}"
            holder.tvName.setOnClickListener {
                // tvName的点击事件
            }
        } else if (holder is AdViewHolder) {
            val adMsg = datas[position] as AdMsg
            holder.tvName.text = "postion: $position  ${adMsg.label}"
            holder.tvName.setOnClickListener {
                // tvName的点击事件
            }
        }
    }

    internal class PostTextViewHolder(itemView: View) : ViewHolder(itemView) {
        var tvName: TextView = itemView.findViewById(R.id.tvName)
    }
    internal class PostImageViewHolder(itemView: View) : ViewHolder(itemView) {
        var tvName: TextView = itemView.findViewById(R.id.tvName)
    }
    internal class AdViewHolder(itemView: View) : ViewHolder(itemView) {
        var tvName: TextView = itemView.findViewById(R.id.tvName)
    }
}

```

事实上，我们真实业务中遇到的，可能要比上面这个例子的类型更多、逻辑更复杂。而且随着需求更迭，类型随时可能增减。维护起来非常不方便。

我期望的结果是：不需要每次都实现一个新的 Adapter，我只需要关系有多少种 ViewHolder，以及每种 ViewHolder 对应的业务逻辑。

下面我们就开始来优化我们的代码，把上面那些机械的、麻烦的逻辑统一处理，让我们只需要写自己真正的业务逻辑就好了。

### 第一步：优化 onBindViewHolder()

这里每增加一种类型，都要多写一个 if 判断条件，重复而且麻烦，期望能省去这里的类型判断。

我的解决方案是：让 ViewHolder 统一提供 onBindData() 方法，在内部实现自己的逻辑；这样就不需要在 onBindViewHolder() 去判断是哪个 ViewHolder 了。

改造结果如下：

```kotlin

override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    if (holder is BasicViewHolder) {
        // 这里统一交由 ViewHolder 内部去处理相关逻辑
        (holder as BasicViewHolder<Any>).onBindData(position, datas[position])
    }
}

```

### 第二步：优化 onCreateViewHolder()

和 onBindViewHolder() 不同的是，这里只需要根据不同的 type，创建相对应的 ViewHolder 就行了，逻辑单一，并没有复杂的判断。

我提供了一个 ViewHolderFactory 专门用来创建 ViewHolder，并且由外部传入。这样的好处是可以解耦，把独立的逻辑拆分到不同的类中，减少了和具体某个 Adapter 的依赖。  
之后我们就可以用相同的 Adapter + 不同的 ViewHolderFactory 组合成列表要展示的数据。

改造结果如下：

```kotlin

override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
    val inflater = LayoutInflater.from(parent.context)
    return viewHolderFactory.onCreateViewHolder(inflater, parent, viewType)
}

```

### 第三步：优化 getItemViewType()

onCreateViewHolder() 可以根据 type 来创建 ViewHolder，而 type 则是由 getItemViewType() 的返回值来决定的。

很明显，他们有一个承上启下的作用，关系非常紧密。我决定把他一起丢给 ViewHolderFactory 去处理。

并且，我们约定`一个 ViewHolder 对应一个布局`。所以，getItemViewType() 的返回值取其对应的 layoutResId 即可。

改造结果如下：

```kotlin

override fun getItemViewType(position: Int): Int = viewHolderFactory.getLayoutResId(position, datas[position]!!)

```

现在，我们将这个通用的 Adapter 命名为：MultiTypeAdapter。

一起来看下它的完整代码：

```kotlin

class MultiTypeAdapter(private val datas: List<Any>,
                       private val viewHolderFactory: BasicViewHolderFactory): Adapter<ViewHolder>() {
    override fun getItemCount(): Int = datas.size

    override fun getItemViewType(position: Int): Int = viewHolderFactory.getLayoutResId(position, datas[position])

    override fun onCreateViewHolder(parent: ViewGroup, layoutResId: Int): BasicViewHolder<*> {
        val inflater = LayoutInflater.from(parent.context)
        return viewHolderFactory.onCreateViewHolder(inflater, parent, layoutResId)
    }

    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        if (holder is BasicViewHolder<*>) {
            // 这里统一交由 ViewHolder 内部去处理相关逻辑
            (holder as BasicViewHolder<Any>).onBindData(position, datas[position])
        }
    }
}

abstract class BasicViewHolderFactory {
    @LayoutRes
    abstract fun getLayoutResId(position: Int, data: Any): Int
    abstract fun onCreateViewHolder(inflater: LayoutInflater, parent: ViewGroup, @LayoutRes layoutResId: Int): BasicViewHolder<*>
}

abstract class BasicViewHolder<T>(itemView: View): ViewHolder(itemView) {
    abstract fun onBindData(position: Int, data: T)
}

```

使用时，我们只需要根据需要提供不同的 ViewHolderFactory。

代码如下：

```kotlin

adapter = MultiTypeAdapter(datas, object: BasicViewHolderFactory() {
    override fun getLayoutResId(position: Int, data: Any): Int {
        return when {
            (data is PostMsg && data.isTextPost()) -> R.layout.item_post_text
            (data is PostMsg && data.isImagePost()) -> R.layout.item_post_image
            (data is AdMsg) -> R.layout.item_ad
            else -> throw IllegalArgumentException("Unkown data: $data")
        }
    }

    override fun onCreateViewHolder(inflater: LayoutInflater, parent: ViewGroup, layoutResId: Int): BasicViewHolder<*> {
        val itemView = inflater.inflate(layoutResId, parent, false)
        return when(layoutResId) {
            R.layout.item_post_text -> PostTextViewHolder(itemView)
            R.layout.item_post_image -> PostImageViewHolder(itemView)
            R.layout.item_ad -> AdViewHolder(itemView)
            else -> throw IllegalArgumentException("Unkown viewType: $layoutResId")
        }
    }
})

```

之后在每个 ViewHolder 实现自己的业务逻辑。

例如：

```kotlin

class PostTextViewHolder(itemView: View) : BasicViewHolder<PostMsg>(itemView) {
    var tvName: TextView = itemView.findViewById(R.id.tvName)

    override fun onBindData(position: Int, data: PostMsg) {
        tvName.text = "postion: $position  ${data.label}"
        tvName.setOnClickListener {
            // tvName的点击事件
        }
    }
}

```

到此为止，我们得到逻辑简单，并且可以复用的 Adapter。每个 ViewHolder 也比较简单，只需要关心自己的业务逻辑。  
但是 ViewHolderFactory 的 getLayoutResId() 和 onCreateViewHolder() 还是存在相似的类型判断。是否还可以再继续优化下呢？  


**前方高能预警！！！请做好准备**

`getLayoutResId()` 用于区分类型，这里的类型判断是没办法省去的。  
`onCreateViewHolder()` 则是一一对应关系，只需要根据提供的 layoutResId 创建对应的 ViewHolder 就可以了。  
所以我们的目标就是尽量省去 `onCreateViewHolder()` 的类型判断。  

`onCreateViewHolder()` 需要2个数据来创建 ViewHolder 实例：layoutResId 以及 目标ViewHolder的类型。  
如果在 getLayoutResId() 同时返回 layoutResId、viewHolderClass，岂不是就搞定了？  

说干就干，下面先定义一个实体类：ViewHolderType

```kotlin

data class ViewHolderType(
	val clazz: KClass<out BasicViewHolder<*>>,
	@LayoutRes val layoutResId: Int
)

```

接着改造 `BasicViewHolderFactory`：  
- 将 `getLayoutResId()` 的返回值改成 ViewHolderType；  
- 去掉 `onCreateViewHolder()` 方法。

```kotlin

abstract class BasicViewHolderFactory {
    abstract fun getViewHolderType(position: Int, data: Any): ViewHolderType
}

```

最后，修改 `MultiTypeAdapter` 的实现：  
- 让 `getItemViewType()` 根据 ViewHolderType 自动生成 viewType;  
- `onCreateViewHolder()` 再通过 viewType 找到对应的 ViewHolderType，并创建 ViewHolder 实例。

```kotlin

private val holderTypes by lazy { mutableMapOf<ViewHolderType, Int>() }
private val viewTypeGenerator by lazy { AtomicInteger(1) }

override fun getItemViewType(position: Int): Int {
    val holderType = viewHolderFactory.getViewHolderType(position, datas[position])
    if (holderTypes.containsKey(holderType)) {
        return holderTypes[holderType]!!
    }

    val viewType = viewTypeGenerator.getAndIncrement()
    holderTypes[holderType] = viewType
    return viewType
}

override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): BasicViewHolder<*> {
    holderTypes.forEach { (_holderType, _viewType) ->
        if (_viewType == viewType) {
            val itemView = LayoutInflater.from(parent.context).inflate(_holderType.layoutResId, parent, false)
            return _holderType.clazz.newInstance(itemView)
        }
    }
    throw IllegalArgumentException("Unkown viewType: $viewType")
}

```

最终使用时，只需要写一次类型判断即可：

```kotlin

adapter = MultiTypeAdapter(datas, object: BasicViewHolderFactory() {
    override fun getViewHolderType(position: Int, data: Any): ViewHolderType {
        return when {
            (data is PostMsg && data.isTextPost()) -> ViewHolderType(PostTextViewHolder::class, R.layout.item_post_text)
            (data is PostMsg && data.isImagePost()) -> ViewHolderType(PostImageViewHolder::class, R.layout.item_post_image)
            (data is AdMsg) -> ViewHolderType(AdViewHolder::class, R.layout.item_ad)
            else -> throw IllegalArgumentException("Unkown data: $data")
        }
    }
})

```

< 完结 >

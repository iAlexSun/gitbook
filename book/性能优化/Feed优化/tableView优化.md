## 1.如何提升 `tableview` 的流畅度？


本质上是降低 CPU、GPU 的工作，从这两个大的方面去提升性能。

- CPU：对象的创建和销毁、对象属性的调整、布局计算、文本的计算和排版、图片的格式转换和解码、图像的绘制

- GPU：纹理的渲染

  

参考资料:
iOS 保持界面流畅的技巧](https://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)

### 1.优化在 CPU 层面

#### 1.1 预排版

当获取到 API JSON 数据后，我会把每条 Cell 需要的数据都在后台线程计算并封装为一个布局对象 CellLayout。CellLayout 包含所有文本的 CoreText 排版结果、Cell 内部每个控件的高度、Cell 的整体高度。每个 CellLayout 的内存占用并不多，所以当生成后，可以全部缓存到内存，以供稍后使用。这样，TableView 在请求各个高度函数时，不会消耗任何多余计算量；当把 CellLayout 设置到 Cell 内部时，Cell 内部也不用再计算布局了。

对于通常的 TableView 来说，提前在后台计算好布局结果是非常重要的一个性能优化点。为了达到最高性能，你可能需要牺牲一些开发速度，不要用 Autolayout 等技术，少用 UILabel 等文本控件。但如果你对性能的要求并不那么高，可以尝试用 TableView 的预估高度的功能，并把每个 Cell 高度缓存下来。这里有个来自百度知道团队的开源项目可以很方便的帮你实现这一点：[FDTemplateLayoutCell](https://github.com/forkingdog/UITableView-FDTemplateLayoutCell/)。

### 1.2 异步绘制

我只在显示文本的控件上用到了异步绘制的功能，但效果很不错。我参考 ASDK 的原理，实现了一个简单的异步绘制控件。这块代码我单独提取出来，放到了这里：[YYAsyncLayer](https://github.com/ibireme/YYAsyncLayer)。YYAsyncLayer 是 CALayer 的子类，当它需要显示内容（比如调用了 [layer setNeedDisplay]）时，它会向 delegate，也就是 UIView 请求一个异步绘制的任务。在异步绘制时，Layer 会传递一个 BOOL(^isCancelled)() 这样的 block，绘制代码可以随时调用该 block 判断绘制任务是否已经被取消。

当 TableView 快速滑动时，会有大量异步绘制任务提交到后台线程去执行。但是有时滑动速度过快时，绘制任务还没有完成就可能已经被取消了。如果这时仍然继续绘制，就会造成大量的 CPU 资源浪费，甚至阻塞线程并造成后续的绘制任务迟迟无法完成。我的做法是尽量快速、提前判断当前绘制任务是否已经被取消；在绘制每一行文本前，我都会调用 isCancelled() 来进行判断，保证被取消的任务能及时退出，不至于影响后续操作。

目前有些第三方微博客户端（比如 VVebo、墨客等），使用了一种方式来避免高速滑动时 Cell 的绘制过程，相关实现见这个项目：[VVeboTableViewDemo](https://github.com/johnil/VVeboTableViewDemo)。它的原理是，当滑动时，松开手指后，立刻计算出滑动停止时 Cell 的位置，并预先绘制那个位置附近的几个 Cell，而忽略当前滑动中的 Cell。这个方法比较有技巧性，并且对于滑动性能来说提升也很大，唯一的缺点就是快速滑动中会出现大量空白内容。如果你不想实现比较麻烦的异步绘制但又想保证滑动的流畅性，这个技巧是个不错的选择。

### 1.3 异步图片加载

SDWebImage 在这个 Demo 里仍然会产生少量性能问题，并且有些地方不能满足我的需求，所以我自己实现了一个性能更高的图片加载库。在显示简单的单张图片时，利用 UIView.layer.contents 就足够了，没必要使用 UIImageView 带来额外的资源消耗，为此我在 CALayer 上添加了 setImageWithURL 等方法。除此之外，我还把图片解码等操作通过 YYDispatchQueuePool 进行管理，控制了 App 总线程数量。

### 2. 优化在 GPU层面

#### 2.1预渲染

对于 TableView 来说，Cell 内容的离屏渲染会带来较大的 GPU 消耗。在 Twitter Demo 中，我为了图省事儿用到了不少 layer 的圆角属性，你可以在低性能的设备（比如 iPad 3）上快速滑动一下这个列表，能感受到虽然列表并没有较大的卡顿，但是整体的平均帧数降了下来。用 Instument 查看时能够看到 GPU 已经满负荷运转，而 CPU 却比较清闲。为了避免离屏渲染，你应当尽量避免使用 layer 的 border、corner、shadow、mask 等技术，而尽量在后台线程预先绘制好对应内容。
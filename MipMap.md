# 开启MipMap的优缺点

优点：

- 会减少采样频率过低导致的画面失真
- 利用GPU缓存减少了频繁与显存的交互以降低显存带宽开销，增强性能。

缺点：

增加33%的磁盘、内存占用

# MipMap为什么会减小显存带宽

要将纹理绘制出来，纹理数据要经过这几个流程：硬盘->内存（system memory）->显存->GPU缓存->纹理运算单元

增加**Texture Cache**命中率。因为采样一次，实际上会把纹理这个采样位置周围的纹素数据都加载到缓存。距离屏幕近的场景，相邻两个屏幕像素所采样的纹素距离很近，都在同一个**Texture Cache**中。采样相邻纹素可以从缓存中取数据，缓存命中率高。距离屏幕远的场景，相邻两个屏幕像素所采样的纹素距离很大，采样相邻纹素无法从缓存中取数据，缓存命中率低。如果使用MipMap，则针对近景使用大纹理，远景使用小纹理，相邻两个屏幕像素所采样的纹素距离就会比较近。





- 我们传递给 GPU 一个带 Mipmap 的纹理，GPU 会在运行时通过(ddx, ddy)偏导选取合适的 Mipmap Level 的纹理。
- Mipmap 有利于节省带宽，并不是说我们传递给 GPU 的纹理数据变小的（相反是增加了）。而是最终渲染的时候相邻的像素更有可能在一个 CacheLine 里面，这就**提高了 Texture cache 的命中率。因为减少了对主存的交互，所以减少带宽**。
- 前面我们介绍 GPU 内存的时候有提到，当需要访问主存的时候，需要消耗几百个时钟周期。这会产生严重的 Stall。提升 Texture Cache 命中率就可以减少这种情况的出现。我们通过一些 GPU 性能分析工具优化游戏性能的时候，Texture L1/L2 Cache Missing 是一个非常重要的指标，通常要控制在一个很低的数值才是合理的。
- Mipmap 本身是会多消耗 1/3 的内存的（多了低级别的 mipmap 图），不过我们是可以决定纹理 Upload 给 GPU 的最高 mipmap level。我们通过引擎动态控制纹理的最高 mipmap level，反而可以有效的控制纹理的内存用量，这就是 Unity 引擎的 Texture Streaming 机制。**基于 Texture Streaming，纹理的内存总量是固定的，把不重要的纹理换出成高 level 的 mipmap 就可以减少纹理的内存占用。**当然如果重新切换到 mipmap0，可能会有纹理加载的过程，不过这个是引擎内部实现的，上层开发者是无感知的。我们看到很多 3D 游戏图片会有从模糊到清晰的过程，有可能就是 Texture Streaming 在起作用。
- 关于纹理的内存占用这里可以再做补充说明。前面介绍移动平台 GPU 内存的时候我们有提到，虽然 CPU 和 GPU 是共用一块儿物理内存，但是其内存空间是分离的。所以纹理提交给 GPU 是需要 Upload 的。当纹理 Upload 给 GPU 之后，CPU 端的纹理内存就会被释放掉。这种情况下，我们将显存中的纹理的内存释放掉，也就相当于释放掉纹理内存。
- 在 Unity 中，还有一部分纹理是需要 CPU 端读写数据的，或者编辑器下某个纹理导入选项中勾选了 Read/Write Enabled。对这些纹理而言，CPU 端的内存就不会被释放掉。此时该纹理就占用了两份内存，CPU 一份，GPU 一份。

参考连接：

- [GPU 渲染管线和硬件架构浅谈 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/2016951)









- [Why you really should be using mipmapping in your graphics applications - Imagination (imaginationtech.com)](https://blog.imaginationtech.com/why-you-really-should-be-using-mipmapping-in-your-graphics-applications/)
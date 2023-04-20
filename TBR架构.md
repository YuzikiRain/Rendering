# 名词解释

## Soc（System on Chip）

- Soc是把cpu、gpu、内存、通信基带、GPS模块等整合到一起的芯片
- 常见的

- - A系Soc（苹果）、M系Soc（Mac）
  - 骁龙Soc（高通）
  - 麒麟Soc（华为）
  - 联发科Soc
  - 猎户座Soc（三星）

- M系Soc的出现（暂用于Mac），说明手机、笔记本、pc的通用Soc已经出现了

## System Memory

- **System Memory**

- - 就是手机内存
  - 是Soc中cpu和gpu共用的一片LPDDR物理内存
  - 一般为几个G

- **On-Chip Memory**

- - 除了系统内存外，cpu和gpu还分别有自己的高速SRAM的Cache缓存，叫作On-Chip Memory
  - 一般为**几百K~几M**

- 访问不同距离的内存需要消耗的时间不同

- - 距离越近的时间消耗越低
  - **读取System Memory 的时间消耗大约是读取On-Chip Memory的几倍~几十倍**

- 补充：

  - 手机上，cpu和gpu共享一个内存地址空间

  - 桌面端两者的内存地址是分开的

## On-Chip Memory

- 在TBDR架构下，会**存储Tile的颜色、深度和模板缓冲**
- 其读写、修改速度都非常快

## Stall

- 当一个GPU核心的两次计算之间有依赖关系而必须串行时，等待的过程就是Stall

## FillRate

- 叫作像素填充率
- 计算方式

- 像素填充率 = ROP运行的时钟频率  × ROP的个数 × 每个时钟的ROP可以处理的像素个数

# IMR

## 渲染流程

- 对每个图元执行顶点着色器，然后进行剔除（比如背面剔除等），如果没有被剔除，则光栅化得到多个片元。
- 对（光栅化后得到的）多个片元执行片元着色器，再进行逐片元操作（Late Z Test、Alpha Blend等）。

## 示意图

![图片1](https://fastly.jsdelivr.net/gh/YuzikiRain/ImageBed/img/%E5%9B%BE%E7%89%871.png)

![图片3](https://fastly.jsdelivr.net/gh/YuzikiRain/ImageBed/img/%E5%9B%BE%E7%89%873.png)

## 伪代码

```
for draw in renderPass:
	for primitive in draw:
		for vertex in primitive:
			execute_vertex_shader(vertex)
		if primitive not culled:
			for fragment in primitive:
				execute_fragment_shader(fragment)
```

# TBR

TB(D)R（Tile-Based（Deferrded）Rendering），Tile指的是屏幕被分块（`16*16` or `32*32`）

TBR是目前主流移动GPU的渲染架构，对应一般PC上的GPU架构则为IMR

相比IMR的畅通无阻，sort简单（以高带宽为代价），TBR选择分割pipeline，使用各种defer、复杂的sort机制，降低了带宽开销（代价是defer导致的render rate降低）

#### ①核心目的

- 为了降低带宽，减少功耗，但渲染帧率上并不比IMR快

#### ②优点

- 给消除OverDraw提供了机会：

- - PowerVR有HSR技术，Mali有Forward Pixel Killing技术，都有为了最大限度减少被遮挡的pixel的texturing和shading

- 缓存友好（Cache friendly），在cache的读写速度要比全局内存中快得多（以降低帧率为代价，降低带宽、功耗）

#### ③缺点

- binning过程是在vertex阶段之后，将输出的数据写到系统内存（DDR）上，然后才能被fragment shader读取。

- - 这样一来**几何数据过多的管线，容易在此处有性能瓶颈**

- 如果某些三角形叠加在数个tile（块）上，会被绘制数次。

- - 这样就意味着：总渲染时间多于IMR

## TBR和TBDR

现在的手机基本都是TBDR架构了，但是受到提出TBDR的PowerVR公司的产权保护限制，大家都不敢说是TBDR架构。

- TBR：VS - Defer - RS - PS 
- TBDR：VS - Defer1 - RS - Defer2 - PS

defer1：延后图元的光栅化处理，所有的图元都生成后，才能开始划分tile所包含的图元。

defer2：延后（Defer）PS的执行，先进行各种剔除（Power的HSR、Mali的Forward Pixel Killing等），以避免执行不必要的PS计算与相关资源调用的带宽开销

### defer1不能避免吗？为什么不能一部分图元先光栅化处理？

因为并不知道图元属于哪个tile。

假设先处理了一个图元，写入到了on-chip memory，之后这部分数据不得不立即写入system memory（on-chip memory只够容纳一个tile的frame buffer数据（color、depth、stencil等）。

下一个图元可能也在同一个tile区域内，将同一tile区域内的depth数据从system memory读取回来是必须的，因为还要进行深度测试。这样一来还凭空多出了写回的开销，不如等到所有图元都生成完毕。

## 渲染流程

### 第一阶段

- binning（划分tile所包含的图元）：执行所有与几何相关的处理，并生成tile的图元列表（每个tile上有哪些图元），写入到System Memory上（因为所有图元的总数据量过大，on-chip memory放不下）
- 所有图元都处理完毕，才进入下一阶段。

### 第二阶段

- 处理一个Tile（或binning）：
  - 从System Memory上读取tile所包含的图元数据，然后执行光栅化，early z test（有HSR等优化手段），逐片元操作（Late Z Test等）
  - 然后将color、depth、stencil等都写入到on-chip memory
  - depth test、stencil test可以直接读取on-chip memory来实现，这都归功于每个tile都很小。
- 将color从Tile Buffer写回到System Memory中，而下一帧不需要用到前一帧的depth、stencil，因此不需要写回。
- 处理下一个Tile（或binning）

## 示意图

![图片5](https://fastly.jsdelivr.net/gh/YuzikiRain/ImageBed/img/%E5%9B%BE%E7%89%875.png)

![图片4](https://fastly.jsdelivr.net/gh/YuzikiRain/ImageBed/img/%E5%9B%BE%E7%89%874.png)

## 伪代码

```
# Pass one
for draw in renderPass:
	for primitive in draw:
		for vertex in primitive:
			execute_vertex_shader(vertex)
		if primitive not culled:
			append_tile_list(primitive)
# Pass two
for tile in renderPass:
	for primitive in tile:
		for fragment in primitive:
			execute_fragment_shader(fragment)
```




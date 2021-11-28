## XiangShan Cache 源码阅读

**蔡洛姗 2018K8009929051**

### 导言



### 一、XiangShan简介

#### 1.1 什么是XiangShan

​	香山是一款由中国科学院计算所和鹏城实验室联合发起的高性能开源RISC-V处理器项目。目前，香山处理器的第一版（雁栖湖）已经投片，第二版南湖还在开发，因此我在本次课程的大作业中主要阅读第一版的代码。.........

#### 1.2 什么是Chisel

​	Chisel将面向对象和函数式语言之类的软件工程优势引入数字系统设计中，以scala语言为基础，表示同步的数字电路，其代码本身是scala程序，在编译运行后生成verilog描述的电路。由于Chisel带来的一些敏捷设计的特征，香山的开发过程本身也是在验证高性能处理器是否能更加敏捷地开发。

......

模块 -> 类/对象

接线 -> Bundle

信号 -> 属性

#### 1.3 XiangShan架构与主要功能模块

​	“雁栖湖”架构是一个11级流水、6发射、4个访存部件的乱序处理器核，微架构图如下：

<img src="D:\学习\大四上\面向对象\XiangShan\香山-雁栖湖架构.jpg" alt="香山-雁栖湖架构" style="zoom:80%;" />

​	可以分为前端、后端、访存三个主要功能模块。前端包括取指、分支预测、指令缓冲等单元，顺序取指。后端包括译码、重命名、重排序缓冲、保留站、整型/浮点寄存器堆、整型/浮点运算单元。访存子系统按照`load`和`store`分割开，包括两条 `load` 流水线和两条 `store` 流水线，以及独立的 `Load Queue`和 `Store  Queue`，`Store Buffer`等，缓存包括`L1Cache(ICache、DCache)`、`L2Cache`、`TLB`和预取器等模块，在访存部件内。我主要选择`L1Cache`进行阅读，L1缓存包括指令缓存 ICache 和数据缓存 DCache，二者结构相似只是 DCache 的访问模式比 ICache 更加丰富，指令是只读不可写的且位宽固定，而数据可读可写且有字节访问、字访问和行访问等形式。

### 二、Cache机制

#### 2.1 什么是cache

​	缓存(`Cache`)技术是在处理器运算速度和内存访问速度差异日渐扩大导致的"存储墙"问题背景下被提出的，主要利用了程序访问内存的时间局部性和空间局部性，使用速度较快、容量较小的`Cache`来临时存放处理器常用的数据，从而提高数据访问效率。

​	`Cache`的主要功能是有序存放处理器常用的数据，响应来自上层的读/写请求，从自身或下层存储器中查找对应的数据块并及时替换。现代处理器普遍在片内集成多级Cache，香山也在每个处理器核中配备了一级指令Cache和数据Cache，二级Cache，以及多核共享的三级Cache。共享存储系统将会引入缓存一致性和总线协议等更加复杂的问题，而本文的重点是面向对象思想，所以我们选择相对简单的一级Cache，特别是同时具备读写响应的L1 DCache来避开项目自身的实现复杂性，只探究核心的需求建模和流程分析。

#### 2.2 DCache的基本结构

我们首先来看香山处理器的一个DCache模块形如：

```scala
class DCacheImp(outer: DCache) extends LazyModuleImp(outer) with HasDCacheParameters {
  val io = IO(new DCacheIO)
  val (bus, edge) = outer.clientNode.out.head
  
  //----------------------------------------
  // core data structures
  val dataArray = Module(new DuplicatedDataArray)
  val metaArray = Module(new DuplicatedMetaArray)

  //----------------------------------------
  // core modules
  val ldu = Seq.fill(LoadPipelineWidth) { Module(new LoadPipe) }
  val storeReplayUnit = Module(new StoreReplayQueue)
  val atomicsReplayUnit = Module(new AtomicsReplayEntry)
  val mainPipe   = Module(new MainPipe)
  val missQueue  = Module(new MissQueue(edge))
  val probeQueue = Module(new ProbeQueue(edge))
  val wb         = Module(new WritebackQueue(edge))

  //----------------------------------------
  // meta array read/write
  // data array read/write
  // load pipe connect
  // store pipe and store miss queue connect
  // atomics excecute
  // refill to load queue connect
  // mainPipe connect
  // wb connect
  /* ......
     ......
     ...... */
}
```

基于此，我们可以提取出香山DCache所需要的主要类和方法：

```
[用例名称]
    XiangShan L1DCache

[用例描述]
    1. XSCore的后端访存模块Memblock实现了DCache，使之与Load/Store Queue连接，队列向DCache输入访存请求
    2. pipe流水线接收请求并根据metaArray判断是否命中：
    	若命中则到dataArray中进行读写操作；
    	若不命中则将请求发送到missQueue，从下一级存储中读写；
    3. 将需要替换的数据块从WritebackQueue写回下一级存储，将新数据块写入dataArray和metaArray
    4. 向Load/Store Queue返回请求结果

[抽象提取类]    
[类]：DCacheIO
[属性]：与其它模块交互的接口信息

[类]：DuplicatedDataArray、DuplicatedMetaArray
[方法]：读操作、写操作、ECC校验
[属性]：配置规格、存储的数据、IO请求地址与数据

[类]：MainPipe、LoadPipe
[方法]：分析请求，查找和读写目标数据块、发出替换和写回脏块的请求
[属性]：配置信息、处理请求的状态机(miss/hit/req/resp)

[类]：MissQueue、WritebackQueue
[方法]：接收请求并适当合并、发送请求到其它模块
[属性]：队列元素、入队出队命令、处理请求的状态机
```



#### 2.3 DCache的请求处理流程

​	接下来我们进一步阅读代码并分析DCache具体如何基于上述类，响应一个来自CPU的读写请求。

##### 2.3.1 Load

​	从`DCacheImp`类的子类模块连接代码中，可以提取下面这些有关Load请求的操作：

```scala
  ldu(w).io.lsu <> io.lsu.load(w)
  val dataReadArb = Module(new Arbiter(new L1DataReadReq, DataReadPortCount))
  dataReadArb.io.in(LoadPipeDataReadPort)  <> ldu(LoadPipelineWidth - 1).io.data_read
  dataArray.io.read(LoadPipelineWidth - 1) <> dataReadArb.io.out
  dataArray.io.resp(LoadPipelineWidth - 1) <> ldu(LoadPipelineWidth - 1).io.data_resp
```

​	分析控制逻辑和数据通路我们可以得出，读请求首先从IO接口进入`LoadPipe`，`LoadPipe`类的代码较为复杂，我们只关注其三级流水来理解它在Dcache中所承担的功能，而不细究其实现，这也是面向对象里“封装”和“信息隐藏”思想的体现，对于自上而下进行代码阅读的读者来说，能够大大提高效率。

```scala
class LoadPipe(implicit p: Parameters) extends DCacheModule {
  ......
  // Pipeline
  // --------------------------------------------------------------------------------
  // stage 0
  val s0_valid = io.lsu.req.fire()
  val s0_req = io.lsu.req.bits
  val s0_fire = s0_valid && s1_ready
  
  // --------------------------------------------------------------------------------
  // stage 1
  val s1_valid = RegInit(false.B)
  val s1_req = RegEnable(s0_req, s0_fire)
  val s1_addr = io.lsu.s1_paddr
  val s1_fire = s1_valid && s2_ready
  s1_ready := !s1_valid || s1_fire

  // tag check
  val meta_resp = VecInit(io.meta_resp.map(r => getMeta(r).asTypeOf(new L1Metadata)))
  def wayMap[T <: Data](f: Int => T) = VecInit((0 until nWays).map(f))
  val s1_tag_eq_way = wayMap((w: Int) => meta_resp(w).tag === (get_tag(s1_addr))).asUInt
  val s1_tag_match_way = wayMap((w: Int) => s1_tag_eq_way(w) && meta_resp(w).coh.isValid()).asUInt
  val s1_tag_match = s1_tag_match_way.orR
  val s1_hit_meta = Mux(s1_tag_match, Mux1H(s1_tag_match_way, wayMap((w: Int) => meta_resp(w))), s1_fake_meta)

  // data read
  val data_read = io.data_read.bits
  data_read.addr := s1_addr
  data_read.way_en := s1_tag_match_way
  data_read.rmask := UIntToOH(get_row(s1_addr))
  io.data_read.valid := s1_fire && !s1_nack

  // --------------------------------------------------------------------------------
  // stage 2
  val s2_valid = RegInit(false.B)
  val s2_req = RegEnable(s1_req, s1_fire)
  val s2_addr = RegEnable(s1_addr, s1_fire)

  val s2_tag_match_way = RegEnable(s1_tag_match_way, s1_fire)
  val s2_tag_match = RegEnable(s1_tag_match, s1_fire)
  val s2_hit_meta = RegEnable(s1_hit_meta, s1_fire)
  val s2_hit = s2_tag_match && s2_has_permission && s2_hit_coh === s2_new_hit_coh

  // load data gen
  val s2_data_words = Wire(Vec(rowWords, UInt(encWordBits.W)))
  for (w <- 0 until rowWords) {
    s2_data_words(w) := s2_data(encWordBits * (w + 1) - 1, encWordBits * w)
  }
  val s2_word = s2_data_words(s2_word_idx)
  val s2_word_decoded = s2_word(wordBits - 1, 0)

  // send load miss to miss queue
  io.miss_req.valid := s2_valid && !s2_nack_hit && !s2_nack_data && !s2_hit
  io.miss_req.bits.addr := get_refill_addr(s2_addr)

  // send back response
  val resp = Wire(ValidIO(new DCacheWordResp))
  resp.valid := s2_valid
  resp.bits.data := s2_word_decoded
  io.lsu.resp.valid := resp.valid
  io.lsu.resp.bits := resp.bits
}
```

​	`LoadPipe`在`stage0`时根据地址`addr`发出`metaArray`和`dataArray`的读请求，在`stage1`时进行`tag`比较判断是否命中，`stage2`如果`hit`，则将读到的`data`返回；如果`miss`，则发送`miss`请求到`MissQueue`，等待请求进入`L2cache`读取整个`block`的数据，将对应数据返回。`MissQueue`还会向`MainPipe`发送`refill`请求，读出被替换的路，写入新的`tag`和`data`。如果被替换的路是`dirty`的，则将其从`WritebackQueue`写回`L2cache`。

##### 2.3.2 Store

​	同样从`DCacheImp`类中，我们可以找到`Store`请求的连接：	

```scala
  storeReplayUnit.io.lsu <> io.lsu.store
  val mainPipeReqArb = Module(new RRArbiter(new MainPipeReq, MainPipeReqPortCount))
  mainPipeReqArb.io.in(MissMainPipeReqPort)    <> missQueue.io.pipe_req
  mainPipeReqArb.io.in(StoreMainPipeReqPort)   <> storeReplayUnit.io.pipe_req
  dataArray.io.write <> mainPipe.io.data_write
  dataArray.io.resp(LoadPipelineWidth - 1) <> mainPipe.io.data_resp
  missQueue.io.pipe_resp         <> mainPipe.io.miss_resp
  storeReplayUnit.io.pipe_resp   <> mainPipe.io.store_resp
```

可以看出，写请求首先从IO接口进入一个`storeReplayQueue`，出队列后进入`mainPipe`。`mainPipe`拥有四级流水线，我们在此就不再展示其代码，其主要功能为处理`Refill`，`Probe`，`Store/AMO`请求，对于`Store`请求，`stage0`读取`Data`和`Meta`，`stage1`根据`tag`判断hit/miss，`stage2`选中数据，`stage3`如果命中则写`Data`和`Meta`，`stage4`如果不命中则发送`miss`请求到`MissQueue`。接下来就和读请求`miss`一样，从下一级读取`block`，写入新块，替换和写回旧块。这也就是cache中常用的写回+写分配的策略。



### 三、XiangShan Cache设计分析

​	在第二节中我们对`XiangShan DCache`的功能进行了需求建模和流程分析，找到了一部分承担主要任务的类，并探索了一个load/store请求被响应的具体流程，本节我们来继续分析XiangShan Cache还涉及哪些重要的类以及类间关系。

![XiangShan DCache类图](D:\学习\大四上\面向对象\大作业\XiangShan DCache类图.jpg)

​	上图体现了我们在第二节中分析过的一些类以及相关类之间的关系。类与类之间主要用到的关系有继承、实现、关联和组合关系：

​	紫色的`DCacheImp`类继承自`LazyModuleImp`类，这是一个从开源`rocket-chip`代码中引入的抽象类，定义了模块实现类的通用属性和方法；类似地，蓝色的`DuplicatedMetaArray, MainPipe, LoadPipe, MissQueue, WritebackQueue`等都继承了一个顶层定义的`DCacheModule`抽象类，它定义了Dcache所涉及的子模块通用的方法，但没有实现，而是由继承的子类来分别实现。这充分体现了面向对象的抽象和复用思想，将对象的共性特征封装在一个父类中，这样继承的子类只需要实现其特性即可，减少了冗余代码，提高了编程效率。

​	为了进一步解耦和便于扩展，我们在课上学过`Java`等语言提供了接口类，但`scala`语言没有接口类，在硬件设计中模块与模块之间却必须有输入输出接口，我不确定用java中的接口类与硬件中的接口相比较是否贴切，不过`Chisel`在此基础上提供了`Bundle`类，用于捆绑不同类型的信号，可以整体引用和分别访问，非常适合在各个模块中实现IO接口。例如上图黄色的`DCacheIO`和其父类`DCacheBundle`就是`Bundle`类，拥有req_valid, req_ready, req_addr, resp_valid, resp_ready, resp_data等信号，在`DCacheImp`中被实现后可以通过`io.req_addr`来访问；此外`DecoupledIO`方法可以反转输入输出，比如与DCache模块相连接的`Load_queue`模块在实现IO接口时就需要用`DecoupledIO(new DCacheIO)`来表示其接线方向与DCache相反，这与`verilog`中需要对每一个输入输出信号都单独定义`input`和`output`的方式相比显然便捷很多。

​	此外，在阅读代码的过程中我发现还有一种特殊的类`Parameters`，通常伴随上述主要的类一起出现，如：`class DCacheImp(outer: DCache) extends LazyModuleImp(outer) with HasDCacheParameters`。这里的with表示复合类型，Chisel支持多继承，但这种多继承并不会产生冲突问题，因为`LazyModuleImp`才是DCacheImp主要继承的父类，而`HasDCacheParameters`只负责表示DCache的参数设置，如Cache的`nSets, nWays, blockBytes, tagBits, wordBits`等基本参数。这和verilog中的宏定义类似，但将所有关于DCache的参数都封装在一起，可配置又便于引用，更加直观。

​	在这当中，面向对象思想的核心要素被体现得淋漓尽致，包括抽象、继承、关联和多态。

### 四、高级设计意图分析

硬件系统和软件系统的不同

面向对象与敏捷开发为硬件设计带来的革命，对性能与开发效率的影响。（从架构机理到实现机制，展示编译后的verilog，对编程过程和效果做一个评估）

面向对象编程：程序被组织成许多组相互协作的对象，每个对象代表某个类的一个实例，而类则属于一个通过继承关系形成的层次结构

硬件开发方式和开发环境的趋势评价



### 五、感想与总结

面向对象仍然体现的不够完全，代码风格还保留了verilog等硬件描述语言的特性，类的方法多数是直接在类中实现，没有封装为函数



### Reference


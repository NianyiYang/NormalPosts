## CPU

点击 Record 按钮，接着操作一下 App 然后点 Stop

Call Chart 以图形的方式呈现方法跟踪数据。其中

- 橙色：系统 API 调用
- 绿色：应用自有方法
- 蓝色：第三方、包括 Java 语言 API 调用

在函数上 右键 - jump to source 可以跳转到对应代码

### 应用启动时使用 CPU Profiler

1. 依次选择 Run > Edit Configurations。
2. 在 Profiling 标签中，勾选 Start recording CPU activity on startup，点击 Apply。
3. 选择 Run > Profile。
4. 完全启动后，点击 stop Record，出现结果数据，使用上述分析方式分析即可。



## Memory

![image-20200925163203214](/Users/nianyi.yang/Library/Application Support/typora-user-images/image-20200925163203214.png)

Allocation Tracking ，**务必选 Full**。默认是 Simple，它会对监测做优化，导致虚线不显示、划定区域内的对象数并非真实对象数等等，进而影响内存泄漏检测

由于 GC 回收机制 **不是** 对象一旦弃用就立即回收，因此你可能会看到虚线条一直增加。为了方便测试，需要即时点击垃圾回收按钮（主动回收，Allocation Tracking 左边的垃圾桶）

![image-20200925163628550](/Users/nianyi.yang/Library/Application Support/typora-user-images/image-20200925163628550.png)

Allocations：划定 **范围内创建** 的对象数

Deadllocation：划定 **范围内销毁** 的对象数

Total Count：当前此对象类型的总对象数

Shallow Size：当前此对象类型总共占用的内存，单位为字节
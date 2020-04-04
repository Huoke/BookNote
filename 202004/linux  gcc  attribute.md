name | message
-|-
_attribute__((error("message"))) | Declare that calling the marked function is an error.
__attribute__((warning("message"))) |	Declare that calling the marked function is suspect and should emit a warning.
__attribute__((deprecated)) | Declare that using the marked function, type, or variable is deprecated and will emit a warning.
__attribute__((const)) | Declare that the marked function is a pure function, only examining its arguments and returning a value without examining or changing anything else.
__attribute__((pure)) | Declare that the marked function is a pure function, with no side effects (although it may examine global state).
__attribute__((nonnull(n1, ...))) | Declare that the specified arguments (one-based) (or all arguments if no indexes are listed) should only be passed nonnull pointers.
__attribute__((noreturn)) | Declare that the marked function will not return (although it may throw).
__attribute__((hot)) | Hint that the marked function is "hot" and should be optimized more aggresively and/or placed near other "hot" functions (for cache locality).
__attribute__((cold)) | Hint that the marked function is "cold" and should be optimized for size, predicted as unlikely for branch prediction, and/or placed near other "cold" functions (so other functions can have improved cache locality).
__attribute__((warn_unused_result)) | Declare that the function's return value is important and should be warned about if ignored.

[参考：](http://blog.csdn.net/lcw_202/article/details/6226217)

__attribute__((hot))|__attribute__((cold))

# __attribute__((hot))
函数前面使用这个扩展，表示该函数会被经常调用到，在编译链接时要对其优化，或说是将它和其他同样热(hot)的函数放到一块，这样有利于缓存的存取。

# __attribute__((cold))
Hint that the marked function is "cold" and should be optimized for size, predicted as unlikely for branch prediction, and/or placed near other "cold" functions (so other functions can have improved cache locality).
函数前面使用这个扩展，表示该函数比较冷门，这样在分支预测机制里就不会对该函数进行预取，或说是将它和其他同样冷门(cold)的函数放到一块，这样它就很可能不会被放到缓存中来，而让更热门的指令放到缓存中。
  
# 分支预测：
在 http://www.groad.net/bbs/read.php?tid-1455.html 这里介绍了一级缓存、二级缓存以及指令预取的概念。虽然实现多个级别的缓存是帮助加快程序逻辑的执行速度的一个途径，但实际上仍然没有解决“跳转的”程序问题。如果程序采用很多不同的逻辑分支，那么使不同级别的缓存跟上分支的跳转几乎是不可能的事情，结果反而会造成在最后时刻有更多的从内存中访问指令码和数据元素的情况。

为了解决这个问题，IA-32 平台的处理器提出了 分支预测(branch prediction)的概念。

分支预测使用专门的算法试图预测接下来在程序分支中需要哪些指令码。

专门统计学算法和分析被引入用来确定指令码中最可能的执行路径。这条路径上的指令码会被预取并且加载到缓存中。

# 3 种分支预测技术(Pentium-4):
- 深度分支预测
- 动态数据流分析
- 推理性执行

深度分支预测使处理器能够试图越过程序中的多个分支对指令进行解码。这里同样使用了统计学的算法来预测程序在分支中最可能的执行路径。虽然这种技术有帮助，但是不是完全没错误，完全解决了跳转的问题。

动态数据流分析对处理器中的数据流进行统计学实时分析。被预测为程序流程必须经过的，但是指令指针还没有达到的指令被传递给乱序执行引擎。另外，在处理器等待与令一条指令相关的数据时，任何可以执行的指令都会被处理。

推理性执行处理器能够确定指令码分支中哪些不是立即就需要执行的，哪些“远距离”指令码以后有可能需要执行的，并且试图处理这些指令，这里同样使用了乱序执行引擎。

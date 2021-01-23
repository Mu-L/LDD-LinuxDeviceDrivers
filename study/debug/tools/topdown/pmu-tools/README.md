---

title: 使用 pmu-tools 进行 TopDown 分析
date: 2021-01-24 18:40
author: gatieme
tags: hexo
categories:
        - hexo
thumbnail: 
blogexcerpt: 这篇文章旨在帮助希望更好地分析其应用程序中性能瓶颈的人们. 有许多现有的方法可以进行性能分析, 但其中没有很多方法既健壮又正式. 而 TOPDOWN 则为大家进行软硬协同分析提供了无限可能. 本文通过 pmu-tools 入手帮助大家进行 TOPDOWN 分析.


---

| CSDN | GitHub | Hexo |
|:----:|:------:|:----:|
| [Aderstep--紫夜阑珊-青伶巷草](http://blog.csdn.net/gatieme) | [`AderXCoding/system/tools`](https://github.com/gatieme/AderXCoding/tree/master/system/tools) | [gatieme.github.io](https://gatieme.github.io) |

<br>

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议</a>进行许可, 转载请注明出处, 谢谢合作

因本人技术水平和知识面有限, 内容如有纰漏或者需要修正的地方, 欢迎大家指正, 鄙人在此谢谢啦

<br>

# 1 Topdown & PMU-TOOLS 简介
-------

## 1.1 TopDown 简介
-------


在这篇文章中, 我展示了进行性能分析的更正式的方法. 它称为自上而下的微体系结构分析方法(TMAM) ([<英特尔®64和IA-32架构优化参考手册> 附录B.1](https://software.intel.com/sites/default/files/managed/9e/bc/64-ia-32-architectures-optimization-manual.pdf)). 在这种方法论中, 我们尝试从高层组件(如前端, 后端, 退休, 分支预测器)开始检测导致执行停滞的原因, 并缩小性能低效的根源. 


进行 TOPDOWN 分析的过程通过是一个不断迭代优化的过程, 他的通常流程如下

1.  确定性能问题的类型;
    1.1.    我们可以先从高层次的划分中, 先粗粒度的当前性能问题主要问题是在哪个阶段(类型).
    1.2.    接着从低层次的划分中, 继续明确具体是哪块出的问题.

2.  使用精确事件 PEBS(X86)/SPE(ARM64) 在代码中找到确切的位置;
    2.1.    如果条件运行, 精确的抓取对应的 PMU 事件, 明确代码中出现问题的原因和位置.

3.  解决性能问题后, 重新开始 1.
    3.1.    修改代码, 解决性能问题, 然后重新进行 TOPDOWN 分析, 验证修改是否 OK, 看是否还有其他瓶颈或者是否引入其他问题.



## 1.2 TopDown 资料汇总
-------


Top-down Microarchitecture Analysis Method(TMAM)资料
之前介绍过TMAM的具体内容, 在这里对网络上相关的信息和资料做一个汇总：

*       国外资料

[Tuning Applications Using a Top-down Microarchitecture Analysis Method](https://software.intel.com/content/www/us/en/develop/documentation/vtune-cookbook/top/methodologies/top-down-microarchitecture-analysis-method.html)

[Top-down Microarchitecture Analysis through Linux perf and toplev tools](http://www.cs.technion.ac.il/~erangi/TMA_using_Linux_perf__Ahmad_Yasin.pdf)

[A Top-Down method for performance analysis and counters architecture](https://ieeexplore.ieee.org/document/6844459/metrics#metrics)

[Performance_Analysis_in_a_Nutshell](https://doc.itc.rwth-aachen.de/download/attachments/28344675/08.Performance_Analysis_in_a_Nutshell.pdf?version=1&modificationDate=1480665136000&api=v2)

[Top Down Analysis Never lost with Xeon® perf. counters](https://indico.cern.ch/event/280897/contributions/1628888/attachments/515367/711139/Top_Down_for_CERN_2nd_workshop_-_Ahmad_Yasin.pdf)

[How TMA* Addresses Challenges in Modern Servers and Enhancements Coming in IceLake](https://dyninst.github.io/scalable_tools_workshop/petascale2018/assets/slides/TMA%20addressing%20challenges%20in%20Icelake%20-%20Ahmad%20Yasin.pdf)

*       国内资料


[几句话说清楚15：Top-Down性能分析方法资料及Toplev使用](https://decodezp.github.io/2019/02/14/quickwords15-toplev)

当然还有一些其他的相关信息, 不过上面几个都可以覆盖


## 1.3 PMU-TOOLS 简介
-------

pmu-tools 是 Andi Kleen 构建的帮忙大家在 intel X86 环境上进行 TOPDOWN 分析的工具.


# 2 PMU-TOOLS 安装
-------

## 2.1 下载 github 仓库
-------

```cpp
git clone https://github.com/andikleen/pmu-tools
```



## 2.2 下载 PMU 表
-------

pmu-tools 完成的具体工作就是采集 PMU 数据, 并按照微架构既定的公式计算 topdown 数据, 但是不同的微架构和 CPU 所支持的 PMU 事件是不同的, 因此 pmu-tools提供了一套完成的映射表, 标记不同 CPU 所支持的 PMU 事件映射表, 以及 topdown 如何使用这些 PMU 数据去计算, 参见 [Intel 01.day perfmon 下载](https://download.01.org/perfmon).


pmu-tools 的工具在第一次运行的时会通过 `event_download.py` 把本机环境的 PMU 映射表自动下载下来, 但是前提是你的机器能正常连接 01.day 的网络. 很抱歉我司内部的服务器都是不行的, 因此 pmu-tools 也提供了手动下载的方式.

因此当我们的环境根本无法连接外部网络的时候, 我们只能通过其他机器下载实际目标环境的事件映射表下载到另一个系统上, 有点交叉编译的意思.

*   首先获取目标机器的 CPU 型号

```cpp
printf "GenuineIntel-6-%X\n" $(awk '/model\s+:/ { print $3 ; exit } ' /proc/cpuinfo )
```

>
> cpu的型号信息是由 vendor_id/cpu_family/model/stepping 等几个标记的.
>
> 他其实标记了当前 CPU 是哪个系列那一代的产品, 对应的就是其微架构以及版本信息. 
>
> 注意我们使用了 %X 按照 16 进制来打印
>
> 注意上面的命令显示制定了 vendor_id 等信息, 因为当前服务器端的 CPU 前面基本默认是 GenuineIntel-6 等.
>
> 不过如果我们是其他机器, 最好查看下 cpufino 信息确认.

比如我这边机器的 CPU 型号为 :

```cpp
processor       : 7
vendor_id       : GenuineIntel`
cpu family      : 6
model           : 85
model name      : Intel(R) Xeon(R) Gold 6161 CPU @ 2.20GHz
stepping        : 4
microcode       : 0x1
```

对应的结果就是 `GenuineIntel-6-55-4`.

我们也可以直接用 `-v` 打出来 CPU 信息.

```cpp
$ python ./event_download.py  -v

My CPU GenuineIntel-6-55-4
```

*   获取 PMU 映射表

如果你可以链接网络, 那么这个命令就会把你 `host` 机器上的 `PMU` 映射表下载下来. 否则你拿到了 `CPU` 型号, 那么你可以直接显式指定 `CPU` 型号来获取.

```cpp
$ python ./event_download.py GenuineIntel-6-55-4

Downloading https://download.01.org/perfmon/mapfile.csv to mapfile.csv
Downloading https://download.01.org/perfmon/SKX/skylakex_core_v1.24.json to GenuineIntel-6-55-4-core.json
Downloading https://download.01.org/perfmon/SKX/skylakex_matrix_v1.24.json to GenuineIntel-6-55-4-offcore.json
Downloading https://download.01.org/perfmon/SKX/skylakex_fp_arith_inst_v1.24.json to GenuineIntel-6-55-4-fp_arith_inst.json
Downloading https://download.01.org/perfmon/SKX/skylakex_uncore_v1.24.json to GenuineIntel-6-55-4-uncore.json
Downloading https://download.01.org/perfmon/SKX/skylakex_uncore_v1.24_experimental.json to GenuineIntel-6-55-4-uncoreexperimental.json
my event list /home/chengjian/.cache/pmu-events/GenuineIntel-6-55-4-core.json
```


最终 PMU 的映射表文件, 将被保存在 `~/.cache/pmu-events`. 将此目录打包后, 放到目标机器上解压即可.

当前我们也可以使用 `event_download.py -a` 下载所有可用的 PMU 事件映射表. 然后通过如下环境变量, 将 PMU 映射手动指向正确的文件.

```cpp
export EVENTMAP=~/.cache/pmu-events/GenuineIntel-6-55-4-core.json
export OFFCORE=~/.cache/pmu-events/GenuineIntel-6-55-4-offcore.json
export UNCORE=~/.cache/pmu-events/GenuineIntel-6-55-4-uncore.json
```

最后两个 `offcore/uncore` 的设置是可选的. 或者可以将名称设置为 CPU 标识符(GenuineIntel-FAM-MODEL), 这将覆盖当前 CPU.

另一种方法是显式设置目标机器的 PMU 映射文件存储路径( 注意是在相对于它的 pmu-events 目录中).

```cpp
export XDG_CACHE_DIR=/opt/cache
```

则将从 `/opt/cache/pmu-events` 中查找对应的映射文件.

# 3 PMU-TOOLS 使用
-------


## 3.1 toplev
-------

`toplev` 是一个基于 `perf` 和 `TMAM` 方法的应用性能分析工具. 从之前的介绍文章中可以了解到 `TMAM` 本质上是对 `CPU Performance Counter` 的整理和加工. 取得 `Performance Counter` 的读数需要 `perf` 来协助, 对读数的计算进而明确是 `Frondend bound` 还是 `Backend bound` 等等. 

在最终计算之前, 你大概需要做三件事: 

1.  明确 CPU 型号, 因为不同的 CPU, 对应的 PMU 也不一样
2.  读取 TMAM 需要的 perf event 读数
3.  按 TMAM 规定的算法计算, 具体算法在这个 Excel 表格里

这三步可以自动化地由程序来做. 本质上 `toplev` 就是在做这件事. 

toplev的Github地址: https://github.com/andikleen/pmu-tools

另外补充一下, TMAM作为一种Top-down方法, 它一定是分级的. 通过上一级的结果下钻, 最终定位性能瓶颈. 那么toplev在执行的时候, 也一定是包含这个“等级”概念的. 

下面是 `toplev` 使用方法的资料：

[toplev manual](https://github.com/andikleen/pmu-tools/wiki/toplev-manual)

[pmu-tools, part II: toplev](http://halobates.de/blog/p/262)

基本上都是由 toplev 的开发者自己写的, 可以作为一个 Quick Start Guide. 


# 4 参考资料
-------

[Top-Down performance analysis methodology](https://easyperf.net/blog/2019/02/09/Top-Down-performance-analysis-methodology)




<br>

*	本作品/博文 ( [AderStep-紫夜阑珊-青伶巷草 Copyright ©2013-2017](http://blog.csdn.net/gatieme) ), 由 [成坚(gatieme)](http://blog.csdn.net/gatieme) 创作.

*	采用<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议</a>进行许可. 欢迎转载、使用、重新发布, 但务必保留文章署名[成坚gatieme](http://blog.csdn.net/gatieme) ( 包含链接: http://blog.csdn.net/gatieme ), 不得用于商业目的. 

*	基于本文修改后的作品务必以相同的许可发布. 如有任何疑问, 请与我联系.

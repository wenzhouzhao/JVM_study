# Young GC 实操

>  我们平时系统运行创建的对象，除非是大对象，否则通常都是优先分配到新生代的 `Eden` 区域的；而且新生代还有另外两块 `Survivor` 区域，默认 `Eden` 区域是占据新生代的80%，每块 `Survivor` 占据新生代的10%。

## 1、开发环境

- **JDK 1.8 +**
- **IntelliJ IDEA ULTIMATE 2019.3 +** 

## 2、参数配置

```
-XX:NewSize=5242880
-XX:MaxNewSize=5242880
-XX:InitialHeapSize=10485760
-XX:MaxHeapSize=10485760
-XX:SurvivorRatio=8
-XX:PretenureSizeThreshold=10485760
-XX:+UseParNewGC
-XX:+UseConcMarkSweepGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-Xloggc:gc.log
```

### 2.1、名词释义

```
-XX:NewSize=5242880 ：初始新生代大小：5242880/1024/1024=5MB
-XX:MaxNewSize=5242880 ：最大新生代大小
-XX:InitialHeapSize=10485760 ：初始堆大小
-XX:MaxHeapSize=10485760 ：最大堆大小
-XX:SurvivorRatio=8 ：Eden:Survivor:Survivor的比例为8:1:1
-XX:PretenureSizeThreshold=10485760 ：指定了大对象阈值是10MB
-XX:+UseParNewGC ：新生代使用 ParNew 垃圾回收器
-XX:+UseConcMarkSweepGC ：老年代使用CMS 垃圾回收器
-XX:+PrintGCDetils ：打印详细的gc日志
-XX:+PrintGCTimeStamps ：打印出来每次GC发生的时间
-Xloggc:gc.log ：将gc日志写入一个磁盘文件
```
> 相当于给堆内存分配10MB内存空间，其中新生代是5MB内存空间，其中Eden区占4MB，每个Survivor区占0.5MB，大对象必须超过10MB才会直接进入老年代，新生代使用ParNew垃圾回收器，老年代使用CMS垃圾回收器

### 2.2、idea 参数配置位置

![img](https://gitee.com/BigYoungZhao/typora/raw/master/img/20210501135737.png)



## 3、实操代码



```java
public static void main(String[] args) {
    // 在JVM的Eden区内放入一个1MB的对象，
    // 同时在main线程的虚拟机栈中会压入一个main()方法的栈帧，
    // 在main()方法的栈帧内部，会有一个“b1”变量，
    // 这个变量是指向堆内存Eden区的那个1MB的数组
    byte[] b1 = new byte[1024*1024];
    // 在堆内存的Eden区中创建第二个数组，并且让局部变量指向第二个数组，
    // 然后第一个数组就没人引用了，此时第一个数组就成了没人引用的“垃圾对象”了
    b1 = new byte[1024*1024];
    // 在堆内存的Eden区内创建了第三个数组，同时让b1变量指向了第三个数组，
    // 此时前面两个数组都没有人引用了，就都成了垃圾对象
    b1 = new byte[1024*1024];
    // 让b1这个变量什么都不指向了，此时会导致之前创建的3个数组全部变成垃圾对象
    b1 = null;
    // 会分配一个2MB大小的数组，尝试放入Eden区中
    // Eden区总共就4MB大小，而且里面已经放入了3个1MB的数组了，所以剩余空间只有1MB了，此时放一个2MB的数组是放不下的
    // 此时会触发Young GC
    byte[] b2 = new byte[2*1024*1024];
}
```
第1行代码
`byte[] b1 = new byte[1024*1024];`
这行代码一旦运行，就会在JVM的Eden区内放入一个 1MB 的对象，同时在 main 线程的虚拟机栈中会压入一个 main() 方法的栈帧，在 main() 方法的栈帧内部，会有一个“b1”变量，这个变量是指向堆内存 Eden 区的那个1MB的数组，如下图
![img](https://gitee.com/BigYoungZhao/typora/raw/master/img/20210501135454.png)



第2行代码
`b1 = new byte[1024*1024];`
此时会在堆内存的 Eden 区中创建第二个数组，并且让局部变量指向第二个数组，然后第一个数组就没人引用了，此时第一个数组就成了没人引用的“垃圾对象”了，如下图所示

![image-20210501140147413](https://gitee.com/BigYoungZhao/typora/raw/master/img/20210501140149.png)

第3行代码

`b1 = new byte[1024*1024];`

这行代码在堆内存的 Eden 区内创建了第三个数组，同时让 b1 变量指向了第三个数组，此时前面两个数组都没有人引用了，就都成了垃圾对象，如下图所示  

![image-20210501140428518](https://gitee.com/BigYoungZhao/typora/raw/master/img/20210501140429.png)



第4行代码

`b1 = null;`

这行代码一执行，就让 b1 这个变量什么都不指向了，此时会导致之前创建的 3 个数组全部变成垃圾对象，如下图  

![image-20210501140542207](https://gitee.com/BigYoungZhao/typora/raw/master/img/20210501140543.png)

第5行代码

`byte[] b2 = new byte[2*1024*1024];`

此时会分配一个 2MB 大小的数组，尝试放入 Eden 区中，明显是不行的，因为 Eden 区总共就 4MB 大小，而且里面已经放入了 3 个 1MB 的数组了，所以剩余空间只有 1MB 了，此时你放一个 2MB 的数组是放不下的，所以这个时候就会触发新生代的Young GC。  

## 4、解析log日志
```
Java HotSpot(TM) 64-Bit Server VM (25.181-b13) for windows-amd64 JRE (1.8.0_181-b13), built on Jul  7 2018 04:01:33 by "java_re" with MS VC++ 10.0 (VS2010)
Memory: 4k page, physical 16612312k(7908952k free), swap 24246156k(8332112k free)
CommandLine flags: -XX:InitialHeapSize=10485760 -XX:MaxHeapSize=10485760 -XX:MaxNewSize=5242880 -XX:NewSize=5242880 -XX:OldPLABSize=16 -XX:PretenureSizeThreshold=10485760 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:SurvivorRatio=8 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:-UseLargePagesIndividualAllocation -XX:+UseParNewGC 
0.376: [GC (Allocation Failure) 0.376: [ParNew: 4096K->510K(4608K), 0.0034817 secs] 4096K->1356K(9728K), 0.0036514 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
0.405: [GC (Allocation Failure) 0.405: [ParNew: 3998K->363K(4608K), 0.0012098 secs] 4844K->1692K(9728K), 0.0012603 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 par new generation   total 4608K, used 2494K [0x00000000ff600000, 0x00000000ffb00000, 0x00000000ffb00000)
  eden space 4096K,  52% used [0x00000000ff600000, 0x00000000ff814930, 0x00000000ffa00000)
  from space 512K,  71% used [0x00000000ffa00000, 0x00000000ffa5af78, 0x00000000ffa80000)
  to   space 512K,   0% used [0x00000000ffa80000, 0x00000000ffa80000, 0x00000000ffb00000)
 concurrent mark-sweep generation total 5120K, used 1328K [0x00000000ffb00000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 3312K, capacity 4556K, committed 4864K, reserved 1056768K
  class space    used 350K, capacity 392K, committed 512K, reserved 1048576K
```

### 4.1、 CommandLine flag……
看 log 第3行
`CommandLine flags: -XX:InitialHeapSize=10485760 -XX:MaxHeapSize=10485760 -XX:MaxNewSize=5242880 -XX:NewSize=5242880 -XX:OldPLABSize=16 `
表示这次运行程序采取的JVM参数是什么。  

### 4.2、 Allocation Failure……
`0.376: [GC (Allocation Failure) 0.376: [ParNew: 4096K->510K(4608K), 0.0034817 secs] 4096K->1356K(9728K), 0.0036514 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] `
表示这次GC的执行情况。
因为要分配一个 2MB 的数组，结果 Eden 区内存不足，所以就出现了 "Allocation Failure" ,即对象分配内存失败，所以要触发一次 `Young GC`。
这次GC是什么时候发生的呢？ "0.376" 表示在系统运行后的多少秒发生了本次 GC ，这里表示大概在系统运行后的 300 多毫秒发生了本次 GC 。

### 4.3、 ParNew: 4096K……
`ParNew: 4096K->510K(4608K), 0.0034817 secs`
表示触发的是新生代的 `Young GC` ，用的是我们指定的 ParNew 垃圾回收器来执行 GC 的。

`4096K->510K(4608K)`
表示新生代可用空间是 4608K ，也就是 4.5MB （Eden 区是 4MB ，两个 Survivor 中只有一个是可以放存活对象的，另外一个是必须一致保持空闲的，所以他考虑新生代的可用空间，就是 Eden +1个 Survivor 的大小）。

然后 4096K->510K ，意思就是对新生代执行了一次 GC ， GC 之前都使用了 4096KB 了，但是GC之后只有 510KB 的对象是存活下来的。

### 4.4、 0.0034817 secs
这个就是本次 GC 耗费的时间，本次大概耗费了 3.5ms，回收 3.5MB 的对象。
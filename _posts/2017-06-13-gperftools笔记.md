# Gperftools笔记

gperftools是Google提供的一套性能分析工具，主要四个组件，Tcmalloc、Heap profiler、
Heap checker和Cpu profiler。Tcmalloc比glibc的malloc更快的内存管理库，heap-checker专门检测内存泄漏，heap-profiler则是内存监控器，cpu profiler是用来分析函数的cpu执行时间。

## 1、Centos7安装gperftools

<pre>
yum install autoconf automake libtool git -y
git clone https://github.com/gperftools/gperftools.git
cd gperftools
./autogen.sh
./configure
make
make install
</pre>

Graphviz是一个由AT&T实验室启动的开源工具包，用于绘制DOT语言脚本描述的图形，gperftools依靠此工具生成图形分析结果
<pre>
yum install graphviz ghostscript
</pre>

## 2、TCMALLOC

比glibc的malloc更快的内存管理库。  
优点：  
（1）ptmalloc2能在300ns执行一个malloc和free对，而TcMalloc能在50ns内执行一个malloc和free对。  
（2）TcMalloc可以减少多线程程序之间的锁争用问题，在小对象上能达到零争用。  
（3）TcMalloc为每个线程单独分配一个线程本地的Cache，少量的地址分配就直接从Cache中分配，并且定期做垃圾回收，将线程本地Cache中的空闲内存返回给全局控制堆。  
（4）TcMalloc认为小于（<=）32K为小对象，大对象直接从全局控制堆上以页（4K）为单位进行分配，也就是说大对象总是页对齐的。  
（5）TcMalloc中一个页可以存入一些相同大小的小对象，小对象从本地内存链表中分配，大对象从中心内存堆中分配。  
使用方法：  
将libtcmalloc.so/libtcmalloc.a链接到程序中，或者设置LD_PRELOAD=libtcmalloc.so。这样就可以使用tcmalloc库中的函数替换掉操作系统的malloc、free、realloc、strdup内存管理函数。  

## 3、HEAP PROFILER与HEAP CHECKER

heap-checker专门检测内存泄漏，heap-profiler则是内存监控器，可以随时知道当前内存使用情况（程序中内存使用热点），当然也能检测内存泄漏。工作中一般是使用heap-profiler，实时监控程序的内存使用情况。    
简单的使用方法：    
```c++
#include <cstdio>
#include <cstdlib>
int* fun(int n)
{
 	   int *p1=new int[n];
 	   int *p2=new int[n];
 	   return p2;
}
int main()
{
  	  int n;
  	  scanf("%d",&n);
  	  int *p=fun(n);
   	  delete [] p;
  	  return 0;
}
```

编译：
<pre>
g++ -O0 -g test_heap_checker.cpp -ltcmalloc -o test_heap_checker
</pre>

运行：

<pre>
env HEAPPROFILE=./mybin.hprof ./test_heap_checker
</pre>

注意：按照代码的意思应该是生成.hprof文件，但是自己结果只生成了一个heap文件（mybin.hprof.0001.heap），但是并不影响后面的操作  

<pre>
[root@aboss Heap_Profiler]# pprof --text test_heap_checker ./mybin.hprof.0001.heap
Using local file test_heap_checker.
Using local file ./mybin.hprof.0001.heap.
Total: 0.1 MB
     0.1 100.0% 100.0%      0.1 100.0% fun
     0.0   0.0% 100.0%      0.1 100.0% __libc_start_main
     0.0   0.0% 100.0%      0.1 100.0% _start
     0.0   0.0% 100.0%      0.1 100.0% main说明：
</pre>

(1) 第一列表示直接使用了多少内存，单位为MB  
(2) 第四列包含了该方法自己占用的以及所有他调用所占用的内存  
(3) 第二列和第五列分别是第一列和第四列的百分比表示  
(4) 第三列是第二列的一个综合  
另外，看pprof的参数，可以生成gif或者pdf文件：  

<pre>
pprof --pdf ./test_heap_checker ./mybin.hprof.0001.heap > a.pdf
</pre>

内存泄露分析：  

## 4、CPU PROFILER

用于分析函数的CPU执行时间  
对于一个函数的CPU使用时间分析，分为两个部分：  
(1)整个函数消耗的CPU时间，包括函数内部其他函数调用所消耗的CPU时间  
(2)不包含内部其他函数调用所消耗的CPU时间（内联函数除外）  
使用方法：  
```c++
#include <gperftools/profiler.h>
#include <iostream>
using namespace std;
void func1() {
    int i = 0;
    while (i < 100000) {
        ++i;
    }  
}
void func2() {
    int i = 0;
    while (i < 200000) {
        ++i;
    }  
}
void func3() {
    for (int i = 0; i < 1000; ++i) {
        func1();
        func2();
    }  
}
int main(){
    ProfilerStart("my.prof"); // 指定所生成的profile文件名
    func3();
    ProfilerStop(); // 结束profiling
    return 0;
}
```

编译  

<pre>
g++ -o cpu_profile2 cpu_profile2.cpp -lprofiler -lunwind
</pre>

备注：64位操作系统同时链接libunwind库，即-lunwind  
具体操作：  
<pre>
yum install libunwind
ln -s /usr/lib64/libunwind.so.8 /usr/lib64/libunwind.so
g++ -o cpu_profile2 cpu_profile2.cpp -lprofiler -lunwind
</pre>

运行程序
<pre>
[root@aboss cpu_profiler2]# ./cpu_profile2
PROFILE: interrupts/evictions/bytes = 57/10/960
</pre>

查看分析结果
<pre>
[root@aboss cpu_profiler2]# pprof --text ./cpu_profile2 my.prof
Using local file ./cpu_profile2.
Using local file my.prof.
Total: 57 samples
      34  59.6%  59.6%       34  59.6% func2
      23  40.4% 100.0%       23  40.4% func1
       0   0.0% 100.0%       57 100.0% __libc_start_main
       0   0.0% 100.0%       57 100.0% _start
       0   0.0% 100.0%       57 100.0% func3
       0   0.0% 100.0%       57 100.0% main
</pre>

关于文本风格输出结果   
第1列分析样本数量（不包含其他函数调用）  
第2列分析样本百分比（不包含其他函数调用）  
第3列目前为止的分析样本百分比（不包含其他函数调用）  
第4列分析样本数量（包含其他函数调用）  
第5列分析样本百分比（包含其他函数调用）  
第6列表示函数名  
也可以以图片方式查看  

<pre>
pprof --pdf ./cpu_profile2 my.prof >cpu_profile2.pdf
</pre>

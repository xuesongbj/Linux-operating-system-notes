# Linux 内存管理

## 摘要
* slab、kmalloc/kfree、/proc/slabinfo和slabtop
* malloc/free、kmalloc/kfree与Buddy关系
* mallocopt
* vmalloc
* Linux lazy行为
* OOM(内存耗尽)、oom_score和oom_adj
* Android 进程生命周期与OOM



### Buddy算法问题
&emsp;&emsp;&emsp; 通过第一部分学习,咱们知道内存所有的ZONE都通过Buddy算法进行管理。在buddy算法中空闲物理内存会被分为11个组，其中第0,1，N个组分别对应2^0、2^n个连续物理界面。当需要申请的内存小于1页(4K),Buddy算法也会分配1页(4K)。此时再使用Buddy算法进行分配粒度太大了,对内存是一种极大的浪费。

* 
![buddy](imgs/buddy_2.png "buddy")

## Slab
&emsp;&emsp;&emsp; Buddy的最小单位是页(4k), 无论是内核还是用户程序都会申请一些更小的内存。所以在在Linux中对heap(堆)内存进行了二次管理,从Buddy拿到的内存(1K,4K,8K....)再次进行分割管理,这种就是Slab。


### slab原理
&emsp;&emsp;&emsp; 例如,申请8byte内存。Linux 内核首先会从Buddy申请到1页(4K)内存,然后将4K分成多个8byte,每一个8byte被称为Slab的一个object。
<br>
&emsp;&emsp;&emsp; Slab机制的实现算法有slab、slub、slob。每种算法的实现请参考[slab](imgs/slaballocators.pdf)。

&emsp;&emsp;&emsp; 备注: Slab英文"大块"的意思。就像在超市中买一大块牛肉,然后再进行切割,当牛肉被吃完后,还是需要去超市买一大块牛肉。Slab也是这样的原理,每次从Buddy申请一大块儿内存,然后分割成一个个object提供给内核或应用程序使用。

* 
![slab](imgs/slab_2_1.png "slab")

### Linux slab使用
&emsp;&emsp;&emsp; Linux会针对一些小粒度内存的申请以及一些常规数据结构内存做slab,可以查看slab具体的分配情况(cat /proc/slabinfo):

* 
![slabinfo](imgs/slabinfo_2.png "slabinfo")

* 
![slabinfo](imgs/slabinfo_2_1.png "slabinfo")

<br>
&emsp;&emsp;&emsp; slab主要分为两类:

&emsp; &emsp; &emsp; 1. 第一类是内核里常用的数据结构,如TCPv6,UDPv6等。由于内核经常要申请和释放这类数据结构,所以就真对这些常用的数据结构分别做了slab,再次申请这类数据结构时就从所属的slab进行申请一个object(使用kmem_cache_alloc()函数申请)。
<br>
&emsp; &emsp; &emsp; 2. 第二类是一些小粒度的内存申请,如slabinfo中的kmalloc-16,kmalloc-32,kmalloc-64...(使用kmalloc申请)。

<br>
&emsp; &emsp; &emsp; 注意:不同数据类型申请内存时需要向当前所属数据类型的slab缓存进行申请,不能向其它数据类型slab缓存申请。而且slab只针对内核空间,跟用户空间没有关系。用户空间malloc/free 调用的libc库。

### Buddy 和Slab区别
&emsp; &emsp; &emsp; buddy将内存条当成一个池子进行管理(按页进行管理),slab从buddy拿到的内存当成一个池子进行管理。buddy和slab这两个都是内存分配器。slab管理的是来自Buddy的内存,而buddy管理的是Zone的内存。


### 用户态malloc/free、内核态kmalloc/kfree 与Buddy关系
&emsp; &emsp; &emsp; 所有的内存申请最终都来自于Buddy,但malloc/free和kmalloc/kfree都不是直接从Buddy申请和释放内存。而是从libc和slab二级分配器(由该二级分配器向Buddy申请和释放内存)申请和释放内存。
<br>
&emsp; &emsp; &emsp; slab和Buddy都是内存分配器,slab内存来自Buddy。slab和Buddy在算法级别上是对等的。
<br>
&emsp; &emsp; &emsp; malloc/free不是系统调用,它们是C语言libc库函数,与内核没有一一映射调用关系。malloc/free对内存操作真对的是libc(free()释放1M内存,并不会进行系统调用将内存还给buddy。malloc()也不会进行系统调用向buddy申请内存,可能会直接使用刚free掉的1M内存);同理,内核态kmalloc/kfree和Buddy不存在一一对应关系,而是从slab池操作。
<br>
*
![slab](imgs/slab_2.png "slab")

### malloopt
当累计到多少free之后才还给内核

### 明天开搞...
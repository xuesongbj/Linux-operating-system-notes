# I/O

## Linux I/O架构

![Architecture-of-Linux-Kernel-I-O-stack](./Architecture-of-Linux-Kernel-I-O-stack.png)

&nbsp;

### 文件系统block与磁盘sector关系

![io](./io.jpg)

* Block(块)： 文件系统上的概念，一般文件系统block大小为4KB。
* Sector(扇区)：HDD磁盘最小单元，一般为512字节。

Block(块)一般与Sector(扇区)值是不相等的。
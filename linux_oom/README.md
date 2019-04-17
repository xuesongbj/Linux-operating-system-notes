# Linux OOM Killer

## OOM Killer定义
OOM Killer或Out Of Memory killer是Linux内核在系统内存严重不足时才会被启动,选择性的Kill一些进程,以释放一些内存。具体的记录日志在/var/log/message中,如果出现Out of memory,说明系统曾将出现过OOM。


### 内存过量使用应对策略

通过/proc/sys/vm/overcommit_memory可以控制进程对内存过量泗洪的应对策略。

* overcommit_memory = 0,允许进程轻微过量使用内存,但对于大量过载请求则不允许,也就是当内存消耗过大就会触发OOM killer。
* overcommit_memory = 1,永远允许进程overcommit,不会触发OOM killer。
* overcommit_memory = 2,永远禁止进程overcommit,不会触发OOM killer。


### OOM Killer触发
如果触发了OOM机制,系统会杀掉某些进程。


* 什么进程会被杀掉？
	
	通过查看/proc/[pid]/oom_adj权重大小(-17~15范围,具体需要查看内核确认),越高的权重,意味着有可能被oom killer掉,-17标示禁止被kill。
	
	
#### 查看进程权重范围
oom_adj和oom_score是oom_killer的主要参考值。

##### 方式1
 * uname -a 查看内核版本。
 * 进入/usr/src/kernels/内核版本/include/linux/oom.h确认具体的权重范围。
 

#### 方式2
  查看/proc/[pid]/oom_score值是当前进程score分数,越高的score,意味着可能被kill,这个数值根据oom_adj运算(2^n,n就是oom_adj的值)后的结果。
  
  
### 防止出发OOM killer

#### 修改oom_score_adj

```
ps -ef|gerp + pid
echo -17 > /proc/PID/oom_score_adj
```

#### 关闭OOM机制
如果不启动OOM机制,内存使用过大,会让系统产生很多异常数据。

```
echo "vm.panic_on_oom=1" >> /etc/sysctl.conf
systcal -p
```
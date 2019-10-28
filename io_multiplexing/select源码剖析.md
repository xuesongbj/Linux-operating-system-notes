# select 源码剖析(基于Linux X86_64平台)
select最终是通过底层驱动对应设备文件的poll函数来查询是否有可用资源(可读或者可写)，如果没有则睡眠。

### fd_set
fd_set是一个文件描述符fd的集合，由于每个进程可打开的文件描述符默认值为1024，fd_set可记录的fd个数上限也是为1024个。 fd_set采用位图bitmap算法。

```
#define __FD_SETSIZE    1024
#define FD_SETSIZE      __FD_SETSIZE    // 最大文件描述符
typedef long int        __fd_mask;

typedef struct
{
#ifdef __USE_XOPEN
    // __fd_mask ==> 8byte
    // fds_bits[__FD_SETSIZE / __NFBITS] ==> 8byte[16] ==> (8*8)bit[16] => 1024
    __fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
# define __FDS_BITS(set) ((set)->fds_bits)
#else
    __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
# define __FDS_BITS(set) ((set)->__fds_bits)
#endif
} fd_set;
```

### select 源码剖析

#### select主要工作
select主要工作分为3部分:

* 将需要监控的用户空间inp(可读)、outp(可写)、exp(异常)事件拷贝到内核空间fds的in、out、ex;
* 执行do_select()方法,将in、out、ex监控到的事件结果写入到res_in、res_out、res_ex;
* 将内核空间fds的res_in、res_out、res_ex事件结果信息拷贝回用户空间inp、outp、exp。

```
SYSCALL_DEFINE5(select, int, n, fd_set __user *, inp, fd_set __user *, outp,
		fd_set __user *, exp, struct timeval __user *, tvp)
{
	return kern_select(n, inp, outp, exp, tvp);
}
```

```
static int kern_select(int n, fd_set __user *inp, fd_set __user *outp,
		       fd_set __user *exp, struct timeval __user *tvp)
{
	struct timespec64 end_time, *to = NULL;
	struct timeval tv;
	int ret;

    // 设置超时阈值
    // ...略...
    
	ret = core_sys_select(n, inp, outp, exp, to);
	return poll_select_finish(&end_time, tvp, PT_TIMEVAL, ret);
}
```

```
int core_sys_select(int n, fd_set __user *inp, fd_set __user *outp,
			   fd_set __user *exp, struct timespec64 *end_time)
{
	fd_set_bits fds;
	void *bits;
	int ret, max_fds;
	size_t size, alloc_size;
	struct fdtable *fdt;

    // 小内存在stack上分配, 256Byte
	long stack_fds[SELECT_STACK_ALLOC/sizeof(long)];

	
    // select可监控个数小于等于进程可打开的文件描述上限
	rcu_read_lock();
	fdt = files_fdtable(current->files);
	max_fds = fdt->max_fds;
	rcu_read_unlock();
	if (n > max_fds)
		n = max_fds;

	size = FDS_BYTES(n);
	bits = stack_fds;

    // 根据n来计算需要多少个字节, 展开为size=4*(n+32-1)/32
    // 如果大于stack_fds大小,则在heap上分配内存
	if (size > sizeof(stack_fds) / 6) {
		alloc_size = 6 * size;
		bits = kvmalloc(alloc_size, GFP_KERNEL);
	}

    // 需要6个位图,int/out/ex以及其对应的3个结果集
	fds.in      = bits;
	fds.out     = bits +   size;
	fds.ex      = bits + 2*size;
	fds.res_in  = bits + 3*size;
	fds.res_out = bits + 4*size;
	fds.res_ex  = bits + 5*size;

    // 将用户空间的inp、outp、exp拷贝到内核空间fds的in、out、ex
	if ((ret = get_fd_set(n, inp, fds.in)) ||
	    (ret = get_fd_set(n, outp, fds.out)) ||
	    (ret = get_fd_set(n, exp, fds.ex)))
		goto out;
    
    // 将fds的res_in、res_out、res_ex内容清零
	zero_fd_set(n, fds.res_in);
	zero_fd_set(n, fds.res_out);
	zero_fd_set(n, fds.res_ex);
    
    // 核心算法
	ret = do_select(n, &fds, end_time);

    // 将fds的res_in、res_out、res_ex结果拷贝到用户空间inp、outp、exp
	if (set_fd_set(n, inp, fds.res_in) ||
	    set_fd_set(n, outp, fds.res_out) ||
	    set_fd_set(n, exp, fds.res_ex))
		ret = -EFAULT;

out:
    kfree(bits);
out_nofds:
    return ret;
}
```

##### fdset相关操作方法
```
// 记录可读、可写、异常的输入和输出结果信息
typedef struct {
	unsigned long *in, *out, *ex;
	unsigned long *res_in, *res_out, *res_ex;
} fd_set_bits;

// 将用户空间的ufdset拷贝到内核空间fdset
static inline
int get_fd_set(unsigned long nr, void __user *ufdset, unsigned long *fdset)
{
	nr = FDS_BYTES(nr);
	if (ufdset)
        // 用户空间拷贝到内核空间
		return copy_from_user(fdset, ufdset, nr) ? -EFAULT : 0;

    // 没有指定ufdset,则对fdset内存空间清零
	memset(fdset, 0, nr);
	return 0;
}

// 将fdset内存空间清零
static inline
void zero_fd_set(unsigned long nr, unsigned long *fdset)
{
	memset(fdset, 0, FDS_BYTES(nr));
}

// 将内核空间fdset内存空间拷贝到用户空间ufdest
static inline unsigned long __must_check
set_fd_set(unsigned long nr, void __user *ufdset, unsigned long *fdset)
{
	if (ufdset)
		return __copy_to_user(ufdset, fdset, FDS_BYTES(nr));
	return 0;
}
```

##### poll_wqueues
当调用select()时,当前进程就会创建一个struct poll_weueues结构体,用来统一辅佐实现这个进程中所有待监测的fd的轮询工作,后面所有的工作和都这个结构体有关,所以它非常重要。
```
struct poll_wqueues {
	poll_table pt;
	struct poll_table_page *table;

    // 保存当前调用select的用户进程struct task_struct结构体
	struct task_struct *polling_task; 

    // 当前用户进程被唤醒后置成1，以免该进程接着进睡眠
	int triggered;

    // 错误码
	int error;

	int inline_index;
	
    struct poll_table_entry inline_entries[N_INLINE_POLL_ENTRIES];
};
```

##### poll_table
将等待队列添加到poll_table.

```
typedef struct poll_table_struct {
	poll_queue_proc _qproc;
	unsigned long _key;
} poll_table;
```

##### poll_table_page
申请的物理页都会将起始地址强制转换成该结构体指针。
```
struct poll_table_page {
	// 指向下一个申请的物理页
    struct poll_table_page * next;

    // 指向entries[]中首个待分配(空的) poll_table_entry地址
	struct poll_table_entry * entry;

    // 该page页后面剩余的空间都是待分配的
	struct poll_table_entry entries[0];
};
```

##### poll_table_entry
该结构体是用来连接驱动和应用进程的关键结构体。这个结构体中内嵌了一个等待队列项wait_queue_t，和一个等待队列头 wait_queue_head_t，它就是驱动程序中定义的等待队列头，应用进程就是在这里保存了每一个硬件设备驱动程序中的等待队列头(当然每一个fd都有一个poll_table_entry结构体)。
```
struct poll_table_entry {
    struct file *filp;
    unsigned long key;
    wait_queue_t wait;                  // wait等待队列项
    wait_queue_head_t *wait_address;    // wait的等待队列头
};
```

##### do_select
select核心算法,do_select最核心的还是调用文件系统*f_op->poll函数,来检测I/O事件(比如fd可读或可写)。

* 当存在被监控的fd触发目标事件,则将fd记录下来，退出循环体，返回用户空间。
* 当没有找到目标事件，如果已超时或者有待处理的信号，也会退出循环体，返回Null给用户空间。
* 当以上两种情况都不满足,则会让当前进程进入休眠状态，以等待fd或者超时定时器来唤醒自己,再循环一次。

```
static int do_select(int n, fd_set_bits *fds, struct timespec64 *end_time)
{
	ktime_t expire, *to = NULL;
	struct poll_wqueues table;
	poll_table *wait;
	int retval, i, timed_out = 0;
	u64 slack = 0;
	__poll_t busy_flag = net_busy_loop_on() ? POLL_BUSY_LOOP : 0;
	unsigned long busy_start = 0;

    // 找出最大文件描述符
    // 将n设置成最大值+1
	retval = max_select_fd(n, fds);
	n = retval;

    // 初始化等待队列
	poll_initwait(&table);
	wait = &table.pt;
	if (end_time && !end_time->tv_sec && !end_time->tv_nsec) {
		wait->_qproc = NULL;
		timed_out = 1;
	}

	if (end_time && !timed_out)
        // 处理超时
		slack = select_estimate_accuracy(end_time);

	retval = 0;
	for (;;) {
		unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;
		bool can_busy_loop = false;

		inp = fds->in; outp = fds->out; exp = fds->ex;
		rinp = fds->res_in; routp = fds->res_out; rexp = fds->res_ex;

		for (i = 0; i < n; ++rinp, ++routp, ++rexp) {
			unsigned long in, out, ex, all_bits, bit = 1, j;
			unsigned long res_in = 0, res_out = 0, res_ex = 0;
			__poll_t mask;

            // 所有注册事件
			in = *inp++; out = *outp++; ex = *exp++;
			all_bits = in | out | ex;
			if (all_bits == 0) {
                // BITS_PRE_LONG,long型数据中bit位的个数。
                // 以32|64bits步长遍历位图,直到在该区间存在目标fd。
				i += BITS_PER_LONG;
				continue;
			}

			for (j = 0; j < BITS_PER_LONG; ++j, ++i, bit <<= 1) {
				struct fd f;
                // 找到目标fd
				f = fdget(i);
				if (f.file) {
                    const struct file_operations *f_op;

                    // 检查集合
					wait_key_set(wait, in, out, bit,
						     busy_flag);
                    // 调用poll_wait函数,检查文件设备状态(检查IO事件),并且将当前进程加入到设备等待队列,返回掩码
					mask = vfs_poll(f.file, wait);

					fdput(f);

                    // 写入in/out/ex相对应的结果
					if ((mask & POLLIN_SET) && (in & bit)) {
						res_in |= bit;
						retval++;
						wait->_qproc = NULL;
					}
					if ((mask & POLLOUT_SET) && (out & bit)) {
						res_out |= bit;
						retval++;
						wait->_qproc = NULL;
					}
					if ((mask & POLLEX_SET) && (ex & bit)) {
						res_ex |= bit;
						retval++;
						wait->_qproc = NULL;
					}
					
                    // 返回值不为0,则停止循环
					if (retval) {
						can_busy_loop = false;
						busy_flag = 0;
					} else if (busy_flag & mask)
						can_busy_loop = true;
				}
			}

            // 本轮循环遍历完成,则更新fd事件结果
			if (res_in)
				*rinp = res_in;
			if (res_out)
				*routp = res_out;
			if (res_ex)
				*rexp = res_ex;
            // 让出CPU时间片让其它进程运行
			cond_resched();
		}
		wait->_qproc = NULL;
        
        // 当有文件描述符准备就绪、超时、有待处理的信号,则退出循环
		if (retval || timed_out || signal_pending(current))
			break;

		if (end_time && !to) {
            // 首轮循环设置超时时间
			expire = timespec64_to_ktime(*end_time);
			to = &expire;
		}

        // 设置当前进程状态为TASK_INTERRUPTIBLE，进入休眠直到超时。
		if (!poll_schedule_timeout(&table, TASK_INTERRUPTIBLE,
					   to, slack))
			timed_out = 1;
	}

    // 释放poll等待队列
	poll_freewait(&table);
	return retval;
}
```

##### vs_poll 
```
static inline __poll_t vfs_poll(struct file *file, struct poll_table_struct *pt)
{
    // 轮训函数不为空,每当设备模块加载就自动会加载设备轮训函数,
    // 等于将轮回函数给poll指针,以便调用。
	if (unlikely(!file->f_op->poll))
		return DEFAULT_POLLMASK;
	return file->f_op->poll(file, pt);
}
```

##### poll_initwait
poll_initwait函数主要初始化poll_wqueues。
```
void poll_initwait(struct poll_wqueues *pwq)
{
    // 初始化poll函数指针
	init_poll_funcptr(&pwq->pt, __pollwait);

    // 将当前进程记录在pwq结构体
	pwq->polling_task = current;

	pwq->triggered = 0;
	pwq->error = 0;
	pwq->table = NULL;
	pwq->inline_index = 0;
}

// EXPORT_SYMBOL标签内定义的函数或者符号对全部内核代码公开，不用修改内核代码就可以在内核模块中直接调用。
EXPORT_SYMBOL(poll_initwait);
```
将结构体poll_wqueues->poll_table->poll_queue_proc赋值给__pollwait,__pollwait会在poll过程中调用。

```
static inline void init_poll_funcptr(poll_table *pt, poll_queue_proc qproc)
{
	pt->_qproc = qproc;
	pt->_key   = ~0UL; /* all events enabled */
}

typedef void (*poll_queue_proc)(struct file *, wait_queue_head_t *, struct poll_table_struct *);
```
#### file_operations->poll
struct file_operations设备驱动操作函数，每个文件系统都有自己的一套文件操作集合。

```
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*aio_read) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
	ssize_t (*aio_write) (struct kiocb *, const struct iovec *, unsigned long, loff_t);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iterate) (struct file *, struct dir_context *);

    // 轮训方法
	unsigned int (*poll) (struct file *, struct poll_table_struct *);
	
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	void (*mremap)(struct file *, struct vm_area_struct *);
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*aio_fsync) (struct kiocb *, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
};
```
(*f_op->poll)(f.file, wait),就是调用文件系统的poll方法,不同驱动设备实现方法略有不同,但都会执行poll_wait(),该方法真正执行就是回掉函数__pollwait,把自己挂入等待队列。

##### pollwait
pollwait()和poll_freewait()完成所有的工作。
```
static void __pollwait(struct file *filp, wait_queue_head_t *wait_address,
				poll_table *p)
{
    // 根据poll_wqueues的成员pt指针p找到所在的poll_wqueue结构指针
	struct poll_wqueues *pwq = container_of(p, struct poll_wqueues, pt);
	struct poll_table_entry *entry = poll_get_entry(pwq);
	if (!entry)
		return;

	entry->filp = get_file(filp);
	entry->wait_address = wait_address;
	entry->key = p->_key;

    // 设置entry->wait.func = pollwake
	init_waitqueue_func_entry(&entry->wait, pollwake);

    // 设置private内容为pwq
	entry->wait.private = pwq;

    // 将该pollwake加入到等待链表头
	add_wait_queue(wait_address, &entry->wait);
}

# include/linux/wait.h
static inline void
init_waitqueue_func_entry(wait_queue_t *q, wait_queue_func_t func)
{
	q->flags	= 0;
	q->private	= NULL;

    // 唤醒回调函数
	q->func		= func;
}
```
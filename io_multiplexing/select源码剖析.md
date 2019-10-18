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

    // 小参数内存在stack上分配
    // 32Byte
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
}
```
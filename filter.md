# net/core/filter.c

## 1. bpf_prog_create(struct bpf_prog **pfp, struct sock_fprog_kern *fprog);

### 入参：

```markdown
struct bpf_prog {
    u16 pages;
    u16 ...; //一堆flag
    enum bpf_prog_type type; //kprobe, tracepoint, perf_event, raw_tracepoint, socket_filter, xdp, cgroup_skb, cgroup_sock，以及其他，分别放在kernel不同的位置
    enum bpf_attach_type expected_attach_type; //BPF_CGROUP_INET_INGRESS等等
    u32 len;
    u32 jited_len;
    u8 tag[BPF_TAG_SIZE];
    struct bpf_prog_aux *aux; //辅助功能，如性能统计
    struct sock_fprog_kern *orig_prog;
    unsigned int (*bpf_func)(void *ctx, const struct bpf_insn *insn);
    union {
        struct sock_filter insns[0];
        struct bpf_insn insnsi[0];
    } //bpf指令
};

struct sock_fprog_kern {
    u16 len;
    struct sock_filter *filter;    
};

struct sock_filter {
    __u16 code;
    __u8 jt;
    __u8 jf;
    __u32 k;
}; //defined in include/uapi/linux/filter.h
```
### 功能：

根据传入的cBPF程序格式的filter程序（fprog），建立bpf程序，保存在pfp指向的位置。

### 实现：

1. 计算所需空间，调用bpf_prog_alloc，返回指针。

2. 复制filter中信息到新分配的bpf_prog中。

3. 调用bpf_prepare_filter。
	1. 检查cBPF程序合法性（无越界指针，非法指令，以ret结尾）。
	2. 尝试bpf_jit_compile该bpf程序（事实上并不作任何事情）。
	3. 如果无法jit编译该bpf程序，调用bpf_migrate_filter，复制原cBPF程序到栈。
		1. 备份原bpf程序。
		2. 第一次调用bpf_convert_filter。过一遍原cBPF指令，计算新eBPF程序长度，调用bpf_prog_realloc重新分配prog空间。
		3. 第二次调用bpf_convert_filter，将原cBPF程序转换成eBPF存入bpf_prog。
		4. 最后进入bpf_prog_select_runtime。
			1. 通过bpf_prog_select_func根据新的eBPF程序的栈深度选择不同的解释器以供JIT无效时使用。
			2. 如果dev_bound，下发BPF程序至smartNIC。
			3. 如果非dev_bound，尝试JIT。

创建BPF程序完毕。

BPF解释器：共16个，深度分别为32：32：512。解释器栈上分配不同深度的BPF栈和寄存器空间，调用__bpf_prog_run。__bpf_prog_run内含256个BPF指令的列表和256个包含汇编的标签，根据指令跳转至标签，执行汇编，循环直到结束。

BPF JIT：
1. 通过bpf_prog_alloc_jited_linfo分配额外JIT所需内存放在aux下。
2. bpf_int_jit_compile。
	1. bpf_jit_blind_contants。
		1. 克隆bpf_prog。
		2. bpf_jit_bind_insn，对克隆出来的BPF机器码进行了一系列神奇的随机加密操作。
		3. bpf_patch_insn_single把修改过的克隆机器码填回原代码。
	2. do_jit翻译成机器码并精简。循环直到无法精简。保存生成的二进制。

### 调用地点：

#### drivers/net/ppp/pppgeneric.c :

1. ppp_init中调用register_chrdev注册ppp_device_fops，其中字段unlocked_ioctl指向ppp_ioctl。
2. 当ppp_ioctl参数cmd为PPPIOCSPASS或PPPIOCSACTIVE时，调用bpf_prog_create创建filter，挂上ppp（由入参file直接转换而来）->pass_filter或者active_filter。
3. pass_filter和active_filter都会在ppp_send_frame，ppp_receive_nonmp_frame中使用BPF_PROG_RUN运行，返回值为0则前者通过，后者标记时间。

#### drivers/net/team/team_mode_loadbalance.c：

1. 在lb_init中，调用team_options_register注册lb_options，其中一个成员标明为BPF并封装了lb_bpf_func_get和lb_bpf_func_set。
2. 在lb_bpf_func_set中调用__fprog_create，复制传入数据至fprog的filter字段，再调用bpf_prog_create创建bpf_prog，于fprog一并保存至team包含的lb_priv。
3. 在lb_transmit中，该BPF程序被用来计算skb的哈希值。

#### net/core/ptp_classifier.c (ptp_classifier_init)：

初始化bpf_prog并且注册在全局变量上。该bpf_prog在ptp_classify_raw中被调用。

## bpf_prog_create_from_user(struct bpf_prog **pfp, struct sock_fprog *fprog, bpf_aux_classic_check_t trans, bool save_orig);

### 入参：

与bpf_prog_create类似，多了trans与save_orig，且提供的filter程序来自用户。
```markdown
typedef int (*bpf_aux_classic_check_t)(struct sock_filter *filter, unsigned int flen);
```

### 功能：

与bpf_prog_create几乎相同。多了trans额外的检查和保存用户提供的filter程序save_orig选项。

### 实现：

与bpf_prog_create几乎相同。

如果save_orig选项为真，额外分配空间保存用户filter程序，存至产生bpf_prog结构的orig_prog。

## bpf_prog_destroy(struct bpf_prog *fp);

### 入参：

### 功能：

释放bpf_program

### 实现：

如果bpf_program是socket_filter，进入bpf_prog_put。

如果不是，调用bpf_release_orig_filter和bpf_prog_free。

socket_filter情况的内部实现倒是挺复杂的。有空了再看。

## sk_attach_filter(struct sock_fprog *fprog, struct sock *sk);

### 入参：

与bpf_prog_create类似，多了attach的目标套接字。

### 功能：

从filter程序建立bpf_prog，并连接到套接字。

### 实现：

先调用__get_filter创建bpf_prog，过程与bpf_prog_create_from_user类似，多了对套接字的检查，接着调用_sk_attach_prog，把上一步产生的bpf_prog封装成sk_filter（多了refcount），放到sk->sk_filter。

## struct bpf_redirect_info;

```markdown
struct bpf_redirect_info {
    u32 flags;
    u32 tgt_index;
    void *tgt_value;
    struct bpf_map *map;
    struct bpf_map *map_to_flush;
    u32 kern_flags;
};
```

## xdp_do_flush_map(void);

### 功能：

flush目前全局bpf_redirect_info指向map，可为devmap，cpumap或xskmap。

## xdp_do_redirect(struct net_device *dev, struct xdp_buff *xdp, struct bpf_prog *xdp_prog);

### 入参：

```markdown
struct xdp_buff {
    void *data;
    void *data_end;
    void *data_meta;
    void *data_hard_start;
    unsigned long handle;
    struct xdp_rxq_info *rxq;
};

struct xdp_rxq_info {
    struct net_device *dev;
    u32 queue_index;
    u32 reg_state;
    struct xdp_mem_info mem;
} ____cacheline_aligned;

struct xdp_mem_info {
    u32 type;
    u32 id;
};

struct xdp_frame {
    void *data;
    u16 len;
    u16 headroom;
    u16 metasize;
    struct xdp_mem_info mem;
    struct net_device *dev_rx;
};

struct xdp_sock {
    struct sock sk;
    struct xsk_queue *rx;
    struct net_device *dev;
    struct xdp_umem *umem;
    struct list_head flush_node;
    u16 queue_id;
    bool zc;
    enum {
        XSK_READY = 0,
        XSK_BOUND,
        XSK_UNBOUND,
    } state;
    struct mutex mutex;
    struct xsk_queue *tx ___cacheline_aligned_in_smp;
    struct list_head list;
    spinlock_t tx_completion_lock;
    spinlock_t rx_lock;
    u64 rx_dropped;
};

struct xdp_umem {
    struct xsk_queue *fq;
    struct xsk_queue *cq;
    struct xdp+umem_page *pages;
    u64 chunk_mask;
    u64 size;
    u32 headroom;
    u32 chunk_size_nohr;
    struct user_struct *user;
    unsigned long addresses;
    refcount_t users;
    struct work_struct work;
    struct page **pgs;
    u32 npgs;
    int id;
    struct net_device *dev;
    struct xdp_umem_fq_reuse *fq_reuse;
    u16 queue_id;
    bool zc;
    spinlock_t xsk_list_lock;
    struct list_head xsk_list;
}

struct xsk_queue {
    u64 chunk_mask;
    u64 size;
    u32 ring_mask;
    u32 nentries;
    u32 prod_head;
    u32 prod_tail;
    u32 cons_head;
    u32 cons_tail;
    struct xdp_ring *ring;
    u64 invalid_descs;
};

struct xdp_ring {
    u32 producer ___cacheline_aligned_in_smp;
    u32 consumer ___cacheline_aligned_in_smp;
};
```

### 功能：

把xdp指向的信息送进网卡。

### 实现：

从全局bpf_redirect_info获得目前的map。

如果map被设置，调用xdp_do_redirect_map，如map_to_flush非空，调用xdp_do_flush_map清空，确保无map_to_flush为空后调用__bpf_tx_xdp_map根据map类型调用函数。

cpumap与devmap调用不同的enqueue函数，结构相似，先由入参xdp_buff设置xdp_frame（直接写在xdp_buff开头，如零拷贝从buddy分配新页），再调用bq_enqueue将xdp_frame放入目标bulk_queue，满了则调用ndo_xdp_xmit清空队列中的xdp包，不满则挂上flush_list表示该设备尚有任务。

xskmap的将会进去xsk_rcv，然后根据是否zc进入__xsk_rcv_zc或者__xsk_rcv。两个函数最终都会调用xskq_produce_batch_desc，这个函数将会把得到的地址放置在xdp_sock直接指向的ring的某个位置以供发送。如果zero_copy，传进去的地址来自xdp_buff指向的handle。如果非zc，传入的地址来自xdp_sock指向umem指向xsk_queue指向ring中的某个节点，节点含有数据表示xdp_sock成员umem的一部分，此节点本身不含数据，拷贝自xdp_buff的data。

如果map未被设置，进入慢路径，从来源设备获得相关的net，再由全局的index和net获得目的设备，之后进入__bpf_tx_xdp，将xdp_buff转化为xdp_frame后调用与设备相关的ndo_xdp_xmit方法发送。

也就是说，使用dev_map和cpu_map产生的主要性能提升来自批量发送vs单个发送。

## xdp_do_generic_redirect(struct net_device *dev, struct sk_buff *skb, struct xdp_buff *xdp, struct bpf_prog *xdp_prog);

### 入参：

与xdp_do_redirect类似，差别是此时已产生了sk_buff，调用该函数地点已在协议栈中。

### 功能：

同上

### 实现：

该函数的实现也分为存在或不存在map。

存在map的情况，与map相关的发送函数会被分别调用（cpumap的该函数尚未实现），且xdp参数被彻底无视了。devmap中，dev_map_generic_redirect会被调用，经过一系列调用最后用net_device上注册的ndo_start_xmit送skb。xskmap中，调用xsk_generic_rcv把数据从xdp中拷贝至xsk_sock的xsk_queue中，通知xsk发送，再调用consume_skb将skb释放（不是很懂这个逻辑）。

不存在map时，获得目标net_device，调用generic_xdp_tx，讲真与devmap的实现好像没什么差别。唯一的差别就是devmap的skb目标设备来自全局ri->tgt_value->dev，而无map是目标设备来自dev_get_by_index_rcu(dev_net(dev), ri->tgt_index)。后者需要遍历net中所有ifindex寻找匹配net_device。

xdp_prog貌似只在trace中起了作用。

## int sk_detach_filter(struct sock *sk);

### 入参：

### 功能：

detach filter

### 实现：

设置sock的filter位置为NULL并调用__bpf_prog_release。算是bpf_prog_destroy的封装。

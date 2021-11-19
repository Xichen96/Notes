# Device Initialization

## Kernel boot sequence

When kernel boots up, it starts to initialize a bunch of subsystems. Eventually it calls `do_initcalls()`, which initializes subsystems as they are registered on a leveled linked list. `net_dev_init()` is then called.

### `dev_proc_init()`

This function calls `dev_proc_net_init()` or `dev_mc_net_init()` on every `struct net` to initialize network related procfs entries, and corresponding `dev_proc_net_exit()` or `dev_mc_net_exit()` is scheduled on exit.

### `netdev_kobject_init()`

This function register functions for procfs and sysfs (under /sys/class/net/DEVICE/..) operations.

### `netdev_init()`

This function is called on every `struct net` to initialize to basic state.

### initialize `struct softnet_data`

```markdown
struct softnet_data {
	struct list_head	poll_list;
	struct sk_buff_head	process_queue;
	/* stats */
	unsigned int		processed;
	unsigned int		time_squeeze;
	unsigned int		received_rps;
#ifdef CONFIG_RPS
	struct softnet_data	*rps_ipi_list;
#endif
#ifdef CONFIG_NET_FLOW_LIMIT
	struct sd_flow_limit __rcu *flow_limit;
#endif
	struct Qdisc		*output_queue;
	struct Qdisc		**output_queue_tailp;
	struct sk_buff		*completion_queue;
#ifdef CONFIG_XFRM_OFFLOAD
	struct sk_buff_head	xfrm_backlog;
#endif
	/* written and read only by owning cpu: */
	struct {
		u16 recursion;
		u8  more;
	} xmit;
#ifdef CONFIG_RPS
	/* input_queue_head should be written by cpu owning this struct,
	 * and only read by other cpus. Worth using a cache line.
	 */
	unsigned int		input_queue_head ____cacheline_aligned_in_smp;
	/* Elements below can be accessed between CPUs for RPS/RFS */
	call_single_data_t	csd ____cacheline_aligned_in_smp;
	struct softnet_data	*rps_ipi_next;
	unsigned int		cpu;
	unsigned int		input_queue_tail;
#endif
	unsigned int		dropped;
	struct sk_buff_head	input_pkt_queue;
	struct napi_struct	backlog;
};
```

For every CPU, initialize workqueue for future call to `flush_backlog()`, which walks `input_pkt_queue` and `process_queue` and call `dev_kfree_sk_irq()` on every linked skb. Then initialize `struct softnet_data *sd`. Then initilize IPI struct `csd` for future call to `rps_trigger_softirq()`, which calls `___napi_schedule()` to invoke an napi poll on said CPU. For details regarding napi, see napi.md. It is used by RPS (Receive Packet Steering, software version of Receive Side Scaling, direct packets to specific CPUs). Finally `struct napi_struct` is initialized. GRO hash is initialized and `poll` is set to `process_backlog()` and `weight` is set to 64.

### initialize loopback device and default exit

`loopback_net_init()` is called on all `struct net`. For details on loopback devices, see loopback.md. `default_default_exit()` is scheduled on exit for all `struct net`.

### register softirq

`net_tx_action()` and `net_rx_action()` is register for softirq `NET_TX_SOFTIRQ` and `NET_RX_SOFTIRQ`.

### register callback for CPU state `CPUHP_NET_DEV_DEAD`

Register callback `dev_cpu_dead()` to be triggered when working CPU is in state `CPUHP_NET_DEV_DEAD`. The function will link dead CPU's `struct softnet_data`'s send queues to target CPU's, call `NET_TX_SOFTIRQ`, and then receive on receive queue skbs.

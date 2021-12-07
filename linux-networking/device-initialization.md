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

## Driver initialization

Now the kernel is ready for device registration. Below is how the Intel E1000 driver sets up a net device. Starting with `module_init(e1000_init_module)`.

### `pci_register_driver()`

`pci_register_driver()` is called on `struct pci_driver e1000_driver`.

```markdown
struct pci_driver {
	struct list_head	node;
	const char		*name;
	const struct pci_device_id *id_table;	/* Must be non-NULL for probe to be called */
	int  (*probe)(struct pci_dev *dev, const struct pci_device_id *id);	/* New device inserted */
	void (*remove)(struct pci_dev *dev);	/* Device removed (NULL if not a hot-plug capable driver) */
	int  (*suspend)(struct pci_dev *dev, pm_message_t state);	/* Device suspended */
	int  (*resume)(struct pci_dev *dev);	/* Device woken up */
	void (*shutdown)(struct pci_dev *dev);
	int  (*sriov_configure)(struct pci_dev *dev, int num_vfs); /* On PF */
	int  (*sriov_set_msix_vec_count)(struct pci_dev *vf, int msix_vec_count); /* On PF */
	u32  (*sriov_get_vf_total_msix)(struct pci_dev *pf);
	const struct pci_error_handlers *err_handler;
	const struct attribute_group **groups;
	const struct attribute_group **dev_groups;
	struct device_driver	driver;
	struct pci_dynids	dynids;
};

struct device_driver {
	const char		*name;
	struct bus_type		*bus;
	struct module		*owner;
	const char		*mod_name;	/* used for built-in modules */
	bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */
	enum probe_type probe_type;
	const struct of_device_id	*of_match_table;
	const struct acpi_device_id	*acpi_match_table;
	int (*probe) (struct device *dev);
	void (*sync_state)(struct device *dev);
	int (*remove) (struct device *dev);
	void (*shutdown) (struct device *dev);
	int (*suspend) (struct device *dev, pm_message_t state);
	int (*resume) (struct device *dev);
	const struct attribute_group **groups;
	const struct attribute_group **dev_groups;
	const struct dev_pm_ops *pm;
	void (*coredump) (struct device *dev);
	struct driver_private *p;
};
```

After some pci specific setup, `driver_register()` is called, which in turn calls `bus_add_driver()`. `struct driver_private` will be initialized, which contains a list of devices that uses this driver. The driver's related sysfs entry will be set up.

`driver_attach()` calls `bus_for_each_device` to iterate over all devices connected on the bus and attempts adding the device with `__driver_attach()`. A `match` function registered under `struct bus_type` will be called to check for the match between device and driver, in this case the `match` is `pci_bus_match()`, which then calls `pci_match_device()` to check that the device id is in the approved list. If matched, `__driver_attach()` will call `driver_probe_device()` to bind device and driver. `pinctl_bind_pins()` is called to bind device to pinctrl sybsystem. Bus `dma_configure()` is called to set up dma, which, for pci device, is `pci_dma_configure()`, which in turn calls either `of_dma_configure()` (for Open Firmware device) or `acpi_dma_configure()` (for acpi device). `driver_sysfs_add()` will add the sysfs entries for the driver. `call_driver_probe()` will invoke bus specific probe function `pci_device_probe()` and if failed, driver specific probe `e1000_probe()`, which sets up and initializes net device. `device_add_groups()` will create sysfs entries for the device. Then driver and device will be officially bound, and bus notifier will notify that the device is bound.

After driver registered devices, `module_add_driver()` will add entries under /sys/module for the driver module. 

### `e1000_probe()`

This is the driver specific setup routine for E1000. Some standard pci hardware setup is done on the device. Then a new `struct net_device` is created with `alloc_etherdev()` with the size of a E1000 specific `struct e1000_adaptor`. `alloc_etherdev()` is a wrapper around `alloc_netdev_mqs()` with a ethernet specific `ether_setup()`.

The initialization of `struct net_device` starts at the initialization of red-black trees of addresses. Then the device will be connected to the initial `struct net`. All the queues are initialized (most notably `struct netdev_queue`, which contains `struct Qdisc` and `struct xsk_buff_pool`, and `struct netdev_rx_queue`, which contains `struct xdp_rxq_info` and RPS information), then `ether_setup()`, `default_ethtool_ops` and netfilter hooks ingress and egress.

`e1000_adaptor` is the private data portion of `struct net_device`. Memory mapped io is set up by standard pci functions. Then `e1000_init_hw_struct()` is called to initialize e1000 hardware, mac related operations. Then `netdev_ops` field is set to `e1000_netdev_ops`, which we will investigate further, and `ethtool_ops` is set to `e1000_ethtool_ops`. `netif_napi_add()` will initialize the `struct napi` embedded in `struct net_device` and register a driver specific poll function, which is
`e1000_clean()`. `e1000_init_sw_struct()` is called to create a ring of `struct e1000_rx_ring` in private data, disable interrupt for the NIC device and set private states. Some very hardware specific setting is done and flushed.

Several work queues and delayed work queues are initialized. `watchdog_task` is initialized to use `e1000_watch()`, `phy_info_task` to `e1000_update_phy_info_task()`, `reset_task` to `e1000_reset_task()`.

Eventually, the NIC is ready to accept traffic and `register_netdev()` is called to make the device officially available in the system.

### `register_netdev()`

`register_netdev()` is a `rtnl_lock` held version of `register_netdevice()`. Through `netdev_ops`, driver specific `ndo_init()` is called, which is nonexistent for E1000. A ifindex is allocated for the new network device. After some settings are checked and synced, message of device up `NETDEV_POST_INIT` is notified to the device's net and global net via `call_netdevice_notifiers()`. No function is registered by the driver yet, but devmap, fib and others might use it. The device entry in sysfs will be created. Then feature related device ops will be invoked if present, depending on the feature. `dev_init_scheduler()` will set up qdisc, tx, rx queue and the watchdog, which has `dev_watchdog()` registered. Device registration is notified.

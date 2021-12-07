# Configuration

Now the network device is registered in the system, but it is still not functioning because it is lacking some needed configuration. It has to be initialized from userspace in order to start working.

The entry for most network related setting is `dev_ioctl()`.

## Open device

There are many userspace tools that control and configure net devices (ifupdown, netplan, "/etc/network/interfaces"). A device is not active right after initialization.

After dispensed by `dev_ioctl()`, a device up request will reach `dev_ifsioc()`, which in turn calls `dev_change_flags()`. If the changed flag is `IFF_UP` or `IFF_DOWN`, the net device will be brought up or down. In the case of up, the relavent function is `dev_open()`.

### prepare

First netpoll will be disabled. Message of `NETDEV_PRE_UP` will be published. An registered operation `ndo_validate_addr()` will be called on the address configured for the net device, which, for E1000 driver, is `eth_validate_addr()`. Then `ndo_open()` will be called on the device, which is `e1000_open()`.

### `e1000_open()`

`e1000_open()` has saveral major components, setting up resource for tx and rx, powering up the physical device, configuring the hardware for tx and rx, allocating interrupt and registering its handler, enabling napi, enabling irq and finally enableing the queue to be used by upper layer. It will be investigated further.

#### set up tx and rx resource

The first task is to setup tx and rx resource. `e1000_setup_tx_resources()` will be called on all tx queues, which calls `dma_alloc_coherent()` to allocate memory for tx queues, and store the dma descriptor and address. `dma_alloc_coherent()` first attemps to allocate from the device coherent pool of dma memory, if not successful, `dma_direct_alloc()`, and then `alloc()` from `dma_ops`, which is configured in `dma_configure()` during initialization of device. More on dma will be covered in other documents. Rx resources are setup similarly. See this [article](https://www.kernel.org/doc/Documentation/DMA-API-HOWTO.txt) for basic knowledge of dma. Here only dma memory as accessible by the driver is setup.

#### power up device

Then the physical device is powered up by reading and writing some registers, and then some vlan settings.

#### `e1000_configure()`

`e1000_configure()` includes mostly of device specific hardware settings, most notably the configuration of tx and rx queue dma. For tx and rx queue, dma address is written to the hardware. After that, `alloc_rx_buf()` as registed to the adapter by `e1000_configure_rx()` is called, which could have many options but here we look only at `e1000_alloc_rx_buffers()` or `e1000_alloc_jumbo_rx_buffers()`. The queue is made up of a ring of buffers, each containing info for a skb, which should be setup for dma. Skb is first allocated for the ring via `__netdev_alloc_skb()`, which has a lot of magic build into it to make it faster. `dma_map_single()`, which returns address of memory as accessible by the hardware, is called on skb's data section and the handle is stored in the buffer info.

#### `e1000_request_irq()`

`e1000_request_irq()` requests irq for the handler `e1000_intr()`. If the interrupt is caused by rx error or link state change, the handler schedules a delayed work to `watchdog_task`, which is initialized in `e1000_probe()` to `e1000_watchdog()`. The delayed work records some info and prints debug message. If the link state is down, schedule additional work to `phy_info_task`, records and prints debug info for physical registers. If everything goes without error, the interrupt handler will call `__napi_schedule()` on this cpu's softnet structure. More on napi will be covered in a separate file.

#### `enable napi and irq`

After irq has been requested, `napi_enable()` is called to enable napi for this device. `e1000_irq_enable()` will enable default interrupt generation setting. Bits will be cleared for tx queue to allow transmit. Then the last step is to fire an interrupt to trigger the aforementioned watchdog in the interrupt handler.

### finish setting up and start watchdog

After `e1000_open()` finished, `dev_open()` will finally setting `IFF_UP` flag for the device. Then a few finishing operations is done. `ndo_set_rx_mode()`, which is device specific `e1000_set_rx_mode()`, sets up unicast, multicast and stuff. `dev_activate()` does mostly 2 things, creating and attaching a qdisc for every tx queue. If the qdisc is not noqueue_qdisc, Watchdog as configured by `dev_init_scheduler()`, `dev_watchdog()`, will be started and will run with the interval set by the driver, 5 HZ if not provided. When the device is working properly and the transmit queue is active (it will stop when there is a fault or too many packets piled up), the watchdog will check if the device transmit queue has timed out, and if so, call `ndo_tx_timeout` of the device.

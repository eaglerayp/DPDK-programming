# DPDK programming:
DPDK程式運行都必須要帶 -c -n 參數 (for EAL initial: -c 0xff -n 4)  
-c: 代表要用哪些 core 以 bit 表示 0x0f : 0~3 core  0xff 0~7 core 0xFFFF 0~16   
-n: Number of memory channels per processor socket  
multi core流程 example/helloworld  
基礎的封包轉發流程可參考example/skeleton

##### rte_eal_init(): 吃入參數初始化EAL包含：
* rte_eal_log_early_init()   
* rte_eal_cpu_init; 讀取cpu資訊(/proc/cpuinfo)紀錄在rte_config  
* eal_hugepage_info_init();  讀取/sys/kernel/mm/hugepages 初始化hugepage  
* rte_config_init();  建立rte_config 紀錄version,core資訊(數量,master,state)存放於shared 
* memory；且可使用rte_eal_get_configuration() retreive data  
* rte_eal_pci_init();  初始化網卡的pci driver；  
* rte_eal_memory_init() ;  
* rte_eal_alarm_init();  
* rte_eal_intr_init();  初始化interrupt；  
* rte_eal_timer_init() ;  
##### RTE_LCORE_FOREACH_SLAVE (traverse 所有slave core) 並初始建立pipe和thread  
```
RTE_LCORE_FOREACH_SLAVE(lcore_id){
rte_eal_remote_launch(lcore_hello, NULL, lcore_id);  //slave core call func
}
```
rte_eal_mp_wait_lcore();  最後結束回收其他thread


rte_eth_conf : 設定port(NIC) offload function
```
static const struct rte_eth_conf port_conf = {
	.rxmode = {
		.header_split = 0,      /* Header Split disabled */
		.hw_ip_checksum = 0,    /* IP checksum offload disabled */
		.hw_vlan_filter = 0,    /* VLAN filtering disabled */
		.jumbo_frame = 0,       /* Jumbo Frame Support disabled */
		.hw_strip_crc = 0,      /* CRC stripped by hardware */
	},
	.txmode = {
		.mq_mode = ETH_MQ_TX_NONE,
	},
};
```
##### forwarding
用無限for(;;)模擬NAPI polling  
從指定的port抓取某個receive queue,一共BURST_SIZE的data到bufs,nb_rx是成功拉取的資料量  
把bufs資料塞到portb的queue,總共塞nb_rx size的data , nb_tx是實際送出的packets數量
```
nb_rx = rte_eth_rx_burst(port, queue, bufs, BURST_SIZE);
nb_tx = rte_eth_tx_burst(portb, queue, bufs, nb_rx);
```

##### exception path
port都是設定為rte_eth_promiscuous_enable！！  
建立tap interface 0建立時接Port0 指定core0一直從port0收data轉發到tap0  
tap interface 1建立時就接到Port1 core1 always從tap1轉發到port1  
執行程式後再把兩個tap 利用bridge接起來就可以形成一個packet loop  
![overview of exception path](http://dpdk.org/doc/guides/_images/exception_path_example.svg)

##### rxtx_callbacks
在port init完之後加入 rx/tx port完成後的callback function.  
callbackfunction 會自帶參數如處理完的packet buffer,數量.  
像此例在收到packet後在packet裏面加入了收到時是第幾個cycle,  
然後再送出packet時計算packet在轉發期間花費了幾個cycle.  
```
add_timestamps(uint8_t port __rte_unused, uint16_t qidx __rte_unused,
		struct rte_mbuf **pkts, uint16_t nb_pkts,
		uint16_t max_pkts __rte_unused, void *_ __rte_unused)
	rte_eth_add_rx_callback(port, 0, add_timestamps, NULL);
	rte_eth_add_tx_callback(port, 0, calc_latency, NULL);
```
##### IP Fragmentation
init_mem()綁定direct/indirect mempool在CPU socket上, 建立LPM table.  
此例也可以看到如何在一個port建立multi receive queue per core.  
rte_eth_dev_configure傳入的conf指定哪些NIC的function要開啟, 現在測試用的不能開啟jumbo frame (err=-22代表有些offload功能無法開啟)

##### KNI
config 讓core 0,1負責port0的接發,2負責port0的KNI thread  
```
sudo ./build/kni -c 0x3f -n 4 -- -P -p 0x3 --config="(0,0,1,2),(1,3,4,5)"
```
啟用KNI和利用ifconfig vEth0_0 ip up後,還是必須啟用port=promiscuous mode才接收得到封包. 送封包沒問題,但是收不到(why?).
如果能利用KNI單純的將網卡驅動換成DPDK based 然後接上層應用也有加速效果.

##### layer 3 forwarding (power save)
由於DPDK會讓dedicated的core無限迴圈得poll packets,所以CPU使用率一直是100%>非常耗電, 所以利用DPDK提供powermanagement library, 可以在偵測rx_queue的traffic低的時候將CPU 調整 P/C state, CPU會調整frequency, 藉此降低耗電.

##### Load balancer

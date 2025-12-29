# In the Name of God

**Fatemeh Ghasemi – 404725104**

**Aida Seyedi – 404725086**

## DPDK Preparation and Build
Initially, to prepare and perform a custom build of DPDK so that the LTTng tool can record the entry and exit of all functions in the user space, DPDK must be compiled with specific flags.
We downloaded the latest version of DPDK from the official website and extracted it using the available commands.
The Meson tool was used with the `-finstrument-functions` flag. This flag ensures that function entry and exit points are marked during compilation; otherwise, the LTTng traces would be empty.

In the next step, we configured **Hugepages**. DPDK requires large memory pages for high-speed packet management. We allocated 1024 pages of 2 MB each using the command provided below.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/e110a7f1-745c-4848-ad39-396085e8149c" />

## Traffic Capture and Network Setup
For testing, we needed a traffic file (`Capture.pcap`). Initially, we faced an access challenge in Wireshark, where a standard user was not allowed to view the interfaces. This issue was resolved by running `sudo dpkg-reconfigure wireshark-common` and adding the user to the Wireshark group. Upon checking `ip addr`, it was determined that `eno2` was inactive and real traffic was flowing on `wlo1`. Therefore, `wlo1` had to be selected in Wireshark.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/52791cf6-9807-4489-adfe-476b0d5dc090" />

After clicking on `wlo1`, we entered the capture screen. We then performed some web browsing to record traffic and prepare the pcap file for the transmission phase. The pcap file was then extracted.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/e7990750-6a58-49a0-b17c-208faff9bf8b" />

## Running Testpmd and Traffic Injection
At this stage, the `testpmd` application was executed by pre-loading the LTTng library so that all functions could be traced.
To steer specific traffic to the desired queues, the following configurations were made in the `testpmd` interactive environment:
To view the ports, we used the command `show port stats all`. Then, we stopped the ports, added 2 RX/TX queues to them, ran `port stats all`, and finally defined a rule to steer all IPv4 UDP traffic to queue number 0.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/3f3ed0e5-496a-4810-a2fd-062189a9c6a1" />

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/066754ee-1ed0-4b38-bbe3-b1fa07a7141b" />


Next, to simulate network load, we installed the `tcpreplay` tool and started sending traffic using the pcap file we created.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/88de9ae4-c53c-4ad5-ac3f-9f78e29d86da" />


In the first terminal window (testpmd), we entered `start` followed by `show port stats all` to observe the packet flow on the TAP 0 interface.

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/62f942e2-5c93-4097-ad79-b42dc5e93105" />

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/85e5f547-ad80-4d71-b49e-a1e90b016699" />


## Recording the Trace
In the third terminal, we created a script (`script.sh`) to manage the recording session. This script records all user-space events with `vpid` and `procname` details for 1 second.
We pressed `ctrl+o` to save and `ctrl+x` to exit the environment, then executed the script.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/7d7998fc-a402-4905-aba7-741978b43485" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/d8905f4e-cef2-403d-9abb-fcb4c81152e3" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/2bd80685-e598-4d01-a19f-7c157dacdd59" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/3e900a15-5554-4165-acab-b920e8d91429" />

Since the script ran with root privileges, the files were saved in `/root/lttng-traces/`. Using the commands entered below, we moved them to a `mytraces` file in our own directory to access them in **TraceCompass**, as TraceCompass does not allow root login.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/121e7dd6-1477-4156-9a2f-3589698258ad" />

## TraceCompass Analysis
We then entered the TraceCompass application.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/08d69ed5-aca2-4a34-81d4-a407115420db" />

Approximately **4.28 million events** were recorded in a short time interval.
As shown in the Pie Chart, about 50% of events are related to `func_entry` and the other 50% to `func_exit`.
<img width="1920" height="1080" alt="Screenshot from 2025-12-24 18-26-40" src="https://github.com/user-attachments/assets/2bb69fd8-c777-4077-83ea-616221e2afe9" />

The blue graph below shows that the volume of events over time is very uniform and extremely high.
The selected time interval is about 1 second (995 ms).
Recording over 4 million events in one second indicates intense processing activity in `testpmd`. With this volume of tracing (recording every function entry/exit), a significant **Overhead** was likely imposed on the system, which may have affected the actual network performance results (Latency/Throughput).

### Flame Graph Analysis
This image is the **Flame Graph**, one of the most important sections for performance analysis. This chart hierarchically shows exactly which functions the CPU spent its time on.
The top level (Level 0) is the main processing function, `pkt_burst_io_forward`. This function is the beating heart of the `testpmd` application in DPDK, responsible for forwarding packets. Its subsets are divided into two main sections:
* **Left (Transmission - TX):** The `common_fwd_stream_transmit` function is visible, responsible for sending packets.
* **Right (Reception - RX):** The `common_fwd_stream_receive` function is visible, responsible for taking packets from the input.

<img width="1920" height="1080" alt="Screenshot from 2025-12-24 19-07-27" src="https://github.com/user-attachments/assets/0c9eee55-1f99-453d-b31c-d271512fbd7a" />

In the lower layers, you can see DPDK operational details:
* `rte_eth_tx_burst` and `rte_eth_rx_burst`: These are standard DPDK functions for sending and receiving packets in bursts.
* **PMD Functions:** Functions like `pmd_tx_burst` indicate **Poll Mode Driver** activity. This means the processor does not wait for interrupts and constantly checks the network card.

On the left, `tap_write_mbufs` is visible. This confirms that you are using a virtual **TAP driver**. Since the width of this rectangle is relatively large, it indicates that writing to the TAP interface took up a significant portion of the processing time.

A large part of the lower layers (green and purple colors) is related to buffer management:
* `rte_pktmbuf_free`: Freeing memory of sent packets.
* `rte_pktmbuf_alloc`: Allocating memory for new packets.
* `rte_mempool_get/put`: Interaction with the memory pool (Mempool).

If these blocks are excessively wide, it indicates that your system is more involved in memory management than packet content processing, which can be a bottleneck.
The empty space in the middle of the chart (between TX and RX sections) may indicate **Idle** times or waiting for the next packets to arrive.

## Detailed Function Analysis
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/139c3697-9278-4224-9528-973b1f9b101e" />

### Analysis of `rte_eth_rx_burst`
This is one of the most vital functions in the DPDK library, responsible for receiving a burst of packets from the network card.
* The total time taken for this function to execute and take packets from the input was about **2.7 microseconds**.
* About **1.3 microseconds** was spent on the internal code of this function. The rest (about 1.4 microseconds) was spent calling child functions.
* A time of 2.7 microseconds for a Burst call is very good, but it shows that the main bottleneck in the reception operation is the **memory allocation** section, as `rte_mempool` functions occupy a large part of the rectangle's width.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/65d70113-4293-43b6-969d-fa537eaf10c2" />

### Analysis of `common_fwd_stream_receive`
This function is responsible for calling packet reception mechanisms and preparing them for subsequent processing stages in DPDK.
* This function was called **356,726 times** during the trace interval. This very high number indicates the **Poll-mode** nature of DPDK. The processor constantly checks inputs in a loop to receive a packet as soon as it arrives.
* About **726.7 ms** was spent on this function. Each execution took an average of **2.037 microseconds**. This means system responsiveness in the reception layer is very high.
* The existence of a maximum value (16 ms) versus the average (2 microseconds) indicates **Jitter** or a sudden pause in the system, which could be due to a system Interrupt or Context Switch at that specific moment.
* Comparing the Total Time (726.7ms) and Self Time (167.8ms), we realize that this function spends only about 23% of its time on its own internal code and waits 77% of the time for child functions (like `rte_eth_rx_burst`). This shows the main bottleneck lies in lower layers (driver and hardware access), not in this function's logic.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/76392cbe-f2ca-4dd5-9e6d-4a6462881571" />

### Analysis of `common_fwd_stream_transmit`
This function is responsible for managing and steering packets towards the output (transmission).
* While the receive function was called over 350,000 times, the transmit function was executed only **20 times**. This shows that in this time interval, the system was mostly involved in **Polling** (waiting for packets), and packet transmission operations occurred much less frequently than input checks.
* Each transmission took an average of **17.707 microseconds**. The average transmission time (17.707 µs) is much higher than the average reception time (2.03 µs).
* This means the packet transmission process in your current configuration (likely due to using the TAP interface) is heavier and more time-consuming than the reception process.
* **Total Self Time:** About 9.176 microseconds.
* **Average Self Time:** Only 458 nanoseconds.
* The ratio of Self Time to Total Time shows that this function effectively performs no heavy processing and spends almost 97% of its time waiting for child functions (like `rte_eth_tx_burst` and finally `tap_write_mbufs`).
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/932bf8b8-1ce5-40b8-aaf9-1159bc8a8dbb" />

### Analysis of `rte_eth_tx_burst`
This function is at Depth 2 and is directly called by `common_fwd_stream_transmit`.
* **Number of calls:** 20 times.
* **Total Time:** About 344.97 microseconds.
* Comparing this function's time with its parent (354.14 microseconds), it becomes clear that **97%** of the total transmission process time is spent on this `rte_eth_tx_burst` function.
* **Average Time:** About 17.249 microseconds.
* The transmission time (17.2 µs) is almost **6 times longer** than the reception time (2.7 µs). This significant difference indicates a heavy processing overhead in the output section.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/8694a61f-3b38-48d1-8c96-fbff9fe42a6f" />

### Analysis of `rte_eth_rx_burst` (Reception Layer)
This function is responsible for the final delivery of packets to the hardware or virtual interface in DPDK.
* **Number of calls:** 356,726 times. This number matches exactly with its parent function (`common_fwd_stream_receive`). This means immediately in every reception loop execution, a burst command was issued to check the network card, indicating the correct operation of the Polling mechanism in DPDK.
* **Average Time:** 1.567 microseconds. This time indicates DPDK's incredible speed in checking inputs. Compared to the transmit function (`tx_burst`) which takes about 17 microseconds, the reception section acts much faster.
* **Maximum Time:** 16.015 milliseconds. The difference between the average (1.5 µs) and maximum (16 ms) shows that at a specific moment, the processor stopped for a long time or was involved in a side operation (like an OS interrupt or control tasks). This can cause **Tail Latency**.
* **Average Self Time:** 638 nanoseconds. Out of the total 1.5 microseconds, only 638 ns was spent on this function's own code, and the rest (about 900 ns) was spent calling lower-layer functions (PMD driver). This means your software layer is acting very lightly and optimally.

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/2f026bbe-d5f6-49c1-950b-855489f525c4" />

### Analysis of `rte_mempool_trace_get_bulk`
This function relates to the memory management system (Mempool) in DPDK and is used for tracing events related to retrieving a batch of objects from the memory pool.
* **Average Time:** Only 646 nanoseconds. This number shows that event logging and memory access in DPDK are incredibly fast. In fact, the entire process takes under 1 microsecond.
* **Average Self Time:** 439 nanoseconds. About 68% of this function's total time is spent executing its own internal code. This means the function is very focused and does not rely heavily on calling other functions at this stage.
* While packet receive functions (`rx_burst`) were called over 356,000 times, this specific memory tracing function was recorded only **23 times** in this interval. This indicates that **Bulk** allocation or freeing operations in memory happen much less frequently than checking for incoming packets, leading to system optimization.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/073e8dd5-0ae9-4fbe-82ea-4b94b519d915" />

### Analysis of `rte_pktmbuf_free`
This function is one of the most vital stages in the packet lifecycle in DPDK, responsible for freeing the memory of sent packets (mbufs) and returning them to the memory pool.
* This function is located on the left side of the Flame Graph and is called directly in the Transmit Path.
* It is observed that the process of freeing a packet takes about **9 microseconds**. Comparing this to packet reception time (1.5 microseconds), you realize that the memory cleaning and freeing operation in your system incurs a higher processing cost than the initial reception.
* **Average Self Time:** Only 438 nanoseconds. About 95% of this function's time is spent calling child functions. This shows that the logic of freeing a pointer itself is very fast, but the process of **Mempool Management** and updating packet metadata takes up the main portion of time.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/794b6ded-0229-425a-b6e2-23205422263f" />

### Analysis of `rte_ethdev_trace_rx_burst`
This is a Tracing Point function called exactly at the moment the packet reception operation starts in the abstract Ethernet layer (`ethdev`) in DPDK.
* **Total Time:** 835 nanoseconds.
* **Self Time:** 571 nanoseconds.
* These numbers show that the entire process of recording a Trace Point in the reception layer takes less than 1 microsecond. This means your monitoring tool (LTTng) is so optimized that it does not noticeably impact the main network performance.
* About 68% of this function's time is spent executing its internal code. This high ratio shows the function is very focused, aiming solely to quickly record the entry time into the packet reception stage.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/b00171be-3a9c-4282-ac12-d708edb19f7a" />

### Analysis of `rte_eth_tx_burst` (Depth 2)
This function is one of the most vital functions in the DPDK library for network output management. It is responsible for sending a Packet Burst to the network card or virtual interface.
* **Number of calls:** 20 times. This shows that in the entire recorded interval, the actual transmission operation occurred only 20 times.
* **Average Execution Time:** 17.249 microseconds. This time indicates the transmission latency in your system. This value is very high compared to the reception section (1.5 microseconds).
* **Self Time:** 885 nanoseconds. This very small number shows that the `rte_eth_tx_burst` code itself is very optimized, and almost all of the 17 microseconds were spent on child functions (like copying data in the TAP interface).
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/0fbb0007-8a6f-49ba-8cac-b18c5fdfd4d3" />

### Analysis of `rte_net_get_ptype`
This function plays a key role in packet pre-processing in DPDK. It stands for **Packet Type Discovery**. Its duty is to check packet headers (like L2, L3, and L4) to detect the protocol type (e.g., IPv4, TCP, UDP, or VLAN) without needing full and heavy packet processing.
* **Number of calls:** 23 times.
* **Average Execution Time:** About 5.831 microseconds. This shows that the protocol detection process for each packet takes about 6 microseconds.
* **Self Time Ratio:** 53%. Unlike many previous functions that were just intermediaries, this function spends a large part of its time (over 50%) executing its own internal code to analyze packet header bits.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/dcff0813-2e58-4491-b1e5-281f8d4cfef2" />

### Analysis of `rte_mbuf_raw_alloc` (Layer 5 - RX Path)
This function is one of the key stages in preparing memory for incoming packets. It is responsible for **Raw Allocation** of an mbuf structure from the Mempool. "Raw" means the function only retrieves memory from the pool and, unlike higher-level functions, does not perform full initialization, maximizing data plane processing speed.
* **Number of calls:** 23 times. This number confirms again that exactly 23 packets entered the system during this interval. The consistency of this number across all memory allocation layers (from layer 4 to 10) indicates complete stability in resource management in your test.
* **Average Execution Time:** 6.178 microseconds. This indicates the processing cost to reserve memory for a new packet.
* **Net Self Time:** 706 nanoseconds. Only about 11% of this function's time was spent on its internal code. The rest (5.4 microseconds) was spent calling deeper functions like `rte_mempool_get_bulk`, showing that the bulk of allocation time is spent interacting with the memory pool.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/2cefad2e-55a0-494e-91be-9bb0e528b077" />

### Analysis of `rte_mempool_put_bulk` (Layer 8)
This function is called when packets have been sent and the system needs to free their memory for future use.
* **Number of calls:** 23 times. Note: This number matches exactly with the number of input packets allocated in the receive layer (`raw_alloc`). This coordination indicates a stable system where all acquired packets are successfully freed after traversing the transmission path.
* **Average Duration:** About 5.779 microseconds. This shows that returning a batch of buffers to the memory pool takes close to 6 microseconds on average.
* **Average Self Time:** About 1.072 microseconds. Only about 18% of this function's time is spent on its internal code. The rest (about 4.7 microseconds) is spent calling deeper functions (layers 9 and 10) for final pointer management in the memory pool.
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/8f349a80-59ab-4623-a474-88b4ac00f26d" />

### Analysis of `rte_pktmbuf_prefree_seg` (Layer 6)
This function is responsible for preparing a message buffer segment before actually returning it to the memory pool. Its most important job is checking and managing the Reference Count; i.e., ensuring no other part of the program still needs this memory.
* **Number of calls:** 23 times.
* **Average Duration:** 682 nanoseconds. This function executes very fast (less than 1 microsecond), indicating the optimization of DPDK code in managing packet references.
* **Average Self Time:** 462 nanoseconds. About 68% of this function's time is spent executing its own internal logic.

## Final Conclusion and System Bottleneck Analysis
In this experiment, using the **Trace Compass** tool and tracking user-space events, the packet lifecycle in the `dpdk-testpmd` application was precisely evaluated. The analysis of the **Flame Graph** and extracted timing statistics led to the precise identification of system behavior and bottlenecks, as detailed below:

### 1. Identification and Analysis of the Main Bottleneck
Comparative analyses between function execution times revealed that the system's main bottleneck lies in the **Transmit Path (TX Path)**. The performance asymmetry between system input and output is evident:
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/39c575d9-3edc-40da-8f52-b514ca5990f2" />

* **Precise Bottleneck Point:** The `tap_write_mbufs` function, a subset of the `rte_eth_tx_burst` transmission function, was identified as the main cause of slowness.
* **Statistical Evidence:**
    * Average processing time in the Receive (RX) layer is very favorable at about **1.56 microseconds**.
    * Average processing time in the Transmit (TX) layer increases to about **17.24 microseconds**.
    * This **11-fold difference** shows that the system output cannot process at the same pace as the input, and there is significant overhead in this section.

### 2. Technical Cause of the Bottleneck
A close examination of the call stack reveals that the main reason for this slowness is the inherent limitations of the **TAP Interface**:
* **Violation of Kernel Bypass Mechanism:** In the ideal DPDK architecture, the goal is direct hardware access and bypassing the kernel. However, using the TAP interface prevents DPDK from utilizing the Direct Memory Access (DMA) mechanism.
* **Memory Copy and System Call Overhead:** The system is forced to copy data from user space to kernel space for every packet. This memory copy operation, combined with overhead from system calls and context switching, causes a severe performance drop, increasing transmission time from a few nanoseconds to 17 microseconds.

### 3. Memory Management Status and Stability
Contrary to the bottleneck identified in the transmission section, the memory subsystem analysis indicates complete stability. The number of calls to allocation functions (`rte_pktmbuf_alloc`) and freeing functions (`rte_pktmbuf_free`) was recorded as exactly equal (**23 times**) across all layers. This guarantees that the application is free of memory leaks, although the process of returning memory to the pool (8.9 microseconds) carries its own processing overhead.

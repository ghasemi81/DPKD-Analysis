**In the Name of God**

**Fatemeh Ghasemi – 404725104**

**Aida Seyedi – 404725086**

### 1. DPDK Preparation and Custom Build
To enable LTTng to trace the entry and exit points of all functions within the user space, DPDK required compilation with specific flags. We retrieved the latest version of DPDK from the official website and extracted the archive using the standard commands.

### 2. Hugepages Configuration
High-speed packet management in DPDK necessitates the use of large memory pages. Consequently, we allocated 1,024 hugepages (2 MB each) to the memory using the command provided below.

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/e110a7f1-745c-4848-ad39-396085e8149c" />

### 3. Traffic Capture Setup
A traffic file (`Capture.pcap`) was required for the testing phase. Initially, we encountered permission issues in Wireshark, as standard users were restricted from viewing interfaces. This was resolved by executing `sudo dpkg-reconfigure wireshark-common` and adding the user to the Wireshark group.
Upon inspection via `ip addr`, we identified that `eno2` was down, while active traffic was present on `wlo1`. Thus, `wlo1` was selected as the capture interface in Wireshark.

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/52791cf6-9807-4489-adfe-476b0d5dc090" />

After initiating the capture on `wlo1`, we generated traffic via web browsing to create the necessary PCAP file for the transmission phase. The file was then exported.

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/e7990750-6a58-49a0-b17c-208faff9bf8b" />

### 4. Testpmd Execution
The `testpmd` application was launched with the LTTng library preloaded to ensure comprehensive function tracing.

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/3f3ed0e5-496a-4810-a2fd-062189a9c6a1" />

To route specific traffic to target queues, the following configuration was applied in the `testpmd` interactive mode:
* Displayed port statistics using `show port stats all`.
* Stopped ports to configure **2 RX/TX queues**.
* Verified configuration with `port stats all`.
* Defined a flow rule to direct all **IPv4 UDP traffic** to **Queue 0**.
  
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/066754ee-1ed0-4b38-bbe3-b1fa07a7141b" />

### 5. Traffic Simulation
To simulate network load, the `tcpreplay` tool was installed and utilized to transmit the previously generated PCAP file.

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/88de9ae4-c53c-4ad5-ac3f-9f78e29d86da" />

In the first terminal (`testpmd`), the `start` command was issued, followed by `show port stats all` to monitor packet flow on the **TAP 0** interface.

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/62f942e2-5c93-4097-ad79-b42dc5e93105" />

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/85e5f547-ad80-4d71-b49e-a1e90b016699" />

In the third terminal, a script (`script.sh`) was created to manage the recording session. This script captured all user-space events, including `vpid` and `procname`, for a duration of **1 second**. The script was saved (`Ctrl+O`) and executed after exiting the editor (`Ctrl+X`).

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/7d7998fc-a402-4905-aba7-741978b43485" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/d8905f4e-cef2-403d-9abb-fcb4c81152e3" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/2bd80685-e598-4d01-a19f-7c157dacdd59" />
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/3e900a15-5554-4165-acab-b920e8d91429" />

### 6. File Management
Since the tracing script ran with root privileges, the output files were stored in `/root/lttng-traces/`. To make these files accessible to **Trace Compass** (which lacks root access), we transferred them to a user-accessible directory named `mytraces` using the commands below.

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/121e7dd6-1477-4156-9a2f-3589698258ad" />

### 7. Trace Compass Analysis
Upon importing the data into Trace Compass:
* Approximately **4.28 million events** were recorded within the short capture window.
* The pie chart indicates an even distribution: **50%** for `func_entry` and **50%** for `func_exit`.
* The time-series graph (blue) demonstrates a consistent and high volume of events throughout the capture.
  
<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/08d69ed5-aca2-4a34-81d4-a407115420db" />

#### Flame Graph Interpretation
The **Flame Graph** is a pivotal tool for performance analysis, hierarchically displaying CPU time allocation per function.
* **Wide Blocks:** Indicate that the system is heavily engaged in memory management rather than packet payload processing, suggesting a potential bottleneck.
* **Gaps:** Empty spaces between transmission and reception blocks typically indicate idle time or latency in packet arrival.

<img width="1920" height="1080" alt="Screenshot from 2025-12-24 19-07-27" src="https://github.com/user-attachments/assets/0c9eee55-1f99-453d-b31c-d271512fbd7a" />

### 8. Conclusion and Bottleneck Identification
In this experiment, the packet lifecycle within the `dpdk-testpmd` application was rigorously evaluated using Trace Compass. The analysis of the Flame Graph and timing statistics facilitated the identification of system behavior and potential bottlenecks.

**Criteria for Bottleneck Detection:**
1.  **High Depth:** Indicates deep nesting of function calls.
2.  **High Self Time:** Indicates a significant portion of execution time is spent within the function itself rather than in sub-calls (ratio of Self Time to Total Time).
3.  **High Call Count:** Indicates frequent execution.

*Note: While these metrics suggest potential bottlenecks, source code verification is essential to determine whether the behavior is intrinsic to the function or optimizable.*

<img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/73376dd0-84b8-46f5-a983-be94429dd4c2" />

`rte_net_get_ptype`
The `rte_net_get_ptype` function is critical for packet pre-processing, responsible for parsing headers to determine protocol types without heavy processing.
Analyzing this function against our criteria reveals:
* **Call Count:** 23
* **Average Self Time:** 3.085 µs
* **Average Duration:** 5.831 µs

This yields a useful time ratio of approximately **53%**. The high self-time suggests that a substantial amount of time is consumed by the function's internal logic, making it a primary candidate for bottleneck analysis.


# 3. TShark

### <u>**Task 1 - Introduction**</u>

TShark is an open-source command-line network traffic analyser. It is created by the Wireshark developers and has most of the features of Wireshark. It is commonly used as a command-line version of Wireshark. However, it can also be used like tcpdump. Therefore it is preferred for comprehensive packet assessments.

### <u>**Task 2 - Command-Line Packet Analysis Hints | TShark and Supplemental CLI Tools**</u>

| **Tool/Utility** | **Purpose and Benefit** |
| --- | --- |
| **capinfos** | A program that provides details of a specified capture file. It is suggested to view the summary of the capture file before starting an investigation. |
| **grep** | Helps search plain-text data. |
| **cut** | Helps cut parts of lines from a specified data source. |
| **uniq** | Filters repeated lines/values. |
| **nl** | Views the number of shown lines. |
| **sed** | A stream editor. |
| **awk** | Scripting language that helps pattern search and processing. |

```bash
**user@ubuntu$ capinfos demo.pcapng 
File name:           demo.pcapng
File type:           Wireshark/tcpdump/... - pcap
File encapsulation:  Ethernet**
```

View the details of the demo.pcapng file with "capinfos".

> What is the "RIPEMD160" value?
> 

capinfos demo.pcapng RIPEMD160: 6ef5f0c165a1db4a3cad3116b0c5bcc0cf6b9ab7

### <u>**Task 3 - TShark Fundamentals I | Main Parameters I**</u>

| **Parameter** | **Purpose** |
| --- | --- |
| -h | • Display the help page with the most common features.
• `tshark -h` |
| -v | • Show version info.
• `tshark -v` |
| -D | • List available sniffing interfaces.
• `tshark -D` |
| -i | • Choose an interface to capture live traffic.
• `tshark -i 1`
• `tshark -i ens55` |
| **No Parameter** | • Sniff the traffic like tcpdump.
• `tshark` |

```bash
user@ubuntu$ tshark -v 
TShark (Wireshark) 3 (Git v3. packaged as 3.)
```

**Sniffing:** TShark uses that interface to sniff the traffic. TShark will use the first available interface when no interface is selected, usually listed as 1 in the terminal. 

> What is the installed TShark version in the given VM? tshark -v > 3.2.3
> 

List the available interfaces with TShark. What is the number of available interfaces in the given VM? tshark -D >  12 

### <u>**Task 4 -  TShark Fundamentals I | Main Parameters II**</u>

| **Parameter** | **Purpose** |
| --- | --- |
| -r | • Read/input function. Read a capture file.
• `tshark -r demo.pcapng` |
| -c | • Packet count. Stop after capturing a specified number of packets.
• E.g. stop after capturing/filtering/reading 10 packets.
• `tshark -c 10` |
| -w | • Write/output function. Write the sniffed traffic to a file.
• `tshark -w sample-capture.pcap` |
| -V | • Verbose.
• Provide detailed information **for each packet**. This option will provide details similar to Wireshark's "Packet Details Pane".
• `tshark -V` |
| -q | • Silent mode.
• Suspress the packet outputs on the terminal.
• `tshark -q` |
| -x | • Display packet bytes.
• Show packet details in hex and ASCII dump for each packet.
• `tshark -x` |

**Read Capture Files:** TShark can also process PCAP files. You can use the `-r` parameter to process the file and investigate the packets. You can limit the number of shown packets using the `-c` parameter.

```bash
user@ubuntu$ tshark -r demo.pcapng
```

**Write Data:** TShark can also write the sniffed or filtered packets to a file. You can save the sniffed traffic to a file using the `-w` parameter. This option helps analysts to separate specific packets from the file/traffic and save them for further analysis.

```bash
user@ubuntu$ tshark -r demo.pcapng -c 1 -w write-demo.pcap
```

**Show Packet Bytes:** TShark can show packet details in hex and ASCII format. You can view the dump of the packets by using the `-x` parameter. Once you use this parameter, all packets will be shown in hex and ASCII format.

```bash
user@ubuntu$ tshark -r write-demo.pcap 
```

**Verbosity:** Default TShark packet processing and sniffing operations provide a single line of information and exclude verbosity. The default approach makes it easy to follow the number of processed/sniffed packets; however, TShark can also provide verbosity for each packet when instructed. 

> Read the "demo.pcapng" file with TShark. What are the assigned TCP flags in the 29th packet?
> 

**tshark -r demo.pcapng -c 29 > PSH, ACK**

> What is the "Ack" value of the 25th packet? **12421**
> 

> What is the "Window size value" of the 9th packet? **9660**
> 

### <u>**Task 5 - TShark Fundamentals II | Capture Conditions**</u>

Capture condition parameters only work in the “capturing/sniffing” mode. You will receive an error message if you try to read a pcap file and apply the capture condition parameters. The idea is to save the capture files in specific sizes for different purposes during live capturing. If you need to extract sorts of packets from a specific capture file, you will need to use the read&write options discussed in the previous task.

 ****TShark supports combining autostop (`-a`) parameters with ring buffer control parameters (`-b`). You can combine the parameters according to your needs. Use the infinite loop options carefully; remember, you must use at least one autostop parameter to stop the infinite loop.

| **Parameter** | **Purpose** |
| --- | --- |
|  | Define capture conditions for a single run/loop. STOP after completing the condition. Also known as "Autostop". |
| -a | • **Duration:** Sniff the traffic and stop after X seconds. Create a new file and write output to it.
    ◦ `tshark -w test.pcap -a duration:1`
• **Filesize:** Define the maximum capture file size. Stop after reaching X file size (KB).
    ◦ `tshark -w test.pcap -a filesize:10`
• **Files:** Define the maximum number of output files. Stop after X files.
    ◦ `tshark -w test.pcap -a filesize:10 -a files:3` |
|  | Ring buffer control options. Define capture conditions for multiple runs/loops. (INFINITE LOOP). |
| -b | • **Duration:** Sniff the traffic for X seconds, create a new file and write output to it. 
    ◦ `tshark -w test.pcap -b duration:1`
• **Filesize:** Define the maximum capture file size. Create a new file and write output to it after reaching filesize X (KB).
    ◦ `tshark -w test.pcap -b filesize:10`
• **Files:** Define the maximum number of output files. Rewrite the first/oldest file after creating X files.
    ◦ `tshark -w test.pcap -b filesize:10 -b files:3` |

> Which parameter can help analysts to create a **continuous** capture dump? -b
> 

> Can we combine autostop and ring buffer parameters with TShark? y
> 

### <u>**Task 6 - TShark Fundamentals III | Packet Filtering Options: Capture vs. Display Filters**</u>

There are two dimensions of packet filtering in TShark; live (capture) and post-capture (display) filtering. These two dimensions can be filtered with two different approaches; using a predefined syntax or Berkeley Packet Filters (BPF). TShark supports both, so you can use Wireshark filters and BPF to filter traffic. 

| **Capture Filters** | Live filtering options. The purpose is to **save** only a specific part of the traffic. It is set before capturing traffic and is not changeable during live capture. |
| --- | --- |
| **Display Filters** | Post-capture filtering options. The purpose is to investigate packets by **reducing the number of visible packets**, which is changeable during the investigation. |

Capture filters are used to have a specific type of traffic in the capture file rather than having everything. Capture filters have limited filtering features, and the purpose is to implement a scope by range, protocol, and direction filtering. This might sound like bulk/raw filtering, but it still provides organised capture files with reasonable file size. The display filters investigate the capture files in-depth without modifying the packet.

| **Parameter** | **Purpose** |
| --- | --- |
| -f | Capture filters. Same as BPF syntax and Wireshark's capture filters. |
| -Y | Display filters. Same as **Wireshark's display filters.** |

### <u>**Task 7 - TShark Fundamentals IV | Packet Filtering Options: Capture Filters**</u>

Wireshark's capture filter syntax is used here. The basic syntax for the Capture/BPF filter is shown below. You can read more on capture filter syntax [here(opens in new tab)](https://www.wireshark.org/docs/man-pages/pcap-filter.html) and [here(opens in new tab)](https://gitlab.com/wireshark/wireshark/-/wikis/CaptureFilters#useful-filters). Boolean operators can also be used in both types of filters. 

The Berkeley Packet Filter (BPF) is a technology used in certain computer operating systems for programs that need to, among other things, analyze network traffic. BPF supports filtering packets, allowing a userspace process to supply a filter program that specifies which packets it wants to receive.

| **Qualifier** | **Details and Available Options** |
| --- | --- |
| **Type** | Target match type. You can filter IP addresses, hostnames, IP ranges, and port numbers. Note that if you don't set a qualifier, the "host" qualifier will be used by default.
• host | net | port | portrange
• Filtering a host
    ◦ `tshark -f "host 10.10.10.10"`
• Filtering a network range 
    ◦ `tshark -f "net 10.10.10.0/24"`
• Filtering a Port
    ◦ `tshark -f "port 80"`
• Filtering a port range
    ◦ `tshark -f "portrange 80-100"` |
| **Direction** | Target direction/flow. Note that if you don't use the direction operator, it will be equal to "either" and cover both directions.
• src | dst
• Filtering source address
    ◦ `tshark -f "src host 10.10.10.10"`
• Filtering destination address
    ◦ `tshark -f "dst host 10.10.10.10"` |
| **Protocol** | Target protocol.
• arp | ether | icmp | ip | ip6 | tcp | udp
• Filtering TCP
    ◦ `tshark -f "tcp"`
• Filtering MAC address
    ◦ `tshark -f "ether host F8:DB:C5:A2:5D:81"`
• You can also filter protocols with IP Protocol numbers assigned by IANA.
• Filtering IP Protocols 1 (ICMP)
    ◦ `tshark -f "ip proto 1"`
    ◦ [**Assigned Internet Protocol Numbers**](https://www.iana.org/assignments/protocol-numbers/protocol-numbers.xhtml) |

We need to create traffic noise to test and simulate capture filters. We will use the "terminator" terminal instance to have a split-screen view in a single terminal. The "terminator" will help you craft and sniff packets using a single terminal interface. Now, run the `terminator` command and follow the instructions using the new terminal instance.

- First, run the given TShark command in Terminal-1 to start sniffing traffic.
- Then, run the given cURL command in Terminal-2 to create network noise.
- View sniffed packets results in Terminal-1.

"Terminator" Terminal Emulator Application Terminal-1; 

```bash
user@ubuntu$ tshark -f "host 10.10.10.10"
Capturing on 'ens5'
    1 0.000000000 YOUR-IP → 10.10.10.10  TCP 74 36150 → 80 [SYN] Seq=0 Win=62727 Len=0 MSS=8961 SACK_PERM=1
```

```bash
**user@ubuntu$ curl -v 10.10.10.10** Trying 10.10.10.10:80...
* TCP_NODELAY set
* Connected to 10.10.10.10 (10.10.10.10) port 80 (#0)
> GET / HTTP/1.1
> Host: 10.10.10.10
> User-Agent: curl/7.68.0
```

Being comfortable with the command line and TShark filters requires time and practice. You can use the below table to practice TShark capture filters.

| **Capture Filter Category** | **Details** |
| --- | --- |
| **Host Filtering** | <u>Capturing</u> traffic to or from a specific host.
• Traffic generation with cURL. This command sends a default HTTP query to a specified address.
    ◦ `curl tryhackme.com`
• TShark capture filter for a host
    ◦ `tshark -f "host tryhackme.com"` |
| **IP Filtering** | <u>Capturing</u> traffic to or from a specific port. We will use the Netcat tool to create noise on specific ports.
• Traffic generation with Netcat. Here Netcat is instructed to provide details (verbosity), and timeout is set to 5 seconds.
    ◦ `nc 10.10.10.10 4444 -vw 5`
• TShark capture filter for specific IP address
    ◦ `tshark -f "host 10.10.10.10"` |
| **Port Filtering** | <u>Capturing</u> traffic to or from a specific port. We will use the Netcat tool to create noise on specific ports.
• Traffic generation with Netcat. Here Netcat is instructed to provide details (verbosity), and timeout is set to 5 seconds.
    ◦ `nc 10.10.10.10 4444 -vw 5`
• TShark capture filter for port 4444
    ◦ `tshark -f "port 4444"` |
| **Protocol Filtering** | <u>Capturing</u> traffic to or from a specific protocol. We will use the Netcat tool to create noise on specific ports.
• Traffic generation with Netcat. Here Netcat is instructed to use UDP, provide details (verbosity), and timeout is set to 5 seconds.
    ◦ `nc -u 10.10.10.10 4444 -vw 5`
• TShark capture filter for
    ◦ `tshark -f "udp"` |

> What is the number of packets with SYN bytes? 2
> 

**terminal 1 tshark -f “host 10.10.10.10”** now on the **terminal 2** run this command **curl -v 10.10.10.10** now the packets gets captured on the terminal 1 and now you will need to count the number of SYN packets you will find the answer

`curl` — tool for making network requests (HTTP, HTTPS, FTP, etc.) This command downloads a website's integrity report to the terminal.

`v` — “verbose”; shows detailed connection/debug information

`10.10.10.10` — the target host/IP

> What is the number of packets sent to the IP address "10.10.10.10"? 7
> 

To find the answer count the number of packets to 10.10.10.10 you will need to manually count the packets from the previous results

> What is the number of packets with ACK bytes? 8
> 

To find the answer count the number of ACK packets on the given results 

### <u>**Task 8 - TShark Fundamentals V | Packet Filtering Options: Display Filters**</u>

| **Display Filter Category** | **Details and Available Options** |
| --- | --- |
| **Protocol: IP** | • <u>Filtering</u> an IP without specifying a direction.
    ◦ `tshark -Y 'ip.addr == 10.10.10.10'`
• <u>Filtering</u> a network range 
    ◦ `tshark -Y 'ip.addr == 10.10.10.0/24'`
• <u>Filtering</u> a source IP
    ◦ `tshark -Y 'ip.src == 10.10.10.10'`
• <u>Filtering</u> a destination IP
    ◦ `tshark -Y 'ip.dst == 10.10.10.10'` |
| **Protocol: TCP** | • <u>Filtering</u> TCP port
    ◦ `tshark -Y 'tcp.port == 80'`
• <u>Filtering</u> source TCP port
    ◦ `tshark -Y 'tcp.srcport == 80'` |
| **Protocol: HTTP** | • <u>Filtering</u> HTTP packets
    ◦ `tshark -Y 'http'`
• <u>Filtering</u> HTTP packets with response code "200"
    ◦ `tshark -Y "http.response.code == 200"` |
| **Protocol: DNS** | • <u>Filtering</u> DNS packets
    ◦ `tshark -Y 'dns'`
• <u>Filtering</u> all DNS "A" packets
    ◦ `tshark -Y 'dns.qry.type == 1'` |

Sample filtering query: The below terminal demonstrates using the "IP filtering" option. TShark filters the packets and provides the output in our terminal.

```bash
user@ubuntu$ tshark -r demo.pcapng -Y 'ip.addr == 145.253.2.203'
13 2.55 145.254.160.237 ? 145.253.2.203 DNS Standard query 0x0023 A ..
17 2.91 145.253.2.203 ? 145.254.160.237 DNS Standard query response 0x0023 A ..
```

```bash
user@ubuntu$ tshark -r demo.pcapng -Y 'http' 
4   0.911 145.254.160.237 ? 65.208.228.223 HTTP GET /download.html HTTP/1.1  
18   2.984 145.254.160.237 ? 216.239.59.99 HTTP GET /pagead/ads?client... 
27   3.955 216.239.59.99 ? 145.254.160.237 HTTP HTTP/1.1 200 OK  (text/html) 
38   4.846 65.208.228.223 ? 145.254.160.237 HTTP/XML HTTP/1.1 200 OK
```

 You can use the `nl` command to get a numbered list of your output. Therefore you can easily calculate the "total number of filtered packets" without being confused with "assigned packet numbers". The usage of the `nl` command is shown below.

```bash
user@ubuntu$ tshark -r demo.pcapng -Y 'http' | nl 1    
4  0.911 145.254.160.237 ? 65.208.228.223 HTTP GET /download.html HTTP/1.1  
2   18  2.984 145.254.160.237 ? 216.239.59.99 HTTP GET /pagead/ads?client... 
3   27   3.955 216.239.59.99 ? 145.254.160.237 HTTP HTTP/1.1 200 OK (text/html) 
4   38   4.846 65.208.228.223 ? 145.254.160.237 HTTP/XML HTTP/1.1 200 OK
```

**Use the "demo.pcapng" file to answer the questions.**

> What is the number of packets with a "65.208.228.223" IP address? 34
> 

 **tshark -r demo.pcapng -Y “ip.addr == 65.208.228.223” | wc -l** 

> What is the number of packets with a "TCP port 3371"? 7
> 

 **tshark -r demo.pcapng -Y “tcp.port == 3371” | wc -l**

> What is the number of packets with a "145.254.160.237" IP address as a source address? 20
> 

**tshark -r demo.pcapng -Y “ip.src == 145.254.160.237” | wc -l** 

> **Rerun the previous query and look at the output.** What is the packet number of the "Duplicate" packet? 37
> 

### <u>**Task 9 - Conclusion**</u>

In this room, we covered TShark, what it is, how it operates, and how to use it to investigate traffic captures.

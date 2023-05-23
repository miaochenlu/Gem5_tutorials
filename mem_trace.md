# Generate & Analysis Memory Trace

You can see modified code at https://github.com/miaochenlu/gem5_explore/tree/mem-trace

# 1. CommMonitor
## 1.1 CommMonitor Overview

![](attachments/Pasted%20image%2020230523212048.png)
![](attachments/Pasted%20image%2020230523212052.png)
The communication monitor is a SimObject which can monitor statistics of the communication happening between two ports in the memory system. 

### a. Workflow of CommMonitor
CommMonitor's workflow involves several key functions.
In the `recvTimingReq` function, CommMonitor transmits the request from the memSidePort upon receiving the packet. Additionally, it utilizes the packet information to perform various tasks, such as generating traces.
CommMonitor employs probepoints, like ppPktReq. After successfully sending out a packet, it invokes the notify function and proceeds to carry out additional operations within the `handleRequest` function.
```cpp
void CommMonitor::regProbePoints() {
    ppPktReq.reset(new probing::Packet(getProbeManager(), "PktRequest"));
    ppPktResp.reset(new probing::Packet(getProbeManager(), "PktResponse"));
}

bool CommMonitor::recvTimingReq(PacketPtr pkt) {
    ...
    bool successful = memSidePort.sendTimingReq(pkt);  
    ...  
    if (successful) {
        ppPktReq->notify(pkt_info);
    }  
    ...
}

void notify(const probing::PacketInfo &pkt_info) override {
    parent.handleRequest(pkt_info);
}
```

### b. CommMonitor with MemTrace
The `handleRequest` function in MemTraceProbe is implemented to utilize the packet information for generating traces.
```cpp
void
MemTraceProbe::handleRequest(const probing::PacketInfo &pkt_info)
{
    ProtoMessage::Packet pkt_msg;
    pkt_msg.set_tick(curTick());
    pkt_msg.set_cmd(pkt_info.cmd.toInt());
    pkt_msg.set_flags(pkt_info.flags);
    pkt_msg.set_addr(pkt_info.addr);
    pkt_msg.set_size(pkt_info.size);
    if (withPC && pkt_info.pc != 0)
        pkt_msg.set_pc(pkt_info.pc);
    pkt_msg.set_pkt_id(pkt_info.id);
    pkt_msg.set_inst(pkt_info.inst);
    pkt_msg.set_vaddr(pkt_info.vaddr);
    traceStream->write(pkt_msg);
}

```

## 1.2 Modify Gem5 Codes
### a. Add trace-monitor option

In `configs/common/Options.py`, we add an option to notice that we want to generate memory traces using CommMonitor
```python
    # Trace Monitor option
    parser.add_argument(
        "--trace-monitor", action="store_true", help="enable trace monitor"
    )
```

### b. Add CommMonitor for l1->l2 & l2->mem
Modify `configs/common/CacheConfig.py`
#### i. l2->mem
```diff
--- a/configs/common/CacheConfig.py
+++ b/configs/common/CacheConfig.py
@@ -140,7 +140,16 @@ def config_cache(options, system):
 
         system.tol2bus = L2XBar(clk_domain=system.cpu_clk_domain)
         system.l2.cpu_side = system.tol2bus.mem_side_ports
-        system.l2.mem_side = system.membus.cpu_side_ports
+
+        if options.trace_monitor:
+            system.monitor_l2_mem = CommMonitor()
+            system.monitor_l2_mem.trace = MemTraceProbe(
+                trace_file="l2_mem_trace.tar.gz"
+            )
+            system.monitor_l2_mem.cpu_side_port = system.l2.mem_side
+            system.monitor_l2_mem.mem_side_port = system.membus.cpu_side_ports
+        else:
+            system.l2.mem_side = system.membus.cpu_side_ports
 
     if options.memchecker:
         system.memchecker = MemChecker()
```
#### ii. l1->l2
```diff
@@ -207,11 +216,44 @@ def config_cache(options, system):
 
         system.cpu[i].createInterruptController()
         if options.l2cache:
-            system.cpu[i].connectAllPorts(
-                system.tol2bus.cpu_side_ports,
-                system.membus.cpu_side_ports,
-                system.membus.mem_side_ports,
-            )
+            if options.trace_monitor:
+                system.monitor_cpu_l1d_l2 = CommMonitor()
+                system.monitor_cpu_l1d_l2.trace = MemTraceProbe(
+                    trace_file="l1d_l2_trace.tar.gz"
+                )
+                system.monitor_cpu_l1d_l2.cpu_side_port = system.cpu[
+                    i
+                ].dcache.mem_side
+                system.monitor_cpu_l1d_l2.mem_side_port = (
+                    system.tol2bus.cpu_side_ports
+                )
+
+                system.monitor_cpu_l1i_l2 = CommMonitor()
+                system.monitor_cpu_l1i_l2.trace = MemTraceProbe(
+                    trace_file="l1i_l2_trace.tar.gz"
+                )
+                system.monitor_cpu_l1i_l2.cpu_side_port = system.cpu[
+                    i
+                ].icache.mem_side
+                system.monitor_cpu_l1i_l2.mem_side_port = (
+                    system.tol2bus.cpu_side_ports
+                )
+
+                system.cpu[i].connectUncachedPorts(
+                    system.membus.cpu_side_ports, system.membus.mem_side_ports
+                )
+                system.cpu[
+                    i
+                ].itb_walker_cache.mem_side = system.tol2bus.cpu_side_ports
+                system.cpu[
+                    i
+                ].dtb_walker_cache.mem_side = system.tol2bus.cpu_side_ports
+            else:
+                system.cpu[i].connectAllPorts(
+                    system.tol2bus.cpu_side_ports,
+                    system.membus.cpu_side_ports,
+                    system.membus.mem_side_ports,
+                )
         elif options.external_memory_system:
             system.cpu[i].connectUncachedPorts(
                 system.membus.cpu_side_ports, system.membus.mem_side_ports
```

### c. Add Commonitor for cpu->l1

#### i. Modify `src/cpu/BaseCPU.py`
```diff
--- a/src/cpu/BaseCPU.py
+++ b/src/cpu/BaseCPU.py
@@ -53,6 +53,8 @@ from m5.objects.CPUTracers import ExeTracer
 from m5.objects.SubSystem import SubSystem
 from m5.objects.ClockDomain import *
 from m5.objects.Platform import Platform
+from m5.objects.CommMonitor import CommMonitor
+from m5.objects.MemTraceProbe import MemTraceProbe
 from m5.objects.ResetPort import ResetResponsePort
 
 default_tracer = ExeTracer()
@@ -189,11 +191,32 @@ class BaseCPU(ClockedObject):
             bus.cpu_side_ports, bus.cpu_side_ports, bus.mem_side_ports
         )
 
-    def addPrivateSplitL1Caches(self, ic, dc, iwc=None, dwc=None):
+    def addPrivateSplitL1Caches(
+        self, ic, dc, iwc=None, dwc=None, monitor=False
+    ):
         self.icache = ic
         self.dcache = dc
-        self.icache_port = ic.cpu_side
-        self.dcache_port = dc.cpu_side
+
+        if monitor:
+            self.monitor_cpu_l1i = CommMonitor()
+            self.monitor_cpu_l1i.trace = MemTraceProbe(
+                trace_file="cpu_l1i_trace.tar.gz"
+            )
+            self.monitor_cpu_l1i.mem_side_port = ic.cpu_side
+
+            self.icache_port = self.monitor_cpu_l1i.cpu_side_port
+
+            self.monitor_cpu_l1d = CommMonitor()
+            self.monitor_cpu_l1d.trace = MemTraceProbe(
+                trace_file="cpu_l1d_trace.tar.gz"
+            )
+            self.monitor_cpu_l1d.mem_side_port = dc.cpu_side
+
+            self.dcache_port = self.monitor_cpu_l1d.cpu_side_port
+        else:
+            self.icache_port = ic.cpu_side
+            self.dcache_port = dc.cpu_side
+
         self._cached_ports = ["icache.mem_side", "dcache.mem_side"]
         if iwc and dwc:
             self.itb_walker_cache = iwc
```
#### ii. Modify `configs/common/CacheConfig.py`
```diff
     if options.memchecker:
         system.memchecker = MemChecker()
@@ -177,7 +186,7 @@ def config_cache(options, system):
             # When connecting the caches, the clock is also inherited
             # from the CPU in question
             system.cpu[i].addPrivateSplitL1Caches(
-                icache, dcache, iwalkcache, dwalkcache
+                icache, dcache, iwalkcache, dwalkcache, options.trace_monitor
             )
```

### d. Other
In `src/mem/probes/MemTraceProbe.py`, set with_pc = True
```diff
@@ -47,7 +47,7 @@ class MemTraceProbe(BaseMemProbe):
     trace_compress = Param.Bool(True, "Enable trace compression")
 
     # For requests with a valid PC, include the PC in the trace
-    with_pc = Param.Bool(False, "Include PC info in the trace")
+    with_pc = Param.Bool(True, "Include PC info in the trace")
 
     # packet trace output file, disabled by default
     trace_file = Param.String("", "Packet trace output file")
```

## 1.3 Generate Memory Traces
Run gem5 with --trace-monitor
```
gem5_args="--num-cpus=1 --caches --l2cache --cpu-type=DerivO3CPU \
        --mem-type=SimpleMemory \
        --mem-size 8GB --cacheline_size=64 \
        --l1i_size=64kB --l1i_assoc=8 \
        --l1d_size=64kB --l1d_assoc=8 \
        --l2_size=1MB --l2_assoc=8 \
        --cpu-clock 100MHz \
        --sys-clock 100MHz \
        --bp-type=LTAGE \
        --trace-monitor "

```
It will output traces
![](attachments/Pasted%20image%2020230523214510.png)

To analysis these trace files, I wrote a new python script according to `util/decode_packet_trace.py`
```python
import os
import protolib
import subprocess
import sys
import matplotlib.pyplot as plt

util_dir = os.path.dirname(os.path.realpath(__file__))
# Make sure the proto definitions are up to date.
subprocess.check_call(["make", "--quiet", "-C", util_dir, "packet_pb2.py"])
import packet_pb2

miss_count = {}

def main():
    if len(sys.argv) != 3:
        print("Usage: ", sys.argv[0], " <protobuf input> <ASCII output>")
        exit(-1)

    # Open the file in read mode
    proto_in = protolib.openFileRd(sys.argv[1])

    try:
        ascii_out = open(sys.argv[2], "w")
    except IOError:
        print("Failed to open ", sys.argv[2], " for writing")
        exit(-1)

    # Read the magic number in 4-byte Little Endian
    magic_number = proto_in.read(4).decode()

    if magic_number != "gem5":
        print("Unrecognized file", sys.argv[1])
        exit(-1)

    print("Parsing packet header")

    # Add the packet header
    header = packet_pb2.PacketHeader()
    protolib.decodeMessage(proto_in, header)

    print("Object id:", header.obj_id)
    print("Tick frequency:", header.tick_freq)

    for id_string in header.id_strings:
        print("Master id %d: %s" % (id_string.key, id_string.value))

    print("Parsing packets")

    num_packets = 0
    packet = packet_pb2.Packet()

    # Decode the packet messages until we hit the end of the file
    no_pc_counter = 0
    has_pc_counter = 0
    while protolib.decodeMessage(proto_in, packet):
        num_packets += 1

        if packet.HasField("pc") and packet.cmd == 4:
            # print('pc 0x%x' % (packet.pc))
            if packet.pc in miss_count:
                miss_count[packet.pc] += 1
            else:
                miss_count[packet.pc] = 1
            has_pc_counter = has_pc_counter + 1
        else:
            no_pc_counter += 1
    print("no pc: ", no_pc_counter)
    print("has pc: ", has_pc_counter)
    print("Parsed packets:", num_packets)
    sorted_by_pc = sorted(miss_count.items(), key=lambda x: x[1])
    sorted_miss = dict(sorted_by_pc)

    ascii_out.close()
    proto_in.close()

    miss_count_tmp = dict(
        (k, v) for (k, v) in sorted_miss.items() if v > 60000
    )
    miss_key_tmp = list(miss_count_tmp.keys())
    miss_key = list(map(lambda x: hex(x).split("x")[1].zfill(5), miss_key_tmp))
    x = []
    for i in range(len(miss_key)):
        x.append(i)
    miss_val = list(miss_count_tmp.values())

    plt.rcParams["font.size"] = 8
    bars = plt.bar(x, miss_val)
    plt.ylim(-10, 700000)
    plt.xticks(x, miss_key, rotation=90)
    plt.xlabel("PC")
    plt.ylabel("Count")
    plt.title("CPU->L1d Inst PC")
    plt.savefig("cpu_l1d.png", dpi=1000, bbox_inches="tight")

if __name__ == "__main__":
    main()
```
A figure depicting the frequency of CPU-to-L1D instructions will be generated.
![](attachments/Pasted%20image%2020230523214750.png)

# 2. Add ArchDB Support
Todo
You can refer to Xiangshan Gem5

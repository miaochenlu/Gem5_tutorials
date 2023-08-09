# Gem5 Cache Latency
---
# 1. Calculate Gem5 Cache Read Latency

The following is the cache read hit latency calculated through the cache latency settings in Gem5's configs/common/Caches.py and the latency settings in src/mem/XBar.py.

```python
class L1_DCache(L1Cache):
    ...
    tag_latency = 1
    data_latency = 1
    sequential_access = False
    response_latency = 4
```

| L1 Read Hit Latency  |     |
| -------------------- | --- |
| memory opLat         | 2   |
| max(L1 Tag, L1 Data) | 1   |
| total                | 3   |

```python
class L2Cache(Cache):
    tag_latency = 2
    data_latency = 13
    sequential_access = True

    # This is communication latency between l2 & l3
    response_latency = 4
```

| L2 Read Hit Latency |     |
| ------------------- | --- |
| memory opLat        | 2   |
| L1 Tag              | 1   |
| L2Xbar Frontend     | 1   |
| L2XBar Forward      | 0   |
| Snoop Filter Lookup | 0   |
| L2 Tag              | 2   |
| L2 Data             | 13   |
| L2Xbar response     | 1   |
| L1 reponse          | 4   |
| total               | 24  |

 ```python
 class L3Cache(Cache):
    # aligned latency:
    tag_latency = 2
    data_latency = 12
    sequential_access = True
    # This is L3 miss latency, which act as padding for memory controller
    response_latency = 112
```

| LLC Read Hit Latency |     |
| -------------------- | --- |
| memory opLat         | 2   |
| L1 Tag               | 1   |
| L2Xbar Frontend      | 1   |
| L2XBar Forward       | 0   |
| Snoop Filter Lookup  | 0   |
| L2 Tag               | 2   |
| L3Xbar Frontend      | 1   |
| L3XBar Forward       | 0   |
| Snoop Filter Lookup  | 0   |
| L3 Tag               | 2   |
| L3 Data              | 12   |
| L3Xbar response      | 1   |
| L2 response          | 4   |
| L2Xbar response      | 1   |
| L1 Response          | 4   |
| total                | 31  |


# 2. Cache Latency Test

To test cache latency through programmatic access, you can use the provided test program from this GitHub repository: https://github.com/OpenXiangShan/nexus-am/tree/gem5_benchmark/apps/cachetest_i

## 2.1 Gem5 Latency

When you compile the program in the maprobe directory using the command make ARCH=riscv64-xs, it will generate the maprobe-riscv64-xs.bin binary file. Run this binary in fs mode.

command line:
```shell
./build/RISCV/gem5.fast ./configs/example/fs.py
	--num-cpus=1 --caches --l2cache --cpu-type=DerivO3CPU 
	--mem-type=DRAMsim3 --dramsim3-ini=xiangshan_DDR4_8Gb_x8_2400.ini --mem-size=8GB 
	--cacheline_size=64 --l1i_size=32kB --l1i_assoc=8 
	--l1d_size=128kB --l1d_assoc=8 
	--l2_size=512kB --l2_assoc=8 
	--l3cache --l3_size=2MB --l3_assoc=16 
	--bp-type=DecoupledBPUWithFTB 
	--generic-rv-cpt /path/to/maprobe-riscv64-xs.bin
	--xiangshan-system --raw-cpt
```

![](attachments/Pasted%20image%2020230707173544.png)

## 2.3 Xiangshan RTL Latency
* L1 lat: 3~5
* l2 lat: 24
* llc lat: 31
![](attachments/Pasted%20image%2020230707164754.png)

# 3. Analyzing the composition of Memory Latency through Gem5 code.

![](attachments/Pasted%20image%2020230613101614.png)

## 3.1. OPLat

```python
class ReadPort(FUDesc):
    opList = [ OpDesc(opClass='MemRead',opLat=2),
               OpDesc(opClass='FloatMemRead') ]
    count = 2
```

Here, the opLat for MemRead is set to 2.

In the provided link from the Gem5 users forum, there is an explanation of opLat. https://gem5-users.gem5.narkive.com/YU5VU2Oo/o3-timing 

> My understanding is that issueLat is the minimum number of cycles you have to wait before scheduling an instruction of the same type on the FU. Specifically, if opLat is 6 and issueLat is 1, you have a pipelined unit with a latency of 6 cycles but you have a throughput of 1 op/cycle once the pipe is full.

## 3.2. Cache

### 3.2.1. Latency

```cpp
/**
 * The latency of tag lookup of a cache. It occurs when there is
 * an access to the cache.
 */
const Cycles lookupLatency;

/**
 * The latency of data access of a cache. It occurs when there is
 * an access to the cache.
 */
const Cycles dataLatency;

/**
 * This is the forward latency of the cache. It occurs when there
 * is a cache miss and a request is forwarded downstream, in
 * particular an outbound miss.
 */
const Cycles forwardLatency;

/** The latency to fill a cache block */
const Cycles fillLatency;

/**
 * The latency of sending reponse to its upper level cache/core on
 * a linefill. The responseLatency parameter captures this
 * latency.
 */
const Cycles responseLatency;
```

Another important parameter
```cpp
/**
 * Whether tags and data are accessed sequentially.
 */
const bool sequentialAccess;
```

If sequentialAccess is set to true, then the lat (latency) calculation is lookupLatency + dataLatency.

If sequentialAccess is set to false, then the lat calculation is max(lookupLatency, dataLatency).

During initialization, both lookupLatency and forwardLatency are set to tagLatency.

```cpp
BaseCache::BaseCache(const BaseCacheParams &p, unsigned blk_size)
    : ...
      lookupLatency(p.tag_latency),
      dataLatency(p.data_latency),
      forwardLatency(p.tag_latency),
      fillLatency(p.data_latency),
      responseLatency(p.response_latency),
      sequentialAccess(p.sequential_access),
{ ... }
```

### 3.2.2. recvTimingReq -- tagLatency/dataLatency

![](attachments/Pasted%20image%2020230613113118.png)

### 3.2.3. recvTimingResp -- responseLatency

![](attachments/Pasted%20image%2020230612142223.png)
## 3.3. XBar Latency

### 3.3.1. Latency

```python
# A request incurs the frontend latency, possibly snoop filter
# lookup latency, and forward latency. A response incurs the
# response latency. Frontend latency encompasses arbitration and
# deciding what to do when a request arrives. the forward latency
# is the latency involved once a decision is made to forward the
# request. The response latency, is similar to the forward
# latency, but for responses rather than requests.
frontend_latency = Param.Cycles("Frontend latency")
forward_latency = Param.Cycles("Forward latency")
response_latency = Param.Cycles("Response latency")
```
* frontend_latency: Frontend latency encompasses arbitration and deciding what to do when a request arrives.
* forward_latency: The forward latency is the latency involved once a decision is made to forward the request.
* response_latency: The response latency, is similar to the forward latency, but for responses rather than requests.

### 3.3.2. recvTimingReq -- frontendLatency + forwardLatency + Snoop Filter lookupLatency

```cpp
// a request sees the frontend and forward latency
Tick xbar_delay = (frontendLatency + forwardLatency) * clockPeriod();

// set the packet header and payload delay
calcPacketTiming(pkt, xbar_delay);
```

In the `calcPacketTiming` function, the `xbar_delay` will be placed into the `headerDelay` of the packet.
```cpp
Tick offset = clockEdge() - curTick();

// the header delay depends on the path through the crossbar, and
// we therefore rely on the caller to provide the actual value
pkt->headerDelay += offset + header_delay;
```

Next, a lookup will be performed in the snoop filter, and the lookup latency of the snoop filter needs to be recorded.
```cpp
if (snoopFilter) {
	// check with the snoop filter where to forward this packet
	auto sf_res = snoopFilter->lookupRequest(pkt, *src_port);
	// the time required by a packet to be delivered through
	// the xbar has to be charged also with to lookup latency
	// of the snoop filter
	pkt->headerDelay += sf_res.second * clockPeriod();
	...
}
```

Send the request out.
```cpp
success = memSidePorts[mem_side_port_id]->sendTimingReq(pkt);
```

The packet (`pkt`) records the required latency on the crossbar (`xbar`) in the `headerDelay`, which will be calculated in the `recvTimingReq` function of the cache.

### 3.3.3. recvTimingResp -- responseLatency

```cpp
// a response sees the response latency
Tick xbar_delay = responseLatency * clockPeriod();

// set the packet header and payload delay
calcPacketTiming(pkt, xbar_delay);
```

send out response
```cpp
// send the packet through the destination CPU-side port and pay for
// any outstanding header delay
Tick latency = pkt->headerDelay;
pkt->headerDelay = 0;
cpuSidePorts[cpu_side_port_id]->schedTimingResp(pkt, curTick()+ latency);
```


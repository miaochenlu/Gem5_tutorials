# TraceCPU & Elastic Trace

![](attachments/Pasted%20image%2020230605164701.png)
# 1. How to Use
Follow gem5 tracecpu tutorial https://www.gem5.org/documentation/general_docs/cpu_models/TraceCPU

## 1.1 Generate Elastic Trace
- SE mode
```shell
build/ARM/gem5.opt [gem5.opt options] -d bzip_10Minsts configs/example/se.py [se.py options] --cpu-type=arm_detailed --caches --cmd=$M5_PATH/binaries/arm_arm/linux/bzip2 --options=$M5_PATH/data/bzip2/lgred/input/input.source -I 10000000 --elastic-trace-en --data-trace-file=deptrace.proto.gz --inst-trace-file=fetchtrace.proto.gz --mem-type=SimpleMemory
```
- FS mode: Create a checkpoint for your region of interest and resume from the checkpoint but with O3 CPU model and tracing enabled
```shell
build/ARM/gem5.opt --outdir=m5out/bbench/capture_10M ./configs/example/fs.py [fs.py options] --cpu-type=arm_detailed --caches --elastic-trace-en --data-trace-file=deptrace.proto.gz --inst-trace-file=fetchtrace.proto.gz --mem-type=SimpleMemory --checkpoint-dir=m5out/bbench -r 0 --benchmark bbench-ics -I 10000000
```

Note: Don't configure l2cache; Use SimpleMemory

## 1.2 Replay Elastic Trace
- A trace replay script in the examples folder can be used to play back SE and FS generated traces 
```shell
build/ARM/gem5.opt [gem5.opt options] -d bzip_10Minsts_replay configs/example/etrace_replay.py [options] --cpu-type=TraceCPU --caches --data-trace-file=bzip_10Minsts/deptrace.proto.gz --inst-trace-file=bzip_10Minsts/fetchtrace.proto.gz --mem-size=4GB
```

## 1.3 Decode Elastic Trace
```shell
gem5$ util/decode_inst_trace.py script/test/system.cpu.traceListener.fetchtrace.proto.gz fetch.txt
Parsing instruction header
Object id: system.cpu.traceListener
Tick frequency: 1000000000000
Memory addresses included: False
Parsing instructions
Parsed instructions: 292

gem5$ util/decode_inst_dep_trace.py script/test/system.cpu.traceListen
er.deptrace.proto.gz data.txt
Parsing packet header
Object id: system.cpu.traceListener
Tick frequency: 1000000000000
Parsing packets
Creating enum value,name lookup from proto
         0 INVALID
         1 LOAD
         2 STORE
         3 COMP
Parsed packets: 6153
Packets with at least 1 reg dep: 6101
Packets with at least 1 rob dep: 107
```

Note:![](attachments/Pasted%20image%2020230605164657.png)
Change
```python
    if magic_number != "gem5":
        print("Unrecognized file", sys.argv[1])
        exit(-1)
```
to
```python
    if magic_number != b"gem5":
        print("Unrecognized file", sys.argv[1])
        exit(-1)
```
# 2. Internal Codes to Generate Traces

The general workflow of ElasticTrace in Gem5 can be described as follows:
1. Understanding ProbePoint, ProbeListener, and ProbeManager:
	- ProbeManager acts as a singleton to manage ProbePoints and ProbeListeners.
	- ProbePoints represent specific events, while ProbeListeners choose to listen to specific events.
	- When an event occurs, the ProbeListener's notify function is called to perform certain actions, such as generating a trace.
2. O3CPU's involvement with ProbeManager:
	- O3CPU (Out-of-Order CPU) has a ProbeManager instance.
	- Each crucial class in O3CPU has a pointer to the O3CPU instance, allowing access to the ProbeManager.
	- O3CPU uses the regProbePoint function to register ProbePoints for events such as data access, instruction access, and specific stages like fetch, rename, iew, and commit.
3. Configuring ElasticTrace:
	- During the configuration phase, an elastic trace option is provided.
	- If elastic trace is enabled, the CPU will have a traceListener, which is an ElasticTrace object.
	- When initializing ElasticTrace, it registers a series of listeners and defines their callback functions to be invoked during notification.

Overall, ElasticTrace in Gem5 works by utilizing ProbePoints and ProbeListeners managed by the ProbeManager. O3CPU registers ProbePoints for various events, and if elastic trace is enabled, the CPU's traceListener, an ElasticTrace object, generates traces based on the notifications received from the registered ProbeListeners.

![](attachments/Pasted%20image%2020230605160642.png)


# 3. TraceCPU
`cpu/trace/trace.hh`
* TraceCPU::schedIcacheNext()
* TraceCPU::schedDcacheNext()

## 3.1 Init

* `trace_cpu.cc / TraceCPU::init()`
Initialize icache & dcache access generator
```cpp
    // Get the send tick of the first instruction read request
    Tick first_icache_tick = icacheGen.init();

    // Get the send tick of the first data read/write request
    Tick first_dcache_tick = dcacheGen.init();

	// Set the trace offset as the minimum of that in both traces
    traceOffset = std::min(first_icache_tick, first_dcache_tick);

    // Schedule next icache and dcache event by subtracting the offset
    schedule(icacheNextEvent, first_icache_tick - traceOffset);
    schedule(dcacheNextEvent, first_dcache_tick - traceOffset);


    // Adjust the trace offset for the dcache generator's ready nodes
    // We don't need to do this for the icache generator as it will
    // send its first request at the first event and schedule subsequent
    // events using a relative tick delta
    dcacheGen.adjustInitTraceOffset(traceOffset);
```
Choose the smaller one as offset.
```cpp
traceOffset = std::min(first_icache_tick, first_dcache_tick);
// Schedule next icache and dcache event by subtracting the offset
schedule(icacheNextEvent, first_icache_tick - traceOffset);
schedule(dcacheNextEvent, first_dcache_tick - traceOffset);

```
Schedule Cache event

```cpp
void
schedule(Event &event, Tick when)
{
    eventq->schedule(&event, when);
}
```


## 3.2. ICache


![](attachments/Pasted%20image%2020230605160605.png)

focus this line `Tick first_icache_tick = icacheGen.init();`

* `trace_cpu / TraceCPU::FixedRetryGen::init()`

```cpp
Tick
TraceCPU::FixedRetryGen::init() {
    if (nextExecute()) {
        return currElement.tick;
    } else {
        return MaxTick;
    }
}
```

* `trace_cpu / TraceCPU::FixedRetryGen::nextExecute()`
	* This function reads one trace
```cpp
bool
TraceCPU::FixedRetryGen::nextExecute() {
    if (traceComplete)
        // We are at the end of the file, thus we have no more messages.
        // Return false.
        return false;


    //Reset the currElement to the default values
    currElement.clear();

    // Read the next line to get the next message. If that fails then end of
    // trace has been reached and traceComplete needs to be set in addition
    // to returning false. If successful then next message is in currElement.
    if (!trace.read(&currElement)) {
        traceComplete = true;
        fixedStats.instLastTick = curTick();
        return false;
    }
    return true;
}
```

## 3.3. DCache

![](attachments/Pasted%20image%2020230605161828.png)


focus on this line `Tick first_dcache_tick = dcacheGen.init();`
* `trace_cpu / TraceCPU::ElasticDataGen::init()`

```cpp
Tick
TraceCPU::ElasticDataGen::init() {
...

    // Print readyList
    if (debug::TraceCPUData) {
        printReadyList();
    }
    auto free_itr = readyList.begin();
    // Return the execute tick of the earliest ready node so that an event
    // can be scheduled to call execute()
    return (free_itr->execTick); // the first free element exec tick
}
```



```cpp
bool
TraceCPU::FixedRetryGen::tryNext()
{
    // If there is a retry packet, try to send it
    if (retryPkt) {
        if (!port.sendTimingReq(retryPkt)) {
            // Still blocked! This should never occur.
            return false;
        }
        ++fixedStats.numRetrySucceeded;
    } else {
        // try sending current element
        assert(currElement.isValid());

        ++fixedStats.numSendAttempted;

        if (!send(currElement.addr, currElement.blocksize,
                    currElement.cmd, currElement.flags, currElement.pc)) {
            ++fixedStats.numSendFailed;
            // return false to indicate not to schedule next event
            return false;
        } else {
            ++fixedStats.numSendSucceeded;
        }
    }
    // If packet was sent successfully, either retryPkt or currElement, return
    // true to indicate to schedule event at current Tick plus delta. If packet
    // was sent successfully and there is no next packet to send, return false.
    retryPkt = nullptr;
    // Read next element into currElement, currElement gets cleared so save the
    // tick to calculate delta
    Tick last_tick = currElement.tick;
    if (nextExecute()) {
        assert(currElement.tick >= last_tick);
        delta = currElement.tick - last_tick; // modify delta
    }
    return !traceComplete;
}
```


```cpp
void
TraceCPU::schedDcacheNext() {
    // Update stat for numCycles
    baseStats.numCycles = clockEdge() / clockPeriod();

    dcacheGen.execute();
    if (dcacheGen.isExecComplete()) {
        checkAndSchedExitEvent();
    }
}

```

ready list is sorted according to exec tick

```cpp
TraceCPU::ElasticDataGen::addToSortedReadyList(NodeSeqNum seq_num,
                                               Tick exec_tick) {
    ReadyNode ready_node;
    ready_node.seqNum = seq_num;
    ready_node.execTick = exec_tick;

    // Iterator to readyList
    auto itr = readyList.begin();

    // If the readyList is empty, simply insert the new node at the beginning
    // and return
    if (itr == readyList.end()) {
        readyList.insert(itr, ready_node);
        elasticStats.maxReadyListSize =
            std::max<double>(readyList.size(),
                             elasticStats.maxReadyListSize.value());
        return;
    }

    // If the new node has its execution tick equal to the first node in the
    // list then go to the next node. If the first node in the list failed
    // to execute, its position as the first is thus maintained.
    if (retryPkt) {
        if (retryPkt->req->getReqInstSeqNum() == itr->seqNum)
            itr++;
    }

    // Increment the iterator and compare the node pointed to by it to the new
    // node till the position to insert the new node is found.
    bool found = false;
    while (!found && itr != readyList.end()) {
        // If the execution tick of the new node is less than the node then
        // this is the position to insert
        if (exec_tick < itr->execTick) {
            found = true;
        // If the execution tick of the new node is equal to the node then
        // sort in ascending order of sequence numbers
        } else if (exec_tick == itr->execTick) {
            // If the sequence number of the new node is less than the node
            // then this is the position to insert
            if (seq_num < itr->seqNum) {
                found = true;
            // Else go to next node
            } else {
                itr++;
            }
        } else {
            // If the execution tick of the new node is greater than the node
            // then go to the next node.
            itr++;
        }
    }
    readyList.insert(itr, ready_node);
    // Update the stat for max size reached of the readyList
    elasticStats.maxReadyListSize = std::max<double>(readyList.size(),
                                        elasticStats.maxReadyListSize.value());
}

```
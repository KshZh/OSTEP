作业给出的是一个模拟进程调度和运行的程序，程序中的scheduler.run模拟一个CPU，每个时间单元运行当前进程的一条指令，指令有`DO_COMPUTE`和`DO_IO`两种，分别占用和不占用CPU（假设系统采用DMA硬件加中断的方式处理IO）。另外，run硬编码了从进程表中的第一个进程开始执行。程序可配置的主要参数有：

- `-S PROCESS_SWITCH_BEHAVIOR`，两种选项分别为`SWITCH_ON_IO`和`SWITCH_ON_END`，前者当当前进程发起IO请求或运行结束时切换进程，后者只在运行结束时才切换进程，即在该选项下，一个进程一旦被调度运行，那么会一直运行到结束，即时被阻塞等待IO，调度器也不会调度其他进程来运行。默认是前者。
- `-I IO_DONE_BEHAVIOR`，两种选项分别为`IO_RUN_IMMEDIATE`和`IO_RUN_LATER`，前者在一个进程IO完成时马上被调度运行，后者相反，但如果`-S`参数值为`SWITCH_ON_END`，那么也会马上被调度运行。默认是后者。
- `-L IO_LENGTH`，默认是5个时间单位。
- `-c                    compute answers for me`。
- `-p, --printstats      print statistics at end; only useful with -c flag`。

在选择其他ready进程运行时，调度器简单地从当前进程开始循环遍历进程表，选择遇到的第一个ready进程运行。

Questions

1. Run process-run.py withthefollowingﬂags: -l 5:100,5:100. What should the CPU utilization be?

   两个进程，一直都在做CPU计算，故CPU利用率为100%。

2. Now run with these ﬂags: ./process-run.py -l 4:100,1:0. These ﬂags specify one process with 4 instructions (all to use the CPU), and one that simply issues an I/O and waits for it to be done. How long does it take to complete both processes?

   因为run硬编码了从进程表中的第一个进程开始执行，所以进程1的全部CPU计算执行完后，再执行进程2。进程1花费4个时间单位，进程2执行发起IO请求的指令花费1个时间单位，IO花费5个时间单位，共10个时间单位。

3. Switch the order of the processes: -l 1:0,4:100. What happens now? Does switching the order matter? Why? 

   默认选项是`SWITCH_ON_IO`，所以在时间点1时，进程1发起IO请求后被阻塞/挂起，于是从时间点2开始，进程2开始执行，共花费(1+5)个时间单位。

4. We’ll now explore some of the other ﬂags. One important ﬂag is -S, which determines how the system reacts when a process issues an I/O. With the ﬂag set to SWITCH ON END, the system will NOT switch to another process while one is doing I/O, instead waiting until the process is completely ﬁnished. What happens when you run the following two processes (-l 1:0,4:100 -c -S SWITCH_ON_END), one doing I/O and the other doing CPU work? 

   等待IO完成的进程被阻塞/挂起，但这段时间内OS并没有切换到别的进程运行来利用空闲的CPU。

5. Now, run the same processes, but with the switching behavior set to switch to another process whenever one is WAITING for I/O (-l 1:0,4:100 -c -S SWITCH ON IO). What happens now? Use -c and -p to conﬁrm that you are right. 

   空闲的CPU被利用起来跑别的进程，CPU利用率提高，花费的时间减少了。

6. One other important behavior is what to do when an I/O completes. With -I IO_RUN_LATER, when an I/O completes, the process that issued it is not necessarily run right away; rather, whatever was running at the time keeps running. What happens when you run this combination of processes? (Run ./process-run.py -l 3:0,5:100,5:100,5:100 -S SWITCH_ON_IO -I IO_RUN_LATER -c -p) Are system resources being effectively utilized? 

   ```
   Time     PID: 0     PID: 1     PID: 2     PID: 3        CPU        IOs 
     1      RUN:io      READY      READY      READY          1            
     2     WAITING    RUN:cpu      READY      READY          1          1 
     3     WAITING    RUN:cpu      READY      READY          1          1 
     4     WAITING    RUN:cpu      READY      READY          1          1 
     5     WAITING    RUN:cpu      READY      READY          1          1 
     6*      READY    RUN:cpu      READY      READY          1            
     7       READY       DONE    RUN:cpu      READY          1            
     8       READY       DONE    RUN:cpu      READY          1            
     9       READY       DONE    RUN:cpu      READY          1            
    10       READY       DONE    RUN:cpu      READY          1            
    11       READY       DONE    RUN:cpu      READY          1            
    12       READY       DONE       DONE    RUN:cpu          1            
    13       READY       DONE       DONE    RUN:cpu          1            
    14       READY       DONE       DONE    RUN:cpu          1            
    15       READY       DONE       DONE    RUN:cpu          1            
    16       READY       DONE       DONE    RUN:cpu          1            
    17      RUN:io       DONE       DONE       DONE          1            
    18     WAITING       DONE       DONE       DONE                     1 
    19     WAITING       DONE       DONE       DONE                     1 
    20     WAITING       DONE       DONE       DONE                     1 
    21     WAITING       DONE       DONE       DONE                     1 
    22*     RUN:io       DONE       DONE       DONE          1            
    23     WAITING       DONE       DONE       DONE                     1 
    24     WAITING       DONE       DONE       DONE                     1 
    25     WAITING       DONE       DONE       DONE                     1 
    26     WAITING       DONE       DONE       DONE                     1 
    27*       DONE       DONE       DONE       DONE                       
   
   Stats: Total Time 27
   Stats: CPU Busy 18 (66.67%)
   Stats: IO Busy  12 (44.44%)
   ```

   可以看到，在这种情况下，这种调度并不能很好地利用CPU和外设，如果是`IO_RUN_IMMEDIATE`，那么就可以让IO完成的进程紧接着发起下一个IO请求，然后被阻塞/挂起期间，仍调度别的进程使用CPU。

7. Now run the same processes, but with -I IO_RUN_IMMEDIATE set, which immediately runs the process that issued the I/O. How does this behavior differ? Why might running a process that just completed an I/O again be a good idea?

   **关键就是要让CPU和外设尽可能忙起来，最好不放过任何一个时间单位使它们并行地工作**。满足这一条件的调度往往更高效。

8. Now run with some randomly generated processes: -s 1 -l 3:50,3:50 or -s 2 -l 3:50,3:50 or -s 3 -l 3:50,3:50. See if you can predict how the trace will turn out. What happens when you use the ﬂag -I IO_RUN_IMMEDIATE vs. -I IO_RUN_LATER? What happens when you use -S SWITCH_ON_IO vs. -S SWITCH_ON_END?

   自己试试看就行了。我感觉，综合上面的问题和答案，**往往`SWITCH_ON_IO`和`IO_RUN_IMMEDIATE`是更高效的选择**。（当然，真正的OS不会等到当前进程发起IO或全部执行完才调度其他进程执行，往往每个进程都有一个时间片，不会出现某个进程长期占用CPU，除非CPU上只有一两个runnable/ready进程，但进程被调度一次还是运行一个时间片，只是频繁被选中执行）

**虽然并发（不是并行）有上下文切换的开销，但串行/单线性也不见得CPU有效利用率高，从上面的问题和答案可以看出，当等待I/O时，CPU就闲下来了，而并发就可以在这段时间内把CPU利用起来**。

**当然线程不是越多越好，主要看应用场景。如果是重计算的话，那么做计算的线程数要和CPU数匹配，否则会有额外的线程切换开销，伴随着缓存失效等降低效率的问题；如果是重IO的话，IO线程才可以远多于CPU数量，毕竟IO线程发起阻塞IO时会被阻塞/挂起（使用非阻塞IO API则线程不会被阻塞/挂起），并不消耗CPU/计算资源**。
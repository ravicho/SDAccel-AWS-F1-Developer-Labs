# Host Code Performance Optimizations and Wrap Up

  - [Select Kernel for Execution](#select-kernel-for-execution)
  - [Launch Application and Generate Reports](#launch-application-and-generate-reports)
  - [Application Performance Analysis](#application-performance-analysis)
    - [Profile Summary: Compute Unit Utilization](#profile-summary-compute-unit-utilization)
    - [Application Timeline Analysis](#application-timeline-analysis)
    - [Task Dependency Analysis](#task-dependency-analysis)
  - [Optimizing Host Application Code for Performance](#optimizing-host-application-code-for-performance)
    - [Capturing Task Dependencies](#capturing-task-dependencies)
    - [Batching and Pool of Buffers](#batching-and-pool-of-buffers)
      - [Duplicating Input and Output Buffers](#duplicating-input-and-output-buffers)
      - [Multiple Command Queues](#multiple-command-queues)
    - [Running Application with Software Pipeline](#running-application-with-software-pipeline)
    - [Running Application with Software Pipeline and Kernel With II=4](#running-application-with-software-pipeline-and-kernel-with-ii4)
  - [Estimated Performance versus Measured Performance](#estimated-performance-versus-measured-performance)
  - [Summary](#summary)
  - [Stopping your instance](#stopping-your-instance)
  - [Congratulations!](#congratulations)
  - [Next steps](#next-steps)
  
This lab discusses host code optimizations that can result in big performance improvements. This lab will wrap up module by showing different steps on how to close RDP session and stop AWS F1 instance. It is important to always stop or terminate AWS EC2 instances when you are done using them. This is a recommended best practice to avoid unwanted charges.

In the following section, you will launch the application on F1 instance and analyze application execution using timeline.

## Select Kernel for Execution

1. Modify host code to make sure it runs appropriate hardware kernel. Open terminal if it is not already open from last lab and do:

    ```bash
    source $AWS_FPGA_REPO_DIR/vitis_setup.sh
    export LAB_WORK_DIR=/home/centos/src/project_data/
    cd $LAB_WORK_DIR/Vitis-AWS-F1-Developer-Labs/modules/module_01/idct
    ```

    Now open host.cpp and make changes so that hardware run will use kernel namely "krnl_idct": 

    ```bash
    vim src/host.cpp
    ```  

    Go to label "CREATE_KERNEL" near line 226 and make sure the kernel name string is **"krnl_idct"** and not anything else and compile host application again:

    ```bash
    make compile_host TARGET=hw
    ```
## Launch Application and Generate Reports

1. Run application as follows by first sourcing runtime setup script if it was not done before:

    ```bash
    source $AWS_FPGA_REPO_DIR/vitis_runtime_setup.sh 
    cd $LAB_WORK_DIR/Vitis-AWS-F1-Developer-Labs/modules/module_01/idct
    ./build/host.exe ./xclbin/krnl_idct.hw.awsxclbin $((1024*128)) 32 1
    ```
   it will show an output similar to the one given below with an acceleration about 11x faster than CPU
   
   ```
    Execution Finished
    =====================================================================
    ------ All Task finished !
    ------ Done with CPU and FPGA based IDCTs
    ------ Runs complete validating results
    CPU Time:        1.86819 s ( 1868.19ms )
    CPU Throughput:  274.062 MB/s
    FPGA Time:       0.166588 s (166.588 ms )
    FPGA Throughput: 3073.45 MB/s
    ------ TEST PASSED ------
    =====================================================================
    FPGA accelerations ( CPU Exec. Time / FPGA Exec. Time): 11.2145
    =====================================================================
   ```
   The system run will also create a profile summary and application timeline in the run folder (idct here) open application timeline:
   
   ```bash
   vitis_analyzer ./xclbin.run_summary
   ```
   
   Now in the host side application timeline part go to first write transaction and zoom in appropriately and it will show a timeline similar to the one shown below:
   
      ![](images/module_01/lab_05_idct/hwRunNoSwPipeLine.PNG) 
      
  In next section, we will try to comprehend this waveform and also perform host side code optimization to improve performance.

## Application Performance Analysis

### Profile Summary: Compute Unit Utilization

Open the profile summary and by looking at the profile summary and timeline, find potential for performance improvement. To open the profile summary, proceed as follows:

1. In the run directory (idct) open profile summary

    ```bash
   vitis_analyzer ./build/xclbin.run_summary
    ```
   From the left panel, select **"Profile Summary"** and from the profile summary view, go to **"Kernels and Compute Units"** and it will show numbers as shown in the figure below:
   
      ![](images/module_01/lab_05_idct/hwRunCuUtil.PNG)
      
  It shows that compute unit utilization is around 23%. Meaning compute unit is busy crunching numbers only for 23% of the time and remaining 77% of time it is stalled/idling. This can be confirmed from the application timeline which also provides insight why it is so. 
  
### Application Timeline Analysis 
 
 From the waveform, one can observe that:
 
 *  Host sends data through PCI on write interface for processing and only once the data transfer completes the enqueued kernel can kick starts compute unit.
 
 * Once the compute unit finishes processing current batch of input data only then output data produced by kernel requested by host using enqueue command start to flow.
 
 * Whole of this cycle then repeats till all batches of input are processed.
     
 One very important thing to note is that host does all of these transactions sequentially, only initiating new transaction (task getting enqueued on command queue) after previous one is finished. Given that most of the PCI and DDR interfaces have support for full duplex transports, one can preform a number of optimizations using the dependency analysis.
 
 ### Task Dependency Analysis
  
  - **Input Data:** The host side writes to device memory can continue in background back to back since it is input data coming from host to device for processing and has no dependencies
  
  - **Kernel Compute:** Kernel can be enqueued with dependency on corresponding input data block and can kick start as soon as this data block is transferred to device
  
  - **Output Data Transfer to Host:** Next Kernels Compute has no dependency on output data block calculated by current kernel call and which will be transferred back to host. The output data transfer for current compute can continue in background while next kernel compute can start.
 
 In nutshell this dependency analysis clarifies and reveals that:
  - While kernel is processing current data block host can start sending next input data block for next kernel compute
  - kernel/CU can start processing next input data block as soon as it becomes available
  - the output data transfer to host (produced by kernel/CU execution) can start as soon as current kernel/CU call finishes and continue in background while kernel/CU starts processing next input block. 
  
Once these fact have been identified, one can capture relevant and required dependencies on host side and improve the performance considerably.

## Optimizing Host Application Code for Performance

This section will try to dissect the application host code and explain how it is structured and which kind of objects are used to capture the behavior and dependencies, as discussed in the last section, to gain maximum performance.

### Capturing Task Dependencies
 
This section describes how the host code captures dependencies.

1. Open application source code in file "idct/krnl_idct_wrapper.cpp"

    ```bash
    vim idct/src/krnl_idct_wrapper.cpp
    ```
    The host code that deals with FPGA accelerator and coordinates most of its activity is organized in two different files as explained in one of the previous labs also, namely:
     * "host.cpp" and 
     * "krnl_idct_wrapper.cpp".
      
    The code in "host.cpp" deals with the creation of objects such as command queue, context, kernel and loads FPGA with binary that contains compiled kernels. The piece of host code inside "krnl_idct_wrapper.cpp" is organized in a function body to make it callable inside a std::thread.
     
2.  After opening "krnl_idct_wrapper.cpp" go to label "EVENTS_AND_WAIT_LISTS" near line 48. Here, event vectors are defined that will hold events generated by different tasks (read/write/kernel execute) when they complete. How the lengths of vectors is chosen will be explained in next section. These events help link different tasks by letting us define one event generated by completion of on task as dependency or blocking condition for another task.

    - **writeToDeviceEvent** is a vector which holds events which are generated once the host finishes transfer of input data from host to device. These events can trigger enqueued kernel and hence start compute unit.
    
    - **kernelExecEvent** is vector of events which indicates kernel compute has finished.
     
    - **readFromDeviceEvent** represents events generated when host finishes reading output data from device memory.
       
### Batching and Pool of Buffers

#### Duplicating Input and Output Buffers

For using IDCT kernel three buffers are required one each for:
-input data
-input IDCT co-efficients, 
-and output data.

Since the host side is designed to make sure different operations can overlap, duplicate buffers for all inputs and outputs, which the kernel needs to process per call, are needed. The host application has a parameter called **maxScheduledBatches** which is used to define the level of duplicity. Essentially, it defines how many buffers for storing different inputs or outputs for multiple kernel calls are needed. The host application is written such that this parameter can be passed to host application as commandline argument.
 
 - **maxScheduledBatches = 1:**  Setting its value to 1 means that one buffer each is used for storing input and output blocks. Therefore multiple memory movement commands and kernel enqueues at the same time cannot be issued. So as such no overlapping of transactions can happen from the host side. So write to device, kernel execution and read back from device all happen sequentially.
 
  - **maxScheduledBatches > 1:**  Setting it to a value greater than 1 means that there are duplicate resources (multiple set of input and output buffers) and can potentially enable overlapping transactions from host side.
   
  one can also observe that **maxScheduledBatches** is the parameter which sizes all other vectors such as event vectors and wait list. To understand how **maxScheduledBatches > 1** enable overlapping of different operations, define
  **Full Transaction** as set of the following enqueue operations from host for single kernel call:
   
  * write input data to device (enqueueMigrateMemObjects)
  * execute kernel (enqueueTask)
  * read output data from device (enqueueMigrateMemObjects)
  
  With multiple buffers available for each set of input and output data (maxScheduledBatches > 1), one can conceptually enqueue multiple full transactions on a command queue at a given time which will potentially set stage for overlapping between execution of different full transactions (i.e. write for first read for next, execute for first write for next etc.).
 
 #### Multiple Command Queues
 
  Given the potential created by **maxScheduledBatches > 1**, to achieve the overlapping transactions host uses three different in order command queues:

  - One is used to enqueue all the tasks that move input data to device
  - Another is used to enqueue all kernel executions
  - Third one is used to enqueue all device to host output data movements
  
  The in-order queues are used to make sure that data dependencies between multiple similar tasks (like moving input data for second kernel call should not happen before input data for 1st kernel call has finished since it may create contention at memory interface and lower performance) are defined implicitly.
  
  Now to understand main loop that does all task enqueueing and resource management please go to label "BATCH_PROCESSING_LOOP" near line 68 and observe the following:

  - This loop has total number of iterations equal to maximum number of batches to be processed (passed as host application command line argument)
  - One batch is processed by a single kernel call
  - Initially when this loop starts it will schedule multiple full transactions  equal to **maxScheduledBatches**, essentially starts using all the buffer resources allocated and defined by maxScheduledBatches > 1
  - After this it will wait for very first "full transaction" to complete by using a **earlyKernelEnqueue** pointer which always points to earliest enqueued "full transaction" still not finished ( meaning write to device, kernel execute and output data read back not complete).
  - Once the earliest enqueued "full transaction" is finished it will re-uses these buffers for enqueueing the next full transaction for next batch of input data.
  - This loop continues till all the batches are processed.
  
  After this loop finishes, wait on command queues for all the operations to finish.
  
  After all these details it should be clear that host code is designed to mimic full sequential host side execution by using **maxScheduledBatches=1** and overlapping/pipelined transactional behavior by using **maxScheduledBatches > 1**.
 
### Running Application with Software Pipeline

Now that it is known how to create a software pipeline on host side using pool of buffers and multiple command queues, following section proceeds to experiment with it and shows how performance changes.

1. Run the application again and note that the final commandline argument is **4** instead of **1** as was the case in previous run which will enable host side write/execute/read overlapping (software pipeline).

    ```bash
    cd $LAB_WORK_DIR/Vitis-AWS-F1-Developer-Labs/modules/module_01/idct
    ./build/host.exe ./xclbin/krnl_idct.hw.awsxclbin $((1024*128)) 32 4
    ```
    it will print an output message like the one shown below where **performance has almost improved by 2x or more**.
    
    ```
    Execution Finished
    =====================================================================
    ------ All Task finished !
    ------ Done with CPU and FPGA based IDCTs
    ------ Runs complete validating results
    CPU Time:        1.90758 s ( 1907.58ms )
    CPU Throughput:  268.403 MB/s
    FPGA Time:       0.0793128 s (79.3128 ms )
    FPGA Throughput: 6455.45 MB/s
    ------ TEST PASSED ------
    =====================================================================
    FPGA accelerations ( CPU Exec. Time / FPGA Exec. Time): 24.0513
    =====================================================================
    ```

2. To confirm that now read/write and execute transactions are overlapping (software pipeline is created) you can again open application timeline re-created by the last run, it should show timeline similar to the one shown in figure below:

    ```bash
    vitis_analyzer ./xclbin.run_summary
    ```
   ![](images/module_01/lab_05_idct/hwRunSwPipeLine.PNG)
    
3. Another interesting thing to look for is compute unit utilization:
*   Please open the profile summary and have a look at compute unit utilization
*   it should have gone up from 23% to something more than 70%

which means it is now idling considerably less than what it was before this software pipeline was created on the host side.
     
![](images/module_01/lab_05_idct/hwRunCuUtilOpt.PNG) 

### Running Application with Software Pipeline and Kernel With II=4

Now that the host side software pipeline is setup, which ensures that data movements between host and device are happening at full throughput in an overlappping fashion, go back and try to choose a kernel that has II=4 and verify if the performance estimation that was done in the previous lab comes close to what was predicted. The prediction was kernel with II=4 may have similar throughput to kernel with II=2.

1. Modify host code to make sure it runs appropriate hardware kernel wiht II=4. Open terminal if it is not already open from last lab and do:

    ```bash
    source $AWS_FPGA_REPO_DIR/vitis_setup.sh
    export LAB_WORK_DIR=/home/centos/src/project_data/
    cd $LAB_WORK_DIR/Vitis-AWS-F1-Developer-Labs/modules/module_01/idct
    ```

    Now open `host.cpp` and make changes so that hardware run will use kernel namely "krnl_idct_med":

    ```bash
    vim src/host.cpp
    ```  

    Go to label "CREATE_KERNEL" near line 226 and ensure that the kernel name string is **"krnl_idct_med"** and not anything else and compile host application again:

    ```bash
    make compile_host TARGET=hw
    ``` 
1. Run application again:

    ```bash
    cd $LAB_WORK_DIR/Vitis-AWS-F1-Developer-Labs/modules/module_01/idct
    ./build/host.exe ./xclbin/krnl_idct.hw.awsxclbin $((1024*128)) 32 4
    ```
The throughput estimate and the acceleration factor reported will be similar to the maximum acceleration achieved in the last experiment with II=2.

## Estimated Performance versus Measured Performance

In an earlier experiment, the kernel latency and performance of overall system was discussed as follows:
    
    ```
    Estimted Kernel Latency =  = 1.33 ms 
    Estimated Overall Application Performance = 8.4 GB/s  
    ```
The measured performance is as follows:
    ```
    Measured Kernel Latency =  = 1.8 ms
    Measured Overall Application Performance = 6.4 GB/s  
    ```
The measured kernel latency can be found in run summary for application using Vitis Analyzer as explained in previous labs. From these numbers, it can be seen that in terms of throughput or latency estimates, almost 70% of what was estimated was achieved. The difference between estimates and the measured stems from the fact that once application software pipeline is created the device DDRs memories start to see contention between host and kernel where one is reading and other is writing. For example when host is reading kernel output moving data from device DDR to its global memory at the same time kernel is processing next block and writing new output data for next call, similar phenomenon happens for device DDR where kernel is reading and host is writing input data for next call** This contention can be resolved if more device DDR memories are available by using device DDRs in ping pong fashion. Another AWS training module namely [Bloom filter](https://github.com/Xilinx/SDAccel-AWS-F1-Developer-Labs/tree/master/modules/module_02) uses it.
 
## Summary  

In this lab, you learned the following:

- How to critically look at application timeline and profile summary
- Identifying potential for performance improvements from application timeline
- Identifying tasks that can overlap to create a software pipeline  
- Optimizing host code to enable software pipelining for performance improvements

---------------------------------------

## Stopping your instance

* Click the 'X' icon to close your RDP client.
* On your local machine, return to your browser and to the tab showing the **EC2 Console** and the details of your running instance.
   * If necessary, use the link which was emailed to you to return to the proper web page.
* In the **EC2 Console**, make sure you instance is selected
* Click the **Actions** button, select **Instance State** and then click **Stop** or **Terminate**.
   * Use **Stop** if you want to rapidly restart this instance later
   * Use **Terminate** if you want to permanantly delete this instance and its contents

## Congratulations!

You have successfully completed the first module of Vitis AWS F1 Developer Labs.

---------------------------------------

<p align="center"><b>
Return to the beginning: <a href="../../README.md">Vitis AWS F1 Developer Labs</a>
<p align="center"><sup>Copyright&copy; 2020 Xilinx</sup></p>
</b></p>

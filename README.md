Download Link: https://assignmentchef.com/product/solved-cmput379-assignment-2-posix-threads-and-synchronization-primitives
<br>
This programming assignment is intended to give you experience in using POSIX threads and synchronization primitives.

MapReduce Paradigm

MapReduce is a programming model and a distributed computing paradigm for large-scale data processing. It allows for applications to run various tasks in parallel, making them scalable and fault-tolerant. To use the MapReduce infrastructure, developers need to write just a little bit of code in addition to their program. They do not need to worry about how to parallelize their program; the MapReduce runtime will make this happen!

To use the MapReduce infrastructure, developers must implement two <em>callback </em>functions, namely Map and Reduce. The Map function takes a filename as input and generates intermediate key/value pairs from the data stored in that file. The Reduce function processes a subset of intermediate key/value pairs, <em>summarizing </em>all values associated with the same key. The summary operation typically reduces the list of values associated with a key to a single value. It could be as simple as summing up numerical values (e.g., in a distributed word count program) or concatenating string values (e.g., in a distributed grep program) associated with the same key.

The MapReduce infrastructure utilizes three types of computing resources as shown in Figure 1:

<ol>

 <li>workers executing the Map function (<em>mappers</em>);</li>

 <li>workers executing the Reduce function (<em>reducers</em>);</li>

 <li>a master (<em>controller</em>) assigning tasks to the two types of workers mentioned above.</li>

</ol>

These computing resources can be threads running on a multicore system or nodes in a distributed system. The numbers of mappers and reducers must be specified by the developer. Depending on the workload, adding more workers could improve the performance or not.

<h1>Your Task</h1>

In this programming assignment you are going to build a MapReduce library in C or C++ utilizing POSIX threads and synchronization primitives. This library will support the execution of user-defined Map and Reduce functions on a multicore system. The original MapReduce paper<a href="#_ftn1" name="_ftnref1"><sup>[1]</sup></a> shows the programming paradigm to work in a distributed computing environment where each worker in the map phase has its own set of intermediate key/value pairs stored locally. These intermediate pairs are then distributed across the workers (in the shuffle phase) before the reduce phase starts.

You will implement a different version of MapReduce in this assignment. We assume individual workers are threads running on the same system. Hence, your MapReduce library should create a fixed number of mapper threads (kept in reserve in a thread pool) and a fixed number of reducer threads to run the computation in parallel.

Figure 1: MapReduce execution overview

Implementing the thread pool using synchronization primitives is a central challenge of this assignment. You are not allowed to use an existing thread pool library.

Since the intermediate key/value pairs are stored in shared data structures, your MapReduce library must also use synchronization primitives to access this data. Otherwise, the data will be overwritten and lost. Designing and implementing this data structure is another challenge of the assignment.

<h1>MapReduce Execution</h1>

MapReduce execution can be divided into 7 steps that must be completed one after the other. These steps are explained below:

<ol>

 <li>Creating mappers: the master thread creates <em>M </em>mapper threads in the MRRun library function where <em>M </em>is an argument of this function. The mapper threads are in a thread pool implemented using synchronization primitives, such as mutex locks and condition variables. By maintaining a fixed number of threads, the thread pool helps to minimize the thread creation and destruction overhead. Note that the master thread will wait for the mapper threads to process all splits in the map phase before moving on to the reduce phase.</li>

 <li>Assigning data splits to mappers: in the MRRun function, the master thread iterates over <em>K </em>input data splits (represented by <em>K </em>files) and submits a job for each split to the thread pool. If there is an idle mapper thread in the thread pool, it starts processing the job right away by invoking the user-defined Map function with the passed argument (i.e., the filename). Otherwise, the job is added to a queue to be processed later by an idle mapper thread. The master ensures that each split is processed exactly once. Note that there are several different ways that jobs can be assigned to idle mapper threads; your implementation should use the <em>longest job first </em> This policy assigns the longest job (i.e., the largest input file) in the queue to the next idle mapper thread. To check the size of each input file, you can use the stat <a href="http://man7.org/linux/man-pages/man2/stat.2.html">system call</a><a href="http://man7.org/linux/man-pages/man2/stat.2.html">.</a></li>

 <li>Running the user-defined Map function: each mapper runs the user-defined Map function on a given split to generate key/value pairs. Every time a key/value pair is generated, the Map function invokes the MREmit library function, to write the intermediate key/value pair to a particular shared data structure. Since multiple mapper threads may attempt to concurrently update a shared data structure, this library function must use proper synchronization primitives, such as mutex locks.</li>

 <li>Partitioning the map output: when the MREmit function is called by a mapper thread, it first determines where to write the given key/value pair by calling the MRPartition function. The MRPartition function takes a key and maps it to an integer between 0 and <em>R </em>− 1, which indicates the data structure the key/value pair must be written to. This way MRPartition allows the MapReduce library to create <em>R </em>separate data structures (i.e., <em>partitions</em>), each containing a subset of keys and the value list associated with each of them. Once the partition that a key/value pair must be written to is identified, MREmit inserts the pair in a certain position in that partition, keeping the partition sorted in ascending key order at all times. This is necessary for the MRGetNext library function to easily distinguish when a new reduce task should start, which is exactly when the next key in the sorted partition is different from the previous key.</li>

 <li>Creating reducers: the master thread terminates the mapper threads when all <em>K </em>splits are processed in the map phase. It then creates <em>R </em>reducer threads in the MRRun library function where <em>R </em>is an argument of this function. Each reducer thread is responsible for processing data in a particular partition; hence, the <em>i</em>th partition (0 ≤ <em>i &lt; R</em>) is assigned to the <em>i</em>th reducer. Each reducer thread runs the MRProcessPartition library function which takes as input the index of the partition assigned to it.</li>

 <li>Running the user-defined Reduce function: the MRProcessPartition library function invokes the user-defined Reduce function on the next unprocessed key in the given partition in a loop. Thus, Reduce is invoked only once for each key. To perform the summary operation, the Reduce function calls the MRGetNext library function to iterate over all values that have the same key in its partition. The MRGetNext function returns the next value associated with the given key in the sorted partition or NULL when the key’s values have been processed completely.</li>

 <li>Producing the final output: each reducer thread writes the result of the summary operation performed on the value list associated with the passed key to a file named result-&lt;partitionnumber&gt;.txt. These output files are created initially by the reducer threads. The master thread terminates a reducer thread as soon as all keys in its corresponding partition are processed. The master thread frees up its resources and returns from the MREmit function once all keys in all <em>R </em>partitions are processed.</li>

</ol>

<h1>The MapReduce Library</h1>

We now describe the functions in the mapreduce.h header file (included in the starter code) to help you understand how to write your MapReduce library:

<em>// function pointer types used by library functions </em><strong>typedef </strong>void (*Mapper)(<strong>char </strong>*file_name); <strong>typedef </strong>void (*Reducer)(<strong>char </strong>*key, <strong>int </strong>partition_number);

<em>// library functions you must define </em><strong>void </strong>MR_Run(<strong>int </strong>num_files, <strong>char </strong>*filenames[],

Mapper map, <strong>int </strong>num_mappers, Reducer concate, <strong>int </strong>num_reducers);

<strong>void </strong>MR_Emit(<strong>char </strong>*key, <strong>char </strong>*value);

<strong>unsigned long </strong>MR_Partition(<strong>char </strong>*key, <strong>int </strong>num_partitions);

<strong>void </strong>MR_ProcessPartition(<strong>int </strong>partition_number);

<strong>char </strong>*MR_GetNext(<strong>char </strong>*key, <strong>int </strong>partition_number);

The MapReduce execution is started off by a call to MRRun in the user program. This function is passed an array of filenames containing <em>K </em>data splits, that is filenames[0]<em>,</em>··· <em>,</em>filenames[<em>K </em>− 1] (<em>K </em>= num files). Additionally, the MRRun function is passed the number of mappers <em>M</em>, the number of reducers <em>R</em>, and two function pointers for Map and Reduce functions. The main thread that runs the MRRun function is the master thread. It creates <em>M </em>mappers and <em>R </em>reducers as depicted in Figure 1.

The MREmit library function takes a key and a value associated with it, and writes this pair to a specific partition which is determined by passing the key to the MRPartition library function. This function can be any “good” hash function such as <a href="http://web.mit.edu/freebsd/head/sys/libkern/crc32.c">CRC32</a><a href="http://web.mit.edu/freebsd/head/sys/libkern/crc32.c">,</a> <a href="https://en.wikipedia.org/wiki/MurmurHash">MurmurHash</a><a href="https://en.wikipedia.org/wiki/MurmurHash">,</a> or the following algorithm known as DJB2:

<strong>unsigned long </strong>MR_Partition(<strong>char </strong>*key, <strong>int </strong>num_partitions) { <strong>unsigned long </strong>hash = 5381; <strong>int </strong>c; <strong>while </strong>((c = *key++) != ‘ ’) hash = hash * 33 + c;

<strong>return </strong>hash % num_partitions;

}

You can use the above hash function or the open-source implementation of a well-known hash function. Make sure to cite your sources in the readme file.

The MRProcessPartition library function takes the index of the partition assigned to the thread that runs it. It invokes the user-defined Reduce function in a loop, each time passing it the next unprocessed key. This continues until all keys in the partition are processed.

The MRGetNext library function takes a key and a partition number, and returns a value associated with the key that exists in that partition. In particular, the <em>i</em>th call to this function should return the <em>i</em>th value associated with the key in the sorted partition or NULL if <em>i </em>is greater than the number of values associated with the key.

In the following section, we provide an example user program which is useful for testing your MapReduce library. Your task is to implement the library assuming that there exists a parallelizable user program that includes your library and calls the MRRun function.

<h2>Example Map and Reduce Functions</h2>

To illustrate how the library functions are invoked from the user program, we describe a simple distributed word count program, distwc.c, which uses the MapReduce library. A word count program counts the number of times each word appears in a given set of files. We will assume that the filenames are valid and the files exist in the file system. This main function takes a list of text files and passes it to MRRun along with the numbers of mappers and reducers, and two function pointers. All the developer has to do is to include the MapReduce library and implement Map and Reduce functions. The distributed word count program is described below.

<em>#include </em><em>&lt;assert.h&gt;</em>

<em>#include </em><em>&lt;stdio.h&gt;</em>

<em>#include </em><em>&lt;stdlib.h&gt;</em>

<em>#include </em><em>&lt;string.h&gt;</em>

<em>#include </em><em>“mapreduce.h”</em>

Figure 2: Example MapReduce data flow

<strong>void </strong>Map(<strong>char </strong>*file_name) { <strong>FILE </strong>*fp = fopen(file_name, “r”); assert(fp != NULL);

<strong>char </strong>*line = NULL; <strong>size_t </strong>size = 0; <strong>while </strong>(getline(&amp;line, &amp;size, fp) != -1) { <strong>char </strong>*token, *dummy = line;

<strong>while </strong>((token = strsep(&amp;dummy, ” <strong>t
r</strong>“)) != NULL) { MR_Emit(token, “1”);

} } free(line); fclose(fp);

}

<strong>void </strong>Reduce(<strong>char </strong>*key, <strong>int </strong>partition_number) { <strong>int </strong>count = 0; <strong>char </strong>*value, name[100]; <strong>while </strong>((value = MR_GetNext(key, partition_number)) != NULL) count++;

sprintf(name, “result-%d.txt”, partition_number); <strong>FILE </strong>*fp = fopen(name, “a”); fprintf(fp, “%s: %d<strong>
</strong>“, key, count); fclose(fp);

}

<strong>int </strong>main(<strong>int </strong>argc, <strong>char </strong>*argv[]) {

MR_Run(argc – 1, &amp;(argv[1]), Map, 10, Reduce, 10);

}

<h2>Using Pthreads</h2>

To implement the MapReduce library, you must use the POSIX threads library (pthreads) for thread management.

See the <a href="http://man7.org/linux/man-pages/man7/pthreads.7.html">man</a> page of pthreads for more information. We are going to assume that the intermediate key/value pairs are fixed during the reduce phase. Thus, you must wait until all mapper threads finish execution, before starting the reduce phase.

<h2>Intermediate Data Structures</h2>

The data structures being used for the intermediate key/value pairs are shared among multiple threads. Hence, they must be global variables in the MapReduce library and support concurrency for correctness. Each thread must obtain the lock on a shared data structure before updating it. For locking and unlocking the data structure you will use pthreadmutex.

The implementation of the shared data structures is up to you. You might use data structures from <a href="https://en.cppreference.com/w/cpp/container">C++ STL </a>or implement the data structures you need (if you write code in C, for example). While there are many ways to implement this data structure, it must be thread-safe, and support efficient implementation of MREmit and MRGetNext functions.

<h1>Thread Pool</h1>

The MapReduce library adopts a thread pool library to create a fixed number of mapper threads and assign the task of running the Map callback function on input files to them.

In this assignment, you will write your own thread pool library using POSIX mutex locks and condition variables. Below we describe the functions and struct declarations in the threadpool.h header file to help you understand what this library is supposed to do.

<strong>typedef </strong>void (*thread_func_t)(<strong>void </strong>*arg);

<strong>typedef struct </strong>ThreadPool_work_t {

thread_func_t func;           <em>// The function pointer </em><strong>void </strong>*arg;         <em>// The arguments for the function</em>

<em>// TODO: add other members here if needed</em>

} ThreadPool_work_t;

<strong>typedef struct </strong>{

<em>// TODO: add members here </em>} ThreadPool_work_queue_t;

<strong>typedef struct </strong>{

<em>// TODO: add members here</em>

} ThreadPool_t;

<em>/**</em>

<ul>

 <li><em>A C style constructor for creating a new ThreadPool object * Parameters:</em></li>

 <li><em>num – The number of threads to create * Return:</em></li>

 <li><em>ThreadPool_t* – The pointer to the newly created ThreadPool object</em></li>

</ul>

<em>*/</em>

ThreadPool_t *ThreadPool_create(<strong>int </strong>num);

<em>/**</em>

<ul>

 <li><em>A C style destructor to destroy a ThreadPool object</em></li>

 <li><em>Parameters:</em></li>

 <li><em>tp – The pointer to the ThreadPool object to be destroyed</em></li>

</ul>

<em>*/ </em><strong>void </strong>ThreadPool_destroy(ThreadPool_t *tp);

<em>/**</em>

<ul>

 <li><em>Add a task to the ThreadPool’s task queue * Parameters:</em></li>

 <li><em>tp – The ThreadPool object to add the task to</em></li>

 <li><em>func – The function pointer that will be called in the thread</em></li>

 <li><em>arg – The arguments for the function * Return:</em></li>

 <li><em>true – If successful</em></li>

 <li><em>false – Otherwise</em></li>

</ul>

<em>*/ </em><strong>bool </strong>ThreadPool_add_work(ThreadPool_t *tp, thread_func_t func, <strong>void </strong>*arg);

<em>/**</em>

<ul>

 <li><em>Get a task from the given ThreadPool object * Parameters:</em></li>

 <li><em>tp – The ThreadPool object being passed * Return:</em></li>

 <li><em>ThreadPool_work_t* – The next task to run</em></li>

</ul>

<em>*/</em>

ThreadPool_work_t *ThreadPool_get_work(ThreadPool_t *tp);

<em>/**</em>

<ul>

 <li><em>Run the next task from the task queue * Parameters:</em></li>

 <li><em>tp – The ThreadPool Object this thread belongs to</em></li>

</ul>

<em>*/</em>

<strong>void </strong>*Thread_run(ThreadPool_t *tp);

The ThreadPoolcreate and ThreadPooldestroy functions create and destroy the ThreadPool object, respectively. Each thread created by ThreadPoolcreate runs the Threadrun function which gets a task from the task queue and executes it (this is done in a loop). The ThreadPooldestroy function should wait until all tasks are executed before destroying the ThreadPool.

The ThreadPooladdwork function is used to submit a task (i.e., the execution of a function) to the thread pool. You must create your own task queue and have a mechanism to support concurrency. The ThreadPoolgetwork function is used to get a task from the queue and have it processed by an idle thread.

You will have to define ThreadPoolworkt, ThreadPoolworkqueuet, and ThreadPoolt structs in threadpool.h.

<a href="#_ftnref1" name="_ftn1">[1]</a> Dean, J., &amp; Ghemawat, S. (2008). MapReduce: Simplified Data Processing on Large Clusters. <em>Communications of the ACM</em>, 51(1), 107-113.
---
title: MapReduce原理与实现（二）：MapReduce的实现
date: 2017-01-12 14:05:19
categories:
- Mit 6.824
tags:
- MapReduce
- 分布式
- Mit 6.824
---

## 概览

在实现MapReduce的过程中，我将创建一个MapReduce库。这个库是干嘛的呢？具体来说，就是使用者只需要定义自己的Map和Reduce函数，即可完成MapReduce功能。

如下面所示：

	input is divided into "splits"
	Input Map -> a,1 b,1 c,1
	Input Map ->     b,1
	Input Map -> a,1     c,1
	            |   |   |
	                |   -> Reduce -> c,2
	                -----> Reduce -> b,2

1. MR会对每个split来调用Map()，并产生中间结果集k2,v2

2. 对于每个k2，MR会聚合所有的中间值v2。然后调用Reduce()

3. 从Reduce()产生最终的结果集<k2,v3>

下面看一个具体的例子：word count

input is thousands of text files
Map(k, v)
	split v into words
	for each word w
		emit(w, "1")
Reduce(k, v)
	emit(len(v))

如上图：

- 输入：上千个文本文件

- Map(k, v)：将输入v切割成具体的word，对于每个word，发出(w,"1")

- Reduce(k, v)：计算v的长度，然后返回给调用者。

## MR框架实现

上面给出了word count的功能，以及用户定义的Map和Reduce方法的作用，从下面开始，我们开始用代码来实现具体的功能。

### 单线程MapReduce

为了方便调试，我们先顺序执行MapReduce。

在实现Map和Reduce的功能之前，我们先完成MapReduce库，即MR最核心的功能。

为了方便，我们先写出最简单的Map和Reduce方法，如下：

``` go 
func MapFunc(file string, value string) (res []KeyValue) {
	debug("Map %v\n", value)
	words := strings.Fields(value)
	for _, w := range words {
		kv := KeyValue{w, ""}
		res = append(res, kv)
	}
	return
}

// Just return key
func ReduceFunc(key string, values []string) string {
	for _, e := range values {
		debug("Reduce %s %v\n", key, e)
	}
	return ""
}
```
顺序执行中，会先执行Map阶段，然后再执行Reduce阶段。

根据原始论文的描述，Map阶段分为下面几个步骤：

1. 分配到map任务的worker会读取对应的输入文件的内容。解析出输入数据的k/v对，然后传递给*Map*方法。map方法生成的K/V中间结果会在内存中缓存。

代码如下：

``` go
// doMap does the job of a map worker: it reads one of the input files
// (inFile), calls the user-defined map function (mapF) for that file's
// contents, and partitions the output into nReduce intermediate files.
func doMap(
	jobName string, // the name of the MapReduce job
	mapTaskNumber int, // which map task this is
	inFile string,
	nReduce int, // the number of reduce task that will be run ("R" in the paper)
	mapF func(file string, contents string) []KeyValue,
) {
	// TODO:
	// You will need to write this function.
	// You can find the filename for this map task's input to reduce task number
	// r using reduceName(jobName, mapTaskNumber, r). The ihash function (given
	// below doMap) should be used to decide which file a given key belongs into.
	//
	// The intermediate output of a map task is stored in the file
	// system as multiple files whose name indicates which map task produced
	// them, as well as which reduce task they are for. Coming up with a
	// scheme for how to store the key/value pairs on disk can be tricky,
	// especially when taking into account that both keys and values could
	// contain newlines, quotes, and any other character you can think of.
	//
	// One format often used for serializing data to a byte stream that the
	// other end can correctly reconstruct is JSON. You are not required to
	// use JSON, but as the output of the reduce tasks *must* be JSON,
	// familiarizing yourself with it here may prove useful. You can write
	// out a data structure as a JSON string to a file using the commented
	// code below. The corresponding decoding functions can be found in
	// common_reduce.go.
	//
	//   enc := json.NewEncoder(file)
	//   for _, kv := ... {
	//     err := enc.Encode(&kv)
	//
	// Remember to close the file after you have written all the values!
	//	dat, err := ioutil.ReadFile(inFile)
	//	if err != nil {
	//		log.Fatal("read input file: ", err)
	//	}
	file, err := os.Open(inFile)
	if err != nil {
		log.Fatal("read input file: ", err)
	}
	inf, err := file.Stat()
	contents := make([]byte, inf.Size())
	file.Read(contents)
	file.Close()
	kv := mapF(inFile, string(contents)) //让具体的map去执行
	fileArray := make([]*os.File, nReduce)
	for i := 0; i < nReduce; i++ {
		reduceName := reduceName(jobName, mapTaskNumber, i)
		file, err := os.Create(reduceName) //创建file
		if err != nil {
			log.Fatal("create reduce file: ", err)
		}
		fileArray[i] = file
	}
	for _, v := range kv {
		key := v.Key
		enc := json.NewEncoder(fileArray[ihash(key)%uint32(nReduce)])
		err := enc.Encode(&v)
		if err != nil {
			log.Fatal("write reduce file: ", err)
		}
	}
	for _, v := range fileArray {
		v.Close()
	}
}
```

Map阶段执行完之后，会进入到Reduce阶段。再Reduce阶段，MR框架会做哪些事情呢：

1. 当master通知一个reduce worker这些数据保存的位置时，该worker使用RPC调用来读取本地磁盘保存的数据。当reduce worker读取完所有的中间结果时，它会根据key来排序。

2.  reduce worker迭代所有中间结果，对于每个独一无二的key，它将与这个key对应的集合数据传递给用户定义的*Reduce*方法。reduce输出的结果会被追加到结果文件中。

代码如下：

``` go
// doReduce does the job of a reduce worker: it reads the intermediate
// key/value pairs (produced by the map phase) for this task, sorts the
// intermediate key/value pairs by key, calls the user-defined reduce function
// (reduceF) for each key, and writes the output to disk.
func doReduce(
	jobName string, // the name of the whole MapReduce job
	reduceTaskNumber int, // which reduce task this is
	nMap int, // the number of map tasks that were run ("M" in the paper)
	reduceF func(key string, values []string) string,
) {
	// TODO:
	// You will need to write this function.
	// You can find the intermediate file for this reduce task from map task number
	// m using reduceName(jobName, m, reduceTaskNumber).
	// Remember that you've encoded the values in the intermediate files, so you
	// will need to decode them. If you chose to use JSON, you can read out
	// multiple decoded values by creating a decoder, and then repeatedly calling
	// .Decode() on it until Decode() returns an error.
	//
	// You should write the reduced output in as JSON encoded KeyValue
	// objects to a file named mergeName(jobName, reduceTaskNumber). We require
	// you to use JSON here because that is what the merger than combines the
	// output from all the reduce tasks expects. There is nothing "special" about
	// JSON -- it is just the marshalling format we chose to use. It will look
	// something like this:
	//
	// enc := json.NewEncoder(mergeFile)
	// for key in ... {
	// 	enc.Encode(KeyValue{key, reduceF(...)})
	// }
	// file.Close()

	keyValueGroup := make(map[string][]string) //it sorts it by the intermediate keys so that all occurrences of the same key are grouped together.
	for i := 0; i < nMap; i++ {
		reduceName := reduceName(jobName, i, reduceTaskNumber)
		file, err := os.Open(reduceName)
		if err != nil {
			fmt.Printf("reduce file can't open\n")
		} else {
			enc := json.NewDecoder(file)
			for {
				var kv KeyValue
				err := enc.Decode(&kv)
				if err != nil {
					break
				}
				_, ok := keyValueGroup[kv.Key]
				if !ok {
					keyValueGroup[kv.Key] = make([]string, 0)
				}
				tempArray := keyValueGroup[kv.Key]
				groupArray := append(tempArray, kv.Value)
				keyValueGroup[kv.Key] = groupArray
			}
			file.Close()
		}
	}
	file, err := os.Create(mergeName(jobName, reduceTaskNumber))
	if err != nil {
		log.Fatal("create merge file: ", err)
	}
	enc := json.NewEncoder(file)
	for k := range keyValueGroup {
		enc.Encode(KeyValue{k, reduceF(k, keyValueGroup[k])})
	}
	file.Close()
}
```

可以看到。


### 多线程MapReduce

在多线程的情况下，会有多个worker同时工作。也会有一个master来分配和管理。

``` go
// schedule starts and waits for all tasks in the given phase (Map or Reduce).
func (mr *Master) schedule(phase jobPhase) {
	var ntasks int
	var nios int // number of inputs (for reduce) or outputs (for map)
	switch phase {
	case mapPhase:
		ntasks = len(mr.files)
		nios = mr.nReduce
	case reducePhase:
		ntasks = mr.nReduce
		nios = len(mr.files)
	}

	fmt.Printf("Schedule: %v %v tasks (%d I/Os)\n", ntasks, phase, nios)

	// All ntasks tasks have to be scheduled on workers, and only once all of
	// them have been completed successfully should the function return.
	// Remember that workers may fail, and that any given worker may finish
	// multiple tasks.
	//
	// TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO
	//
	fmt.Println("Start start start")
	var wg sync.WaitGroup //
	for i := 0; i < ntasks; i++ {
		wg.Add(1) // 增加WaitGroup的计数
		go func(taskNum int, nios int, phase jobPhase) {
			defer wg.Done()                // 当整个goroutine完成后, 减少引用计数
			worker := <-mr.registerChannel // 获取工作rpc服务器, worker == address
			fmt.Println("DEBUG: current worker port: %v\n", worker)

			var args DoTaskArgs
			args.JobName = mr.jobName
			args.File = mr.files[taskNum]
			args.Phase = phase
			args.TaskNumber = taskNum
			args.NumOtherPhase = nios
			ok := call(worker, "Worker.DoTask", &args, new(struct{}))
			if ok {
				go func() {
					mr.registerChannel <- worker // 该rpc服务器完成任务, 则重新放入registerChannel
				}()
			}
		}(i, nios, phase)
	}
	wg.Wait() // 等待所有的任务完成

	fmt.Printf("Schedule: %v phase done\n", phase)
}
```


### 故障转移













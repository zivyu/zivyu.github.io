---
title: MapReduce原理与实现（二）：MapReduce的实现
date: 2017-01-12 14:05:19
categories:
- 分布式学习
tags:
- MapReduce
- 分布式
- Mit 6.824
---

## 概览

在实现MapReduce的过程中，主要是实现一个MapReduce框架。这个框架是干嘛的呢？具体来说，就是使用者只需要定义自己的Map和Reduce方法，即可完成MapReduce功能。如下所示：

如下所示：
	
	input is divided into "splits"
	Input Map -> a,1 b,1 c,1
	Input Map ->     b,1
	Input Map -> a,1     c,1
	            |   |   |
	                |   -> Reduce -> c,2
	                -----> Reduce -> b,2
<!-- more -->

主要逻辑如下：

1. MR会对每个split来调用Map()，并产生中间结果集k2,v2
2. 对于每个k2，MR会聚合所有与之相对应的中间值v2。然后调用Reduce()
3. 从Reduce()会产生最终的结果集<k2,v3>

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

根据论文描述，MapReduce主要分为两个阶段：
Map阶段：
1. 分配到map任务的worker会读取对应的输入文件的内容。
2. 将文件内容传递给*Map*方法。
3. map方法生成的K/V中间结果会被保存在本地文件中。

Reduce阶段：
1. 当master通知一个reduce worker这些数据保存的位置时，该worker使用RPC调用来读取本地磁盘保存的数据。
2. 将中间结果按照key进行归类。
3. 将归类后的k/v集合传递给*Reduce*方法。
4. 得到最终结果。

为了实现word count功能。可以由浅入深，分为四个步骤进行：
1. 单线程情况下，完成MR最核心的功能，即MR的doMap()和doReduce()方法。
2. 单线程下，完成用户自定义的Map()和Reduce()方法，实现word count功能。
3. 多线程下，会新增master，完成master的schedule()方法，实现word count功能。
4. 实现故障容错。

### 单线程MapReduce

在单线程情况下，MR首先会执行doMap，然后执行doReduce。

doMap()：被map worker调用，读取一个输入文件，得到文件内容，然后调用用户自定义的map方法，并把map方法返回的结果保存在本地文件中。

doReduce()：被reduce worker调用，读取在map阶段产生的中间结果，并对key进行归类，然后调用用户自定义的reduce方法，得到最终结果。

所以先实现MR框架的doMap()方法，如下：

代码如下：

``` go
func doMap(
	jobName string, // the name of the MapReduce job
	mapTaskNumber int, // which map task this is
	inFile string,
	nReduce int, // the number of reduce task that will be run ("R" in the paper)
	mapF func(file string, contents string) []KeyValue,
) {
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

然后再实现doReduce方法，如下：

代码如下：

``` go
func doReduce(
	jobName string, // the name of the whole MapReduce job
	reduceTaskNumber int, // which reduce task this is
	nMap int, // the number of map tasks that were run ("M" in the paper)
	reduceF func(key string, values []string) string,
) {
	keyValueGroup := make(map[string][]string) 
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

### 单线程的word count

经过上面的步骤，我们已经完成的MR最核心的两个方法，现在再来自定义具体的map和reduce方法，实现word count功能。

map()：完成文本切分，并返回KeyValue的集合。

reduce()：对每个key的value进行计算。得到最终结果。

map()和reduce()方法如下：

``` go
func mapF(document string, value string) (res []mapreduce.KeyValue) {
	// TODO: you have to write this function
	//	words := strings.Fields(value)
	//	for _, v := range words {
	//		res = append(res, mapreduce.KeyValue{v, "1"})
	//	}
	//	return
	res = make([]mapreduce.KeyValue, 0)
	ss := strings.FieldsFunc(value, func(c rune) bool {
		return !unicode.IsLetter(c)
	})
	for _, s := range ss {
		res = append(res, mapreduce.KeyValue{s, "1"})
	}
	return
}

func reduceF(key string, values []string) string {
	// TODO: you also have to write this function
	count := 0
	for _, v := range values {
		i64, err := strconv.ParseInt(v, 10, 32)
		if err != nil {

		}
		count = count + int(i64)
	}
	return strconv.Itoa(count)
}
```

### 多线程MapReduce

经过上面的步骤，已经完成的单线程版的word count功能。

但是在多线程的情况下，会有多个worker同时工作，这时候会有一个master来分配和管理。

在不考虑故障容错的情况下，根据论文描述，master主要的职责如下：

1. worker启动时候会注册到master上。
2. master为空闲的worker分配doMap或者doReduce任务。

master的调度主要体现在schedule()方法中：

实现如下：

``` go
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
		wg.Add(1)
		go func(taskNum int, nios int, phase jobPhase) {
			defer wg.Done()                
			worker := <-mr.registerChannel 
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
					mr.registerChannel <- worker
				}()
			}
		}(i, nios, phase)
	}
	wg.Wait()

	fmt.Printf("Schedule: %v phase done\n", phase)
}
```

### 故障转移

在上面多线程实现中，我们没有考虑worker失败的情况，那么当worker失败该如何处理呢？

在worker执行的过程中，每个worker都是无状态的节点。所以当worker失败的时候，master可以简单的将这个处理失败的任务分配给其他的worker进行处理。

改进schedule()方法，RPC调用每个worker进行处理的时候，如果rpc失败，则将任务分配给其他的worker进行处理，一直到rpc返回ok。

最简单粗暴的方法来实现：

``` go
for {
	worker := <-mr.registerChannel  // 获取工作rpc服务器, worker == address
	var args DoTaskArgs
	args.JobName = mr.jobName
	args.File = mr.files[taskNum]
	args.Phase = phase
	args.TaskNumber = taskNum
	args.NumOtherPhase = nios
	ok := call(worker, "Worker.DoTask", &args, new(struct{}))
	if ok {
		go func() {
			//doneChannel <- taskNum
			mr.registerChannel <- worker 
		}()
		break  //执行成功，才结束这个任务。
	} 
}
```














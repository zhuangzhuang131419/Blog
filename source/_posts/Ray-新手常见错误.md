---
title: Ray 新手常见错误
date: 2020-12-08 18:06:02
tags:
---

# Delay ray.get()
调用 ```get``` 会产生一个副作用, 会阻塞 ```driver``` 程序去做别的操作，这样就会影响并行性.

首先看一个普通的例子:
```Python
import time

def do_some_work(x):
    time.sleep(1) # Replace this with work you need to do.
    return x

start = time.time()
results = [do_some_work(x) for x in range(4)]
print("duration =", time.time() - start)
print("results = ", results)


# Output
# duration = 4.0149290561676025
# results =  [0, 1, 2, 3]
```

现在当我们给这个函数加上 ```remote```

```Python
import time
import ray

ray.init(num_cpus = 4) # Specify this system has 4 CPUs.

@ray.remote
def do_some_work(x):
    time.sleep(1) # Replace this is with work you need to do.
    return x

start = time.time()
results = [do_some_work.remote(x) for x in range(4)]
print("duration =", time.time() - start)
print("results = ", results)


# Output
# duration = 0.0003619194030761719
# results =  [ObjectRef(df5a1a828c9685d3ffffffff0100000001000000), ObjectRef(cb230a572350ff44ffffffff0100000001000000), ObjectRef(7bbd90284b71e599ffffffff0100000001000000), ObjectRef(bd37d2621480fc7dffffffff0100000001000000)]
```
但是这个 result 不是我们想要的, 我们调用 ```ray.get()``` 来获取结果

```Python
results = [ray.get(do_some_work.remote(x)) for x in range(4)]
```

但是改成这样之后呢，输出就变成:

```Python
duration = 4.018050909042358
results =  [0, 1, 2, 3]
```

现在结果是对的, 但程序因此也退化成了串行. 原因也是显而易见的, 每次调用 ```ray.get``` 都会阻塞住程序.

修改的办法就是在调用完所有的 ```task``` 后再调用 ```ray.get``` 

```Python
results = ray.get([do_some_work.remote(x) for x in range(4)])

# Output: 
# duration = 1.0064549446105957
# results =  [0, 1, 2, 3]
```

时刻保持警惕 ```ray.get()``` 是一个会阻塞的操作, 尽可能晚的去调用这个方法 ```ray.get()```

# Avoid tiny tasks
避免出现执行时间过短的 ```task```, 因为并行的结果可能适得其反.

```Python
import time

def tiny_work(x):
    time.sleep(0.0001) # Replace this with work you need to do.
    return x

start = time.time()
results = [tiny_work(x) for x in range(100000)]
print("duration =", time.time() - start)

# Output:
# duration = 13.36544418334961
```

这是一个符合逻辑的结果，当我们换成 ```remote``` 时

```Python
import time
import ray

ray.init(num_cpus = 4)

@ray.remote
def tiny_work(x):
    time.sleep(0.0001) # Replace this is with work you need to do.
    return x

start = time.time()
result_ids = [tiny_work.remote(x) for x in range(100000)]
results = ray.get(result_ids)
print("duration =", time.time() - start)

# Output:
# duration = 27.46447515487671
```

结果非常的出乎意料, 因为我们并行的运行总时间反而增加了. 这是因为调用每一个 ```task``` 都会有一个小的开销(调度, 进程间通讯, 更新系统的状态), 由于 ```task``` 本身所需的时间太小, 那么这些额外的开销反而占据了大头.

修改的办法就是平摊这些开销

```Python
import time
import ray

ray.init(num_cpus = 4)

def tiny_work(x):
    time.sleep(0.0001) # replace this is with work you need to do
    return x

@ray.remote
def mega_work(start, end):
    return [tiny_work(x) for x in range(start, end)]

start = time.time()
result_ids = []
[result_ids.append(mega_work.remote(x*1000, (x+1)*1000)) for x in range(100)]
results = ray.get(result_ids)
print("duration =", time.time() - start)


# Output:
# duration = 3.2539820671081543
```

经过测试, 运行一个空的 ```task``` 大概需要 0.5s, 这就说明我们的 ```task``` 需要一定的运行时间来摊平这个额外的开销.

# Avoid passing same object repeatedly to remote tasks

当我们给一个 remote function 传递一个大的对象时, Ray 调用 ```ray.put()``` 来把对象存在本地 object store 中. 这种操作可以显著的提升性能, 但是也会导致一些问题. 比如: 重复地传递相同的对象


```Python
import time
import numpy as np
import ray

ray.init(num_cpus = 4)

@ray.remote
def no_work(a):
    return

start = time.time()
a = np.zeros((5000, 5000))
result_ids = [no_work.remote(a) for x in range(10)]
results = ray.get(result_ids)
print("duration =", time.time() - start)

# Output: 
# duration = 1.0837509632110596
```

这个时间对于一个仅仅进行了10个 ```tasks``` 显然偏大了. 造成这个情况的原因是每次我们调用 ```no_work(a)```, Ray 就会会调用 ```ray.put(a)``` 就会导致复制一个非常大的数组导致时间的开销. 

解决这个问题我们可以 显式地调用 ```ray.put(a)``` 然后把 obejct 的 ID 传递过去就可以避免复制

```Python
import time
import numpy as np
import ray

ray.init(num_cpus = 4)

@ray.remote
def no_work(a):
    return

start = time.time()
a_id = ray.put(np.zeros((5000, 5000)))
result_ids = [no_work.remote(a_id) for x in range(10)]
results = ray.get(result_ids)
print("duration =", time.time() - start)

# Output: 
# duration = 0.132796049118042
```


# Pipeline data processing

如果我们对一系列 ```tasks``` 使用 ```ray.get()```. 我们就需要等到最后一个 ```task``` 完成, 当每一个 ```task``` 完成的时间相差很大的时候, 就会导致一些问题.

```Python
import time
import random
import ray

ray.init(num_cpus = 4)

@ray.remote
def do_some_work(x):
    time.sleep(random.uniform(0, 4)) # Replace this with work you need to do.
    return x

def process_results(results):
    sum = 0
    for x in results:
        time.sleep(1) # Replace this with some processing code.
        sum += x
    return sum

start = time.time()
data_list = ray.get([do_some_work.remote(x) for x in range(4)])
sum = process_results(data_list)
print("duration =", time.time() - start, "\nresult = ", sum)

# Output:
# duration = 7.82636022567749
# result =  6

```

总的时间由两部分组成(do_some_work + process_results), 由于需要等所有的 ```task``` 都完成(花费了将近4s)

解决这个问题我们可以对一系列 object ID调用 ```ray.wait()```. 返回的参数(1) 就绪的 object ID. (2) 还未就绪的 object ID.

```Python
import time
import random
import ray

ray.init(num_cpus = 4)

@ray.remote
def do_some_work(x):
    time.sleep(random.uniform(0, 4)) # Replace this is with work you need to do.
    return x

def process_incremental(sum, result):
    time.sleep(1) # Replace this with some processing code.
    return sum + result

start = time.time()
result_ids = [do_some_work.remote(x) for x in range(4)]
sum = 0
while len(result_ids):
    done_id, result_ids = ray.wait(result_ids)
    sum = process_incremental(sum, ray.get(done_id[0]))
print("duration =", time.time() - start, "\nresult = ", sum)

# Output:
# duration = 4.852453231811523
# result =  6
```

{% asset_img PipelineDataProcessing.png PipelineDataProcessing %}
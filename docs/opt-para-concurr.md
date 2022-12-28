# Concurrency & Parallelism

Parallel programming means executing operations while using multiple CPU's processes (cores) or using multiple threads in a process. This speeds up the operation multitudes faster based on the number of processes or threads launched.


## Python's GIL problem

Python has a **Global Interpreter Lock** (GIL), which prevent two threads from executing simultaneously in the same program. However, libraries like numpy bypass this limitation by running external code in C.

Because of this GIL limitation, threads provide no benefit for **CPU intensive tasks**, and is only used for **Input/Ouput (IO) tasks**. CPU-limiting processes refer to one where most part of the life is spent in cpu, while IO-limiting processes refer to one where most part of the life is spent in i/o state, i.e. instructing another program to run something.

Why does IO task work when it can also only use a single thread in a process at a time? Let's take an example, one thread fires off a request to a URL and while it is waiting for a response, that thread can be swapped out for another thread that fires another request to another URL. Since a thread doesn’t have to do anything until it receives a response, it doesn’t really matter that only one thread is executing at a given time. This makes multi-threading in Python as a **concurrent** or **asychronous** process.

Given all these, we know when to use multi-processes and threads to run tasks with improved latency.

| Multiprocess | Multithread |
|-|-|
| more overhead than threads as opening and closing processes takes more time | fast as they share memory space and efficiently read and write to the same variables |
| only for CPU-intensive tasks because of overheads |only for IO-intensive tasks because fo GIL |
| E.g. complex calculations | E.g. networking, file read-write, database query |
| True Parallel Process | Concurrent Process |

Great detailed explanations are provided by [Brendan Fortuner](https://medium.com/@bfortuner/python-multithreading-vs-multiprocessing-73072ce5600b), [Thilina Rajapakse](https://medium.com/towards-artificial-intelligence/the-why-when-and-how-of-using-python-multi-threading-and-multi-processing-afd1b8a8ecca), [stackoverflow](https://stackoverflow.com/questions/4844637/what-is-the-difference-between-concurrency-parallelism-and-asynchronous-methods), and testdrive.io [1](https://testdriven.io/blog/concurrency-parallelism-asyncio/) & [2](https://testdriven.io/blog/python-concurrency-parallelism/).


## Examples

Below are two examples of embarassingly parallel tasks, one a CPU-bound job...

```python
def compute_x(x, y=2):
    from time import sleep
    sleep(0.8)        
    return (x,y)

list_to_iterate = [1,2,3,4,5,6,7,8,9,10]
sum_all_x = [compute_x(i) for i in list_to_iterate]
```

and an IO-bound job where it is requesting data from a website.

```python
import requests

wiki_page_urls = [
    "https://en.wikipedia.org/wiki/Ocean",
    "https://en.wikipedia.org/wiki/Island",
    "https://en.wikipedia.org/wiki/this_page_does_not_exist",
    "https://en.wikipedia.org/wiki/Shark",
]

def get_wiki_page_existence(wiki_page_url, timeout=10):
    response = requests.get(url=wiki_page_url, timeout=timeout)
    page_status = "unknown"
    if response.status_code == 200:
        page_status = "exists"
    elif response.status_code == 404:
        page_status = "does not exist"
    return wiki_page_url + " - " + page_status
```

## Joblib

[Joblib](https://joblib.readthedocs.io/en/latest/parallel.html) makes embarrassingly parallel for loops written in a list comprehension very simple, with the following syntax.

```python
from joblib import Parallel, delayed

variable_name = Parallel(n_jobs=<no_of_jobs>)(
    delayed(<func_name>)(<arg1>, <arg2>, <arg3>) for i in <list_to_iterate>)
```

### Parallel

We can run this in parallel with multiprocessing in joblib as this. By default it uses the backend `loky`, which is more robust than `multiprocessing`, taken from the older `multiprocessing.Pool` library. However, it might not always be faster.

```python
from joblib import Parallel, delayed

sum_all_x = Parallel(n_jobs=4)(
    delayed(compute_x)(i) for i in list_to_iterate)
```

### Concurrent

However, if this is an I/O intensive task, using `threading` to run it asynchronously is preferred. 

```python
from joblib import Parallel, delayed

sum_all_x = Parallel(n_jobs=4, backend="threading")(
    delayed(compute_x)(i) for i in list_to_iterate)
```

## Concurrent Futures

`concurrent.futures` is an in-built library that can easily run multi-processing or threading tasks.


### Parallel

```python
from concurrent.futures import ProcessPoolExecutor, as_completed

def multiproc(list_to_iterate, workers=4):
    with ProcessPoolExecutor(max_workers=workers) as executor:
        futures = []
        for i in list_to_iterate:
            executor.submit(compute_x, x=i)
        result = [future.result() for future in as_completed(futures)]
    return result


# alternatively, we can use the map function for a more concise result
# however, it does not allow multiple args to the worker function
def multiproc(list_to_iterate, workers=4):
    with ProcessPoolExecutor(max_workers=workers) as executor:
        futures = executor.map(compute_x, list_to_iterate)
        result = [future for future in futures]
    return result


# to do that, we need to use another library to wrap the worker function
# and additional args together
from functools import partial

def multiproc(list_to_iterate, workers=4):
    with ProcessPoolExecutor(max_workers=workers) as executor:
        wrap = partial(compute_x, y=3)
        futures = executor.map(wrap, list_to_iterate)
        result = [future for future in futures]
    return result


if __name__ == "__main__":
    result = multiproc(list_to_iterate)
```

### Concurrent

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

with ThreadPoolExecutor(max_workers=2) as executor:
    futures = []
    for url in wiki_page_urls:
        futures.append(executor.submit(get_wiki_page_existence, wiki_page_url=url))
    result = [future.result() for future in as_completed(futures)]
```

## Multiprocessing

The `multiprocessing` module is another in-built library that supports both multi-processing `multiprocessing.Pool` or threading `multiprocessing.pool.ThreadPool` tasks.

### Parallel

```python
import multiprocessing as mp

def multiproc(list_to_iterate, workers=2):
    pool = mp.Pool(workers)
    result = pool.map(compute_x, list_to_iterate)
    return result

if __name__ == "__main__":
    result = multiproc(list_to_iterate)
```

### Concurrent

```python
import multiprocessing as mp

def multithread(list_to_iterate, workers=2):
    pool = mp.pool.ThreadPool(workers)
    result = pool.map(get_wiki_page_existence, wiki_page_urls)
    return result

if __name__ == "__main__":
    result = multithread(list_to_iterate)
```

## Threading Library

The python default threading library provides a simple way to spin off a thread for an I/O bound task.

```python
import threading
threading.Thread(target=<function>).start()
```

## Numba

[Numba](http://numba.pydata.org) is one of the unique cases where has true parallel multi-threading, being able to overcome the GIL.
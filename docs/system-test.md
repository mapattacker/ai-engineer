# Systems Test

System Tests, aks integration tests, tests the entire system, with all components integrated to detect any issues arising from it.


## Memory Leak

Memory leak is an insidious fault in the code that can consume the entire memory (RAM or GPU) over time & crash the application, together with other applications in the same server. To detect this, we usually need to run multiple requests over a period of time to the API to see if the memory builds up. We can use an app I developed to do this; [streamlit-memoryleak-checker](https://github.com/mapattacker/streamlit-memoryleak).

![](https://github.com/mapattacker/ai-engineer/blob/master/images/memory.png?raw=true)
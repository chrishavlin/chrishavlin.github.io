---
title: "Plotting Dask Futures"
date: 2021-06-15T17:35:25-05:00
draft: false
tags: ["code", "dask"]
categories: ["demo", "tutorials"]
---

In working on some yt-dask refactoring a question came up as to whether or not we could return data from distributed dask workflows as it completes and do something with it. For example, if we're building a profile plot (a souped up histogram of sorts), can we update our bin statistics as data chunks get read in? I've mostly been concentrating on reading in particle data into delayed `dask.array` objects using `dask.delayed` and `dask.compute`, but after a bit of digging it turns out it's pretty easy to add callbacks to dask tasks submitted to a client.

When you spin up a dask client,

```python
from dask.distributed import Client
client = Client()
```

and submit some task to call a function, `func_handle`, with an argument, `func_arg`:

```python
do_a_thing = client.submit(func_handle, func_arg)
```

you get back a [Future](https://docs.dask.org/en/latest/futures.html). It turns out that there is a context manager, `dask.distributed.as_completed`, that let's you write code to operate on the results of a future as it comes back ([link to dask docs](https://docs.dask.org/en/latest/futures.html#waiting-on-futures)). So I spent a little time fiddling with getting a `matplotlib` figure to dynamically update as dask futures return. Turned out to be straightforward and might be useful for understanding how to add more complex callbacks in dask workflows!  

First, let's import everything, set our `matplotlib` backend to `notebook` (so that our later calls to draw our `matplotlib` canvas are rendered immediately) and spin up a dask client:


```python
from dask.distributed import Client, as_completed
import numpy as np
import matplotlib.pyplot as plt
import time

%matplotlib notebook

client = Client()
```

So now let's define the function that we'll submit to our client:

```python
x = np.linspace(0, 1, 100)
def dy(timedelay):
    time.sleep(timedelay)
    return np.random.rand(x.shape[0])
```

this function represents an expensive computation that we want to visualize as it returns. In the following cell, the call to `futures = client.map(dy, np.random.rand(30))` submits 30 jobs to our client with a random delay between 0 and 1 seconds. Using the `as_completed` context, we'll sum up all of our returned random arrays as they are calculated and plot the current sum:

```python
futures = client.map(dy, np.random.rand(30))
y = np.zeros(x.shape)
f, axs = plt.subplots(1)
for future in as_completed(futures):
    dyvals = future.result()
    y += dyvals         
    axs.set_ylim([0, 20])
    axs.plot(x, y)   
    f.canvas.draw()
```

And here's a video of what it looks like:

{{< youtube 1ry3bBBO398 >}}

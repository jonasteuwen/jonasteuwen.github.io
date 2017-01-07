---
layout: post
title:  "Fill a numpy array using the multiprocessing module"
date:   2017-01-07
categories: numpy multiprocessing
comments: true
---
In one of my projects I had to fill a large array value by value, where each computation lasted up to 30 seconds. Since I had 32 cores at my disposal, I started considering if I could use the multiprocessing module of Python. This module provides a way to side step the global interpreter lock by using subprocesses, for more details see [the python multiprocessing module][python-multi]. However, the array is local to the subprocess, so we need to do something slightly smarted. Luckily the multiprocessing module, and the numpy module provide an interface to C compatible data types which can be inherited by child processes.

You can do this in two steps:

1. Write a piece of code that fills the array window by window.
2. Apply a ```python Pool()``` to the function.

The first part is rather straightforward. This code fills a matrix window by window to match a random matrix. To use it for your own project extrapolate from there.

```python
import numpy as np
import itertools

X = np.random.random((100, 100)) #  A random 100x100 matrix
tmp = np.zeros((100, 100)) #  Placeholder


def fill_per_window(args):
    window_x, window_y = args

    for i in range(window_x, window_y + 2):
        for j in range(window_x, window_y + 2):
            tmp[i, j] = X[i, j]


window_idxs = [(i, j) for i, j in
               itertools.product(range(0, 100, 2), range(0, 100, 2))]

for idx in window_idxs:
    fill_per_window(idx)

print(np.array_equal(X, tmp))
```
Next we apply sharedctypes and the multiprocessing module. See the [python docs][python-ctypes] and the [scipy docs][scipy-ctypes] for further details.

```python
import numpy as np
import itertools
from multiprocessing import Pool #  Process pool
from multiprocessing import sharedctypes

size = 100
block_size = 4

X = np.random.random((size, size))
result = np.ctypeslib.as_ctypes(np.zeros((size, size)))
shared_array = sharedctypes.RawArray(result._type_, result)


def fill_per_window(args):
    window_x, window_y = args
    tmp = np.ctypeslib.as_array(shared_array)

    for idx_x in range(window_x, window_x + block_size):
        for idx_y in range(window_y, window_y + block_size):
            tmp[idx_x, idx_y] = X[idx_x, idx_y]


window_idxs = [(i, j) for i, j in
               itertools.product(range(0, size, block_size),
                                 range(0, size, block_size))]

p = Pool(processes=32)
res = p.map(fill_per_window, window_idxs)
result = np.ctypeslib.as_array(shared_array)

print(np.array_equal(X, result))
```
This works in Linux, if you want it to work in Windows, you need to initialize your pool as explained on [Reddit][reddit].

[python-multi]: https://docs.python.org/2/library/multiprocessing.html
[reddit]: https://www.reddit.com/r/Python/comments/j3qjb/parformatlabpool_replacement/
[python-ctypes]: https://docs.python.org/2/library/multiprocessing.html#module-multiprocessing.sharedctypes
[scipy-ctypes]: https://docs.scipy.org/doc/numpy/reference/routines.ctypeslib.html

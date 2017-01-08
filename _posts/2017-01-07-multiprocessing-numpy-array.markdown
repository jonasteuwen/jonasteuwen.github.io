---
layout: post
title:  "Fill a numpy array using the multiprocessing module"
date:   2017-01-07
categories: numpy multiprocessing
comments: true
---
In one of my projects I had to fill a large array value by value, where each computation lasted up to 30 seconds. Since I had 32 cores at my disposal, I started considering if I could use the multiprocessing module of Python. This module provides a way to side step the global interpreter lock by using subprocesses, for more details see [the python multiprocessing module][python-multi]. However, the array is local to the subprocess, so we need to do something slightly smarter. Luckily the multiprocessing module and numpy provide an interface to C compatible data types which can be inherited by the child processes.

# TL;DR
You can do this in two steps:

1. Write a piece of code that fills the array window by window.
2. Apply a ```Pool()``` to the function.

The first part is rather straightforward. This code fills a matrix window by window to equal a random matrix. To use it for your own projects extrapolate from there.

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
This is hardly useful if we cannot do this using the multiprocessing module. But we can, by applying sharedctypes. Check the [python docs][python-ctypes] and the [scipy docs][scipy-ctypes] for further details.

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

p = Pool()
res = p.map(fill_per_window, window_idxs)
result = np.ctypeslib.as_array(shared_array)

print(np.array_equal(X, result))
```

# Explanation and additional comments
The python interpreter has a [Global Interpreter Lock (GIL)][python-gil] which prevents more than one thread to execute bytecode at a time. The [multiprocessing module][python-multi] sidesteps this by using subprocesses instead of threads.

To be able to use the multiprocessing module on our code, we need to find a way to execute our code in parallel. In the example we fill a square array where each of the values can be computed independently of eachother. If there is some form of independence, you obviously need to extrapolate from here or try something smarter. The approach taken here is to split the larger array into smaller squares and run a process on each square. Because we want to side step the GIL we need to share the memory of the array somehow. Here is where the [sharedctypes][python-ctypes] comes in. This allows us to access the array as if it were a C array. Of course, as C is not dynamically typed as python is, we need to properly declare the type of the array. Numpy provides a convenient function to read in the C array as a numpy array. 

*Note:* You might get a lot of [PEP 3118][PEP-3118] buffer RunTime warnings which is because ctypes uses the [wrong](http://bugs.python.org/issue10744) buffer interface information. These can be safely ignored. If you wish to suppress these warnings, you can use the [warnings module][python-warnings] to suppress the warning using the ```catch_warnings``` method.
*Be aware:* The example code runs on Linux, but if you want to run it on Windows you will need to initialize your pool as explained on [Reddit][reddit].


[python-warnings]: https://docs.python.org/2/library/warnings.html
[python-gil]: https://docs.python.org/2/glossary.html#term-global-interpreter-lock
[python-multi]: https://docs.python.org/2/library/multiprocessing.html
[reddit]: https://www.reddit.com/r/Python/comments/j3qjb/parformatlabpool_replacement/
[python-ctypes]: https://docs.python.org/2/library/multiprocessing.html#module-multiprocessing.sharedctypes
[scipy-ctypes]: https://docs.scipy.org/doc/numpy/reference/routines.ctypeslib.html
[PEP-3118]: https://www.python.org/dev/peps/pep-3118/

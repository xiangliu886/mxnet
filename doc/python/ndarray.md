NDArray API
===========

NDArray package (`mxnet.ndarray`) contains tensor operations similar to `numpy.ndarray`. The syntax is similar except for some additional calls to deal with I/O and multi-devices.

Create NDArray
--------------
Like `numpy`, you could create `mxnet.ndarray` like followings:
```python
>>> import mxnet as mx
>>> a = mx.nd.zeros((100, 50))              # all-zero array of dimension 100x50
>>> b = mx.nd.ones((256, 32, 128, 1))       # all-one array of dimension 256x32x128x1
>>> c = mx.nd.array([[1, 2, 3], [4, 5, 6]]) # initialize array with contents
```

NDArray operations
-------------------
We provide some basic ndarray operations like arithmetic and slice operations. More operations are coming in handy!

### Arithmetic operations
```python
>>> import mxnet as mx
>>> a = mx.nd.zeros((100, 50))
>>> a.shape
(100L, 50L)
>>> b = mx.nd.ones((100, 50))
>>> c = a + b
>>> d = a - b     # c and d will be calculated in parallel here!
>>> b += d        # inplace operation, b's contents will be modified, but c and d won't be affected.
```

### Slice operations
```python
>>> import mxnet as mx
>>> a = mx.nd.zeros((100, 50))
>>> a[0:10] = 1   # first 10 rows will become 1
```

Conversion from/to `numpy.ndarray` and I/O
--------------------------------
MXNet NDArray supports pretty nature way to convert from/to `mxnet.ndarray` to/from `numpy.ndarray`:
```python
>>> import mxnet as mx
>>> import numpy as np
>>> a = np.array([1,2,3])
>>> b = mx.nd.array(a)                  # convert from numpy array
>>> b
<mxnet.ndarray.NDArray object at ...>
>>> b.asnumpy()                         # convert to numpy array
array([ 1., 2., 3.], dtype=float32)
```

We also provide two convenient functions to help save and load file from I/O:
```python
>>> import mxnet as mx
>>> a = mx.nd.zeros((100, 200))
>>> mx.nd.save("/path/to/array/file", a)
>>> mx.nd.save("s3://path/to/s3/array", a)
>>> mx.nd.save("hdfs://path/to/hdfs/array", a)
>>> from_file = mx.nd.load("/path/to/array/file")
>>> from_s3 = mx.nd.load("s3://path/to/s3/array")
>>> from_hdfs = mx.nd.load("hdfs://path/to/hdfs/array")
```
The good thing about using the above `save` and `load` interface is that:
- You could use the format across all `mxnet` language bindings.
- Already support S3 and HDFS.

Multi-device support
-------------------
The device information is stored in `mxnet.Context` structure. When creating ndarray in mxnet, user could either use the context argument (default is CPU context) to create arrays on specific device or use the `with` statement as follows:
```python
>>> import mxnet as mx
>>> cpu_a = mx.nd.zeros((100, 200))
>>> cpu_a.context
cpu(0)
>>> with mx.Context(mx.gpu(0)):
>>>   gpu_a = mx.nd.ones((100, 200))
>>> gpu_a.context
gpu(0)
>>> ctx = mx.Context(mx.gpu(0))
>>> gpu_b = mx.nd.zeros((100, 200), ctx)
>>> gpu_b.context
gpu(0)
```

Currently, we *DO NOT* allow operations among arrays from different contexts. To allow this, use `copyto` member function to copy the content to different devices and continue computation:
```python
>>> import mxnet as mx
>>> x = mx.nd.zeros((100, 200))
>>> with mx.Context(mx.gpu(0)):
>>>   y = mx.nd.zeros((100, 200))
>>> z = x + y
mxnet.base.MXNetError: [13:29:12] src/ndarray/ndarray.cc:33: Check failed: lhs.ctx() == rhs.ctx() operands context mismatch
>>> cpu_y = mx.nd.zeros((100, 200))
>>> y.copyto(cpu_y)
>>> z = x + cpu_y
```

NDArray API Reference
---------------------

```eval_rst
.. automodule:: mxnet.ndarray
    :members:
```

NDArray Random API Reference
----------------------------

```eval_rst
.. automodule:: mxnet.random
    :members:
```


Context API Reference
---------------------

```eval_rst
.. automodule:: mxnet.context
    :members:
```

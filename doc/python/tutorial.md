MXNet Python Overview Tutorial
==============================

This page gives a general overview of MXNet's python package. MXNet contains a
mixed flavor of elements to bake flexible and efficient
applications. There are mainly three concepts:

* [NDArray](#ndarray-numpy-style-tensor-computations-on-cpus-and-gpus)
  offers matrix and tensor computations on both CPU and GPU, with automatic
  parallelization
* [Symbol](#symbol-and-automatic-differentiation) makes defining a neural
  network extremely easy, and provides automatic differentiation.
* [KVStore](#distributed-key-value-store) easy the data synchronization between
  multi-GPUs and multi-machines.

You can find more information at [Python Package Overview Page](index.md)

## NDArray: Numpy style tensor computations on CPUs and GPUs

`NDArray` is the basic operation unit in MXNet for matrix and tensor
computations. It is similar to `numpy.ndarray`, but with two additional
features:

1. **multiple devices**: all operations can be run on various devices including
CPU and GPU
2. **automatic parallelization**: all operations are automatically executed in
   parallel with each other

### Create and Initialization

We can create `NDArray` on either GPU or GPU

```python
>>> import mxnet as mx
>>> a = mx.nd.empty((2, 3)) # create a 2-by-3 matrix on cpu
>>> b = mx.nd.empty((2, 3), mx.gpu()) # create a 2-by-3 matrix on gpu 0
>>> c = mx.nd.empty((2, 3), mx.gpu(2)) # create a 2-by-3 matrix on gpu 2
>>> c.shape # get shape
(2L, 3L)
>>> c.context # get device info
gpu(2)
```

They can be initialized by various ways:

```python
>>> a = mx.nd.zeros((2, 3)) # create a 2-by-3 matrix and filled with 0
>>> b = mx.nd.ones((2, 3))  # create a 2-by-3 matrix and filled with 1
>>> b[:] = 2 # assign all elements of b with 2
```

We can copy the value from one to anther, even if they sit on different devices

```python
>>> a = mx.nd.ones((2, 3))
>>> b = mx.nd.zeros((2, 3), mx.gpu())
>>> a.copyto(b) # copy data from cpu to gpu
```

We can also convert `NDArray` to `numpy.ndarray`

```python
>>> a = mx.nd.ones((2, 3))
>>> b = a.asnumpy()
>>> type(b)
<type 'numpy.ndarray'>
>>> print b
[[ 1.  1.  1.]
 [ 1.  1.  1.]]
```

and verse vice

```python
>>> a = mx.nd.empty((2, 3))
>>> a[:] = np.random.uniform(-0.1, 0.1, a.shape)
>>> print a.asnumpy()
[[-0.06821112 -0.03704893  0.06688045]
 [ 0.09947646 -0.07700162  0.07681718]]
```

### Basic Operations

#### Elemental-wise operations

In default, `NDArray` performs elemental-wise operations:

```python
>>> a = mx.nd.ones((2, 3)) * 2
>>> b = mx.nd.ones((2, 3)) * 4
>>> print a.asnumpy()
[[ 4.  4.  4.]
 [ 4.  4.  4.]]
>>> c = a + b
>>> print c.asnumpy()
[[ 6.  6.  6.]
 [ 6.  6.  6.]]
>>> d = a * b
>>> print d.asnumpy()
[[ 8.  8.  8.]
 [ 8.  8.  8.]]
```

If two `NDArray` sit on different devices, we need explicitly move them into the
same one. The following example performing computations on GPU 0:

```python
>>> a = mx.nd.ones((2, 3)) * 2
>>> b = mx.nd.ones((2, 3), mx.gpu()) * 3
>>> c = a.copyto(mx.gpu()) * b
>>> print c.asnumpy()
[[ 6.  6.  6.]
 [ 6.  6.  6.]]
```

### Load and Save

There are two ways to save data to (load from) disks easily. The first way uses
`pickle`.  `NDArray` is pickle compatible, which means you can simply pickle the
NArray like what you did with `numpy.ndarray`.

```python
>>> import mxnet as mx
>>> import pickle as pkl

>>> a = mx.nd.ones((2, 3)) * 2
>>> data = pkl.dumps(a)
>>> b = pkl.loads(data)
>>> print b.asnumpy()
[[ 2.  2.  2.]
 [ 2.  2.  2.]]
```

On the second way, we directly dump a list of `NDArray` into disk in binary format.

```python
>>> a = mx.nd.ones((2,3))*2
>>> b = mx.nd.ones((2,3))*3
>>> mx.nd.save('mydata.bin', [a, b])
>>> c = mx.nd.load('mydata.bin')
>>> print c[0].asnumpy()
[[ 2.  2.  2.]
 [ 2.  2.  2.]]
>>> print c[1].asnumpy()
[[ 3.  3.  3.]
 [ 3.  3.  3.]]
```

We can also dump a dict.

```python
>>> mx.nd.save('mydata.bin', {'a':a, 'b':b})
>>> c = mx.nd.load('mydata.bin')
>>> print c['a'].asnumpy()
[[ 2.  2.  2.]
 [ 2.  2.  2.]]
>>> print c['b'].asnumpy()
[[ 3.  3.  3.]
 [ 3.  3.  3.]]
```

In addition, we have setup the distributed filesystem such as S3 and HDFS, we
can directly save to and load from them. For example:

```python
>>> mx.nd.save('s3://mybucket/mydata.bin', [a,b])
>>> mx.nd.save('hdfs///users/myname/mydata.bin', [a,b])
```

### Automatic Parallelization
`NDArray` can automatically execute operations in parallel. It is desirable when we
use multiple resources such as CPU, GPU cards, and CPU-to-GPU memory bandwidth.

For example, if we write `a += 1` followed by `b += 1`, and `a` is on CPU while
`b` is on GPU, then want to execute them in parallel to improve the
efficiency. Furthermore, data copy between CPU and GPU are also expensive, we
hope to run it parallel with other computations as well.

However, finding the codes can be executed in parallel by eye is hard. In the
following example, `a+=1` and `c*=3` can be executed in parallel, but `a+=1` and
`b*=3` should be in sequential.

```python
a = mx.nd.ones((2,3))
b = a
c = a.copyto(mx.cpu())
a += 1
b *= 3
c *= 3
```

Luckily, MXNet can automatically resolve the dependencies and
execute operations in parallel with correctness guaranteed. In other words, we
can write program as by assuming there is only a single thread, while MXNet will
automatically dispatch it into multi-devices, such as multi GPU cards or multi
machines.

It is achieved by lazy evaluation. Any operation we write down is issued into a
internal engine, and then returned. For example, if we run `a += 1`, it
returns immediately after pushing the plus operator to the engine. This
asynchronous allows us to push more operators to the engine, so it can determine
the read and write dependency and find a best way to execute them in
parallel.

The actual computations are finished if we want to copy the results into some
other place, such as `print a.asnumpy()` or `mx.nd.save([a])`. Therefore, if we
want to write highly parallelized codes, we only need to postpone when we need
the results.


## Symbol and Automatic Differentiation

NDArray is the basic computation unit in MXNet. Besides, MXNet provides a
symbolic interface, named Symbol, to simplify constructing neural networks. The
symbol combines both flexibility and efficiency. On one hand, it is similar to
the network configuration in [Caffe](http://caffe.berkeleyvision.org/) and
[CXXNet](https://github.com/dmlc/cxxnet), on the other hand, the symbol defines
the computation graph in [Theano](http://deeplearning.net/software/theano/).

### Basic Composition of Symbols

The following codes create a two layer perceptrons network:

```python
>>> import mxnet as mx
>>> net = mx.symbol.Variable('data')
>>> net = mx.symbol.FullyConnected(data=net, name='fc1', num_hidden=128)
>>> net = mx.symbol.Activation(data=net, name='relu1', act_type="relu")
>>> net = mx.symbol.FullyConnected(data=net, name='fc2', num_hidden=64)
>>> net = mx.symbol.Softmax(data=net, name='out')
>>> type(net)
<class 'mxnet.symbol.Symbol'>
```

Each symbol takes a (unique) string name. *Variable* often defines the inputs,
or free variables. Other symbols take a symbol as the input (*data*),
and may accept other hyper-parameters such as the number of hidden neurons (*num_hidden*)
or the activation type (*act_type*).

The symbol can be simply viewed as a function taking several arguments, whose
names are automatically generated and can be get by

```python
>>> net.list_arguments()
['data', 'fc1_weight', 'fc1_bias', 'fc2_weight', 'fc2_bias', 'out_label']
```

As can be seen, these arguments are the parameters need by each symbol:

- *data* : input data needed by the variable *data*
- *fc1_weight* and *fc1_bias* : the weight and bias for the first fully connected layer *fc1*
- *fc2_weight* and *fc2_bias* : the weight and bias for the second fully connected layer *fc2*
- *out_label* : the label needed by the loss

We can also specify the automatic generated names explicitly:

```python
>>> net = mx.symbol.Variable('data')
>>> w = mx.symbol.Variable('myweight')
>>> net = sym.FullyConnected(data=data, weight=w, name='fc1', num_hidden=128)
>>> net.list_arguments()
['data', 'myweight', 'fc1_bias']
```

### More Complicated Composition

MXNet provides well-optimized symbols (see
[src/operator](https://github.com/dmlc/mxnet/tree/master/src/operator)) for
commonly used layers in deep learning. We can also easily define new operators
in python.  The following example first performs an elementwise add between two
symbols, then feed them to the fully connected operator.

```python
>>> lhs = mx.symbol.Variable('data1')
>>> rhs = mx.symbol.Variable('data2')
>>> net = mx.symbol.FullyConnected(data=lhs + rhs, name='fc1', num_hidden=128)
>>> net.list_arguments()
['data1', 'data2', 'fc1_weight', 'fc1_bias']
```

We can also construct symbol in a more flexible way rather than the single
forward composition we addressed before.

```python
>>> net = mx.symbol.Variable('data')
>>> net = mx.symbol.FullyConnected(data=net, name='fc1', num_hidden=128)
>>> net2 = mx.symbol.Variable('data2')
>>> net2 = mx.symbol.FullyConnected(data=net2, name='net2', num_hidden=128)
>>> composed_net = net(data=net2, name='compose')
>>> composed_net.list_arguments()
['data2', 'net2_weight', 'net2_bias', 'compose_fc1_weight', 'compose_fc1_bias']
```

In the above example, *net* is used a function to apply to an existing symbol
*net*, the resulting *composed_net* will replace the original argument *data* by
*net2* instead.

### Argument Shapes Inference

Now we have known how to define the symbol. Next we can inference the shapes of
all the arguments it needed by given the input data shape.

```python
>>> net = mx.symbol.Variable('data')
>>> net = mx.symbol.FullyConnected(data=ent, name='fc1', num_hidden=10)
>>> arg_shape, out_shape, aux_shape = net.infer_shape(data=(100, 100))
>>> dict(zip(net.list_arguments(), arg_shape))
{'data': (100, 100), 'fc1_weight': (10, 100), 'fc1_bias': (10,)}
>>> out_shape
[(100, 10)]
```

The shape inference can be used as an earlier debugging mechanism to detect
shape inconsistency.

### Bind the Symbols and Run

Now we can bind the free variables of the symbol and perform forward and
backward.

```python
>>> in_shape = (128, 3, 100, 100) # minibatch_size, #channel, image_width, image_height
>>> executor = net.simple_bind(mx.gpu(), data = mx.nd.empty(in_shape, mx.gpu())
>>> # feed data and label..
>>> executor.forward()
>>> executor.backward()
>>> print executor.outputs[0].asnumpy()
```

### How Efficient is Symbolic API

In short, they design to be very efficienct in both memory and runtime.

The major reason for us to introduce Symbolic API, is to bring the efficient C++
operations in powerful toolkits such as cxxnet and caffe together with the
flexible dynamic NArray operations. All the memory and computation resources are
allocated statically during Bind, to maximize the runtime performance and memory
utilization.

The coarse grained operators are equivalent to cxxnet layers, which are
extremely efficient.  We also provide fine grained operators for more flexible
composition. Because we are also doing more inplace memory allocation, mxnet can
be ***more memory efficient*** than cxxnet/caffe, and gets to same runtime, with
greater flexiblity.

## Distributed Key-value Store

KVStore is a place for data sharing. We can think it as a single object shared
across different devices (GPUs and machines), where each device can push data in
and pull data out.

### Initialization

Let's first consider a simple example. It initializes
a (`int`, `NDAarray`) pair into the store, and then pull the value out.

```python
>>> kv = mx.kv.create('local') # create a local kv store.
>>> shape = (2,3)
>>> kv.init(3, mx.nd.ones(shape)*2)
>>> a = mx.nd.zeros(shape)
>>> kv.pull(3, out = a)
>>> print a.asnumpy()
[[ 2.  2.  2.]
 [ 2.  2.  2.]]
```

### Push, Aggregation, and Updater

For any key has been initialized, we can push a new value with the same shape to the key.

```python
>>> kv.push(3, mx.nd.ones(shape)*8)
>>> kv.pull(3, out = a) # pull out the value
>>> print a.asnumpy()
[[ 8.  8.  8.]
 [ 8.  8.  8.]]
```

The data for pushing can be on any device. Furthermore, we can push multiple
values into the same key, where KVStore will first sum all these
values and then push the aggregated value.

```python
>>> gpus = [mx.gpu(i) for i in range(4)]
>>> b = [mx.nd.ones(shape, gpu) for gpu in gpus]
>>> kv.push(3, b)
>>> kv.pull(3, out = a)
>>> print a.asnumpy()
[[ 4.  4.  4.]
 [ 4.  4.  4.]]
```

For each push, KVStore applies the pushed value into the value stored by a
`updater`. The default updater is `ASSGIN`, we can replace the default one to
control how data is merged.

```python
>>> def update(key, input, stored):
>>>     print "update on key: %d" % key
>>>     stored += input * 2
>>> kv.set_updater(update)
>>> kv.pull(3, out=a)
>>> print a.asnumpy()
[[ 4.  4.  4.]
 [ 4.  4.  4.]]
>>> kv.push(3, mx.nd.ones(shape))
update on key: 3
>>> kv.pull(3, out=a)
>>> print a.asnumpy()
[[ 6.  6.  6.]
 [ 6.  6.  6.]]
```

### Pull

We already see how to pull a single key-value pair. Similar to push, we can also
pull the value into several devices by a single call.

```python
>>> b = [mx.nd.ones(shape, gpu) for gpu in gpus]
>>> kv.pull(3, out = b)
>>> print b[1].asnumpy()
[[ 6.  6.  6.]
 [ 6.  6.  6.]]
```

### Handle a list of key-value pairs

All operations introduced so far are about a single key. KVStore also provides
the interface for a list of key-value pairs. For single device:

```python
>>> keys = [5, 7, 9]
>>> kv.init(keys, [mx.nd.ones(shape)]*len(keys))
>>> kv.push(keys, [mx.nd.ones(shape)]*len(keys))
update on key: 5
update on key: 7
update on key: 9
>>> b = [mx.nd.zeros(shape)]*len(keys)
>>> kv.pull(keys, out = b)
>>> print b[1].asnumpy()
[[ 3.  3.  3.]
 [ 3.  3.  3.]]
```

For multi-devices:

```python
>>> b = [[mx.nd.ones(shape, gpu) for gpu in gpus]] * len(keys)
>>> kv.push(keys, b)
update on key: 5
update on key: 7
update on key: 9
>>> kv.pull(keys, out = b)
>>> print b[1][1].asnumpy()
[[ 11.  11.  11.]
 [ 11.  11.  11.]]
```

### Multiple machines
Base on parameter server. The `updater` will runs on the server nodes.
This section will be updated when the distributed version is ready.


<!-- ## How to Choose between APIs -->

<!-- You can mix them all as much as you like. Here are some guidelines -->
<!-- * Use Symbolic API and coarse grained operator to create established structure. -->
<!-- * Use fine-grained operator to extend parts of of more flexible symbolic graph. -->
<!-- * Do some dynamic NArray tricks, which are even more flexible, between the calls of forward and backward of executors. -->

<!-- We believe that different ways offers you different levels of flexibilty and -->
<!-- efficiency. Normally you do not need to be flexible in all parts of the -->
<!-- networks, so we allow you to use the fast optimized parts, and compose it -->
<!-- flexibly with fine-grained operator or dynamic NArray. We believe such kind of -->
<!-- mixture allows you to build the deep learning architecture both efficiently and -->
<!-- flexibly as your choice. To mix is to maximize the peformance and flexiblity. -->

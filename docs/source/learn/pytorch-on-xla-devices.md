# PyTorch on XLA Devices

PyTorch runs on XLA devices, like TPUs, with the [torch_xla package](https://github.com/pytorch/xla/). This document describes how
to run your models on these devices.

## Creating an XLA Tensor

PyTorch/XLA adds a new `xla` device type to PyTorch. This device type
works just like other PyTorch device types. For example, here's how to
create and print an XLA tensor:

``` python
import torch
import torch_xla
import torch_xla.core.xla_model as xm

t = torch.randn(2, 2, device='xla')
print(t.device)
print(t)
```

This code should look familiar. PyTorch/XLA uses the same interface as
regular PyTorch with a few additions. Importing `torch_xla` initializes
PyTorch/XLA, and `torch_xla.device()` returns the current XLA device. This
may be a CPU or TPU depending on your environment.

## XLA Tensors are PyTorch Tensors

PyTorch operations can be performed on XLA tensors just like CPU or CUDA
tensors.

For example, XLA tensors can be added together:

``` python
t0 = torch.randn(2, 2, device='xla')
t1 = torch.randn(2, 2, device='xla')
print(t0 + t1)
```

Or matrix multiplied:

``` python
print(t0.mm(t1))
```

Or used with neural network modules:

``` python
l_in = torch.randn(10, device='xla')
linear = torch.nn.Linear(10, 20).to('xla')
l_out = linear(l_in)
print(l_out)
```

Like other device types, XLA tensors only work with other XLA tensors on
the same device. So code like

``` python
l_in = torch.randn(10, device='xla')
linear = torch.nn.Linear(10, 20)
l_out = linear(l_in)
print(l_out)
# Input tensor is not an XLA tensor: torch.FloatTensor
```

will throw an error since the `torch.nn.Linear` module is on the CPU.

## Running Models on XLA Devices

Building a new PyTorch network or converting an existing one to run on
XLA devices requires only a few lines of XLA-specific code. The
following snippets highlight these lines when running on a single device
and multiple devices with XLA multi-processing.

(running-on-a-single-xla-device)=

### Running on a Single XLA Device

The following snippet shows a network training on a single XLA device:

``` python
import torch_xla.core.xla_model as xm

device = torch_xla.device()
model = MNIST().train().to(device)
loss_fn = nn.NLLLoss()
optimizer = optim.SGD(model.parameters(), lr=lr, momentum=momentum)

for data, target in train_loader:
  optimizer.zero_grad()
  data = data.to(device)
  target = target.to(device)
  output = model(data)
  loss = loss_fn(output, target)
  loss.backward()

  optimizer.step()
  torch_xla.sync()
```

This snippet highlights how easy it is to switch your model to run on
XLA. The model definition, dataloader, optimizer and training loop can
work on any device. The only XLA-specific code is a couple lines that
acquire the XLA device and materializing the tensors. Calling `torch_xla.sync()`
at the end of each training iteration causes XLA to execute its current
graph and update the model's parameters. See [](#xla-tensor-deep-dive)
for more on how XLA creates graphs and runs
operations.

(running-on-multiple-xla-devices-with-multi-processing)=

### Running on Multiple XLA Devices with Multi-processing

PyTorch/XLA makes it easy to accelerate training by running on multiple
XLA devices. The following snippet shows how:

``` python
import torch_xla
import torch_xla.core.xla_model as xm
import torch_xla.distributed.parallel_loader as pl

def _mp_fn(index):
  device = torch_xla.device()
  mp_device_loader = pl.MpDeviceLoader(train_loader, device)

  model = MNIST().train().to(device)
  loss_fn = nn.NLLLoss()
  optimizer = optim.SGD(model.parameters(), lr=lr, momentum=momentum)

  for data, target in mp_device_loader:
    optimizer.zero_grad()
    output = model(data)
    loss = loss_fn(output, target)
    loss.backward()
    xm.optimizer_step(optimizer)

if __name__ == '__main__':
  torch_xla.launch(_mp_fn, args=())
```

There are three differences between this multi-device snippet and the
previous single device snippet. Let's go over then one by one.

-   `torch_xla.launch()`
    -   Creates the processes that each run an XLA device.
    -   This function is a wrapper of multithreading spawn to allow user
        run the script with torchrun command line also. Each process
        will only be able to access the device assigned to the current
        process. For example on a TPU v4-8, there will be 4 processes
        being spawn up and each process will own a TPU device.
    -   Note that if you print the `torch_xla.device()` on each process you
        will see `xla:0` on all devices. This is because each process
        can only see one device. This does not mean multi-process is not
        functioning. The only execution is with PJRT runtime on TPU v2
        and TPU v3 since there will be `#devices/2` processes and each
        process will have 2 threads(check this
        [doc](https://github.com/pytorch/xla/blob/master/docs/pjrt.md#tpus-v2v3-vs-v4)
        for more details).
-   `MpDeviceLoader`
    -   Loads the training data onto each device.
    -   `MpDeviceLoader` can wrap on a torch dataloader. It can preload
        the data to the device and overlap the dataloading with device
        execution to improve the performance.
    -   `MpDeviceLoader` also call `torch_xla.sync()` for you every
        `batches_per_execution`(default to 1) batch being yield.
-   `xm.optimizer_step(optimizer)`
    -   Consolidates the gradients between devices and issues the XLA
        device step computation.
    -   It is pretty much a `all_reduce_gradients` +
        `optimizer.step()` + `torch_xla.sync()` and returns the loss being
        reduced.

The model definition, optimizer definition and training loop remain the
same.

> **NOTE:** It is important to note that, when using multi-processing,
> the user can start retrieving and accessing XLA devices only from
> within the target function of `torch_xla.launch()` (or any function
> which has `torch_xla.launch()` as parent in the call stack).

See the [full multiprocessing
example](https://github.com/pytorch/xla/blob/master/test/test_train_mp_mnist.py)
for more on training a network on multiple XLA devices with
multi-processing.

### Running on TPU Pods

Multi-host setup for different accelerators can be very different. This
doc will talk about the device independent bits of multi-host training
and will use the TPU + PJRT runtime(currently available on 1.13 and 2.x
releases) as an example.

Before you begin, please take a look at our user guide at
[here](https://cloud.google.com/tpu/docs/run-calculation-pytorch) which
will explain some Google Cloud basics like how to use `gcloud` command
and how to setup your project. You can also check
[here](https://cloud.google.com/tpu/docs/how-to) for all Cloud TPU
Howto. This doc will focus on the PyTorch/XLA perspective of the Setup.

Let's assume you have the above mnist example from above section in a
`train_mnist_xla.py`. If it is a single host multi device training, you
would ssh to the TPUVM and run command like

    PJRT_DEVICE=TPU python3 train_mnist_xla.py

Now in order to run the same models on a TPU v4-16 (which has 2 host,
each with 4 TPU devices), you will need to - Make sure each host can
access the training script and training data. This is usually done by
using the `gcloud scp` command or `gcloud ssh` command to copy the
training scripts to all hosts. - Run the same training command on all
hosts at the same time.

``` console
    gcloud alpha compute tpus tpu-vm ssh $USER-pjrt --zone=$ZONE --project=$PROJECT --worker=all --command="PJRT_DEVICE=TPU python3 train_mnist_xla.py"
```

Above `gcloud ssh` command will ssh to all hosts in TPUVM Pod and run
the same command at the same time..

> **NOTE:** You need to run run above `gcloud` command outside of the
> TPUVM vm.

The model code and training script is the same for the multi-process
training and the multi-host training. PyTorch/XLA and the underlying
infrastructure will make sure each device is aware of the global
topology and each device's local and global ordinal. Cross-device
communication will happen across all devices instead of local devices.

For more details regarding PJRT runtime and how to run it on pod, please
refer to this
[doc](https://github.com/pytorch/xla/blob/master/docs/pjrt.md#tpu). For
more information about PyTorch/XLA and TPU pod and a complete guide to
run a resnet50 with fakedata on TPU pod, please refer to this
[guide](https://cloud.google.com/tpu/docs/pytorch-pods).

## XLA Tensor Deep Dive

Using XLA tensors and devices requires changing only a few lines of
code. But even though XLA tensors act a lot like CPU and CUDA tensors,
their internals are different. This section describes what makes XLA
tensors unique.

### XLA Tensors are Lazy

CPU and CUDA tensors launch operations immediately or eagerly. XLA
tensors, on the other hand, are lazy. They record operations in a graph
until the results are needed. Deferring execution like this lets XLA
optimize it. A graph of multiple separate operations might be fused into
a single optimized operation, for example.

Lazy execution is generally invisible to the caller. PyTorch/XLA
automatically constructs the graphs, sends them to XLA devices, and
synchronizes when copying data between an XLA device and the CPU.
Inserting a barrier when taking an optimizer step explicitly
synchronizes the CPU and the XLA device. For more information about our
lazy tensor design, you can read [this
paper](https://arxiv.org/pdf/2102.13267.pdf).

### Memory Layout

The internal data representation of XLA tensors is opaque to the user.
They do not expose their storage and they always appear to be
contiguous, unlike CPU and CUDA tensors. This allows XLA to adjust a
tensor's memory layout for better performance.

### Moving XLA Tensors to and from the CPU

XLA tensors can be moved from the CPU to an XLA device and from an XLA
device to the CPU. If a view is moved then the data its viewing is also
copied to the other device and the view relationship is not preserved.
Put another way, once data is copied to another device it has no
relationship with its previous device or any tensors on it. Again,
depending on how your code operates, appreciating and accommodating this
transition can be important.

### Saving and Loading XLA Tensors

XLA tensors should be moved to the CPU before saving, as in the
following snippet:

``` python
import torch
import torch_xla
import torch_xla.core.xla_model as xm

device = torch_xla.device()

t0 = torch.randn(2, 2, device=device)
t1 = torch.randn(2, 2, device=device)

tensors = (t0.cpu(), t1.cpu())

torch.save(tensors, 'tensors.pt')

tensors = torch.load('tensors.pt')

t0 = tensors[0].to(device)
t1 = tensors[1].to(device)
```

This lets you put the loaded tensors on any available device, not just
the one on which they were initialized.

Per the above note on moving XLA tensors to the CPU, care must be taken
when working with views. Instead of saving views it is recommended that
you recreate them after the tensors have been loaded and moved to their
destination device(s).

A utility API is provided to save data by taking care of previously
moving it to CPU:

``` python
import torch
import torch_xla
import torch_xla.core.xla_model as xm

xm.save(model.state_dict(), path)
```

In case of multiple devices, the above API will only save the data for
the master device ordinal (0).

In case where memory is limited compared to the size of the model
parameters, an API is provided that reduces the memory footprint on the
host:

``` python
import torch_xla.utils.serialization as xser

xser.save(model.state_dict(), path)
```

This API streams XLA tensors to CPU one at a time, reducing the amount
of host memory used, but it requires a matching load API to restore:

``` python
import torch_xla.utils.serialization as xser

state_dict = xser.load(path)
model.load_state_dict(state_dict)
```

Directly saving XLA tensors is possible but not recommended. XLA tensors
are always loaded back to the device they were saved from, and if that
device is unavailable the load will fail. PyTorch/XLA, like all of
PyTorch, is under active development and this behavior may change in the
future.

### Unexpected Tensor Materialization During AOT (ahead of time) Tracing

While tensor materialization is normal for JIT workflow, it is not expected during traced inference (i.e. [AOT model tracing in AWS Neuron](https://awsdocs-neuron.readthedocs-hosted.com/en/latest/frameworks/torch/torch-neuronx/programming-guide/inference/trace-vs-xla-lazytensor.html)).
When working with traced inference, developers may encounter tensor materialization, which leads to graphs being compiled based on example input tensor value and unexpected program behavior.
Therefore we need to take advantage of PyTorch/XLA's debugging flags to identify when unexpected tensor materialization happens and make appropriate code changes to avoid tensor materialization.


A common issue occurs when tensor values are evaluated during model compilation (traced inference). Consider this example:
```python
def forward(self, tensor):
    if tensor[0] == 1:
        return tensor
    else:
        return tensor * 2
```

While this code can compile and run, it may lead to unexpected behavior because:

* The tensor value is being accessed during tracing (``tensor[0]``). 
* The resulting graph becomes fixed based on the tensor value available during tracing
* Developers might incorrectly assume the condition will be evaluated dynamically during inference
* The solution for the code above is to utilize the debugging flags below to catch the issue and modify the code. One example is to feed the flag through model configuration

See the updated code without tensor materialization:
```python
class TestModel(torch.nn.Module):
    def __init__(self, flag=1):
        super().__init__()
        # the flag should be pre-determined based on the model configuration
        # it should not be an input of the model during runtime
        self.flag = flag

    def forward(self, tensor):
        if self.flag:
            return tensor
        else:
            return tensor * 2
```


#### Debugging Flags
To help catch tensor materialization issues, PyTorch/XLA provides two useful approaches:

1. Enable warning messages for tensor materialization:
```
import os
os.environ['PT_XLA_DEBUG_LEVEL'] = '2'
```

2. Disable graph execution to catch issues during development:
```
import torch_xla
torch_xla._XLAC._set_allow_execution(False)
```

#### Recommendations

Using these flags during development can help identify potential issues early in the development cycle. The recommended approach is to:

* Use ``PT_XLA_DEBUG_LEVEL=2`` during initial development to identify potential materialization points
* Apply ``_set_allow_execution(False)`` when you want to ensure no tensor materialization occurs during tracing
* When you see warnings or errors related the tensor materialization, look into the code path and make appropriate changes. The example above moved the flag to the `__init__` function which does not depend on the model input during runtime.

For more detailed debugging information, refer to the [XLA troubleshoot](https://github.com/pytorch/xla/blob/master/docs/source/learn/troubleshoot.md#pytorchxla-debugging-tool).


## Compilation Caching

The XLA compiler converts the traced HLO into an executable which runs
on the devices. Compilation can be time consuming. In case
the HLO doesn't change across executions, the compilation result can be
persisted to disk for reuse, significantly reducing development
iteration time.

NOTE:

* If the HLO changes between executions, a recompilation will still
  occur.
* When the version of `torch_xla` changes, a recompilation will occur
  (so that we can generate the executables using the latest compiler).

This is currently an opt-in API, which must be activated
before any computations are executed. Initialization is done through the
`initialize_cache` API:

``` python
import torch_xla.runtime as xr
xr.initialize_cache('YOUR_CACHE_PATH', readonly=False)
```

This will initialize a persistent compilation cache at the specified
path. The `readonly` parameter can be used to control whether the worker
will be able to write to the cache, which can be useful when a shared
cache mount is used for an SPMD workload.

If you want to use the persistent compilation cache in multi-process training (with `torch_xla.launch` or `xmp.spawn`), you should use different paths for different processes.

``` python
def _mp_fn(index):
  # cache init needs to happens inside the mp_fn.
  xr.initialize_cache(f'/tmp/xla_cache_{index}', readonly=False)
  ....

if __name__ == '__main__':
  torch_xla.launch(_mp_fn, args=())
```

If you don't have access to `index`, you can use `xr.global_ordinal()`. Check out the runnable example in [here](https://github.com/pytorch/xla/blob/master/examples/data_parallel/train_resnet_xla_ddp.py).

## Further Reading

Additional documentation is available at the [PyTorch/XLA
repo](https://github.com/pytorch/xla/). More examples of running
networks on TPUs are available
[here](https://github.com/pytorch-tpu/examples).

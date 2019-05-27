# torch2trt: PyTorch Module Instance to TensorRT

## Install

1. install pytorch 1.1+

2. download TensorRT 5.1.2.2+, install tensorrt python package, add TensorRT libraries to LD_LIBRARY_PATH.

3. clone this project, run ```python setup.py install```

## Usage

### Basic Example


```Python
import torch
import torchvision
import tensorrt as trt
import torch2trt
import numpy as np 
TRT_LOGGER = trt.Logger(trt.Logger.WARNING)

net = torchvision.models.inception_v3(pretrained=True).eval()
inputs = torch.rand(1, 3, 299, 299)
graph_pth = torch2trt.GraphModule(net, inputs, param_exclude=".*AuxLogits.*")
# run in pytorch debug mode, like torch_net(...)
torch_mode_out = graph_pth(inputs)
# you can convert another module or function:
def toy_example(x):
    return torch.softmax(x, 1), torch.sigmoid(x)
graph_pth_toy = torch2trt.GraphModule(toy_example, torch_mode_out)
probs, sigmoid = graph_pth_toy(torch_mode_out, verbose=True)

with trt.Builder(TRT_LOGGER) as builder, builder.create_network() as trt_net:
    builder.max_workspace_size = 1 << 30
    # you can use either trt.ITensor or torch.Tensor as inputs.
    # need trt_network to enter tensorrt mode, otherwise pytorch mode
    with torch2trt.trt_network(trt_net): # must use this to enter trt mode
        img = trt_net.add_input(name="image", shape=[3, 299, 299], dtype=trt.float32)
        trt_mode_out = graph_pth(img, verbose=True) # call graph_pth like torch module call
        # plugin = net.add_plugin_v2(trt_mode_out, ...) # add custom operator
        # trt_mode_out = plugin.get_output(0)
        # use another module here:
        trt_mode_out, sigmoid = graph_pth_toy(trt_mode_out)
    trt_mode_out.name = "output_softmax"
    sigmoid.name = "output_sigmoid"
    trt_net.mark_output(tensor=trt_mode_out)
    trt_net.mark_output(tensor=sigmoid)
    with builder.build_cuda_engine(trt_net) as engine:
        engine_bin = engine.serialize()
        with open("engine.bin", "wb") as f:
            f.write(engine_bin)
# get torch output for comparsion
# we need to execute some cuda functions before using TorchInferenceContext
probs, sigmoid = graph_pth_toy(torch_mode_out.cuda())
probs = probs.cpu().detach().numpy()
sigmoid = sigmoid.cpu().detach().numpy()
with trt.Runtime(TRT_LOGGER) as runtime:
    with open("engine.bin", "rb") as f:
        engine_bin = f.read()

    with runtime.deserialize_cuda_engine(engine_bin) as engine:
        with engine.create_execution_context() as trt_ctx:
            ctx = torch2trt.InferenceContext(trt_ctx)
            # all inputs are np.array, all inputs will be copied to page-locked host memory.
            output_dict = ctx.inference_async(inputs.cpu().detach().numpy())
            # sync inference and kwargs support. sync inference should only
            # be used in profiling
            output_dict = ctx.inference(image=inputs.cpu().detach().numpy())
            # start with tensorrt 5.0.2.6, tensorrt may not keep order of outputs.
            # so we must use name to get output
            # all outputs are numpy array
            output_softmax = output_dict["output_softmax"]
            output_sigmoid = output_dict["output_sigmoid"]
            print(np.linalg.norm(output_softmax.reshape(-1) - probs.reshape(-1)))
            print(np.linalg.norm(output_sigmoid.reshape(-1) - sigmoid.reshape(-1)))

            # now we use a inference class designed for pytorch:
            ctx = torch2trt.TorchInferenceContext(trt_ctx)
            # directly take a torch cuda tensor, have no host to device and device to host overhead.
            # WARNING: currently only support input tensor in device("cuda:0") and default cuda context
            # all output tensor are in device("cuda:0") too.
            # WARNING: don't support torch stream because we can't get handle of torch stream.
            # WARNING: don't support sync inference
            inputs = inputs.cuda()
            torch.cuda.synchronize() # do we need to synchronize default stream since we use custom stream?
            output_dict = ctx.inference_async(inputs)
            # all outputs are torch cuda tensor (shape maybe different from torch origin output)
            # all outputs are guaranteed to be synchronized.
            output_softmax = output_dict["output_softmax"]
            output_sigmoid = output_dict["output_sigmoid"]
            print(np.linalg.norm(output_softmax.cpu().detach().numpy().reshape(-1) - probs.reshape(-1)))
            print(np.linalg.norm(output_sigmoid.cpu().detach().numpy().reshape(-1) - sigmoid.reshape(-1)))
```

* Inputs and Outputs of TensorRT network

Inputs is inputs of net.forward, Outputs is outputs of net.forward.

* ```param_exclude``` and ```param_include```

torch2trt can't convert module with unused weights and buffers. if your module contains them, you need to use regex string to filter them.

### Add new handler

You can add handlers for missing nodes or tensorrt custom plugin. see ```handlers/ops.py``` for more examples.

```Python
@torch2trt.register_node_handler("aten::sum")
def aten_sum(inputs, attributes, scope):
    inp, axes, keepdim = inputs
    net = torch2trt.current_network()
    if net is not None and torch2trt.has_trt_tensor(inputs):
        axis_trt = _axes_to_trt_axis(axes, len(inp.shape))
        layer = net.add_reduce(inp, trt.ReduceOperation.SUM, axis_trt,
                               bool(keepdim))
        output = layer.get_output(0)
        output.name = scope
        layer.name = scope
        return [output]
    return [inp.sum(tuple(axes), keepdim=bool(keepdim))]
```

1. figure out the input format, you can check this [page](https://github.com/pytorch/pytorch/blob/master/aten/src/ATen/native/native_functions.yaml) as a reference.

2. use ```current_network``` to get current tensorrt INetworkDefinition instance. If None, the pytorch debug mode is enabled, you should implement pytorch code in this node handler for debugging.

3. use ```has_trt_tensor(inputs)``` to ensure inputs contains trt.ITensor. If there is no trt.ITensor, this node is a constant node and should be evaluated in pytorch mode.

4. return list of output tensors. all nodes MUST return list or tuple to handle node with multiple outputs.

5. for unsupported operators, write custom tensorrt plugin to handle this, then use net.add_plugin_v2 to add plugin in handler. you should also write custom torch jit operator for debugging. if you don't want to write jit operator, you can call torch2trt with several sub-module and connect them by tensorrt layers.

6. (optional) assign a name to output tensor and layer. the ```scope``` argument of handler is unique.

## Tips to write TensorRT-compatible modules

* inputs and outputs of net.forward can't be dict.

* tensorrt don't support type cast for now. all data should be float. avoid to use operations such as "to" and "type_as"

* all operations MUST use a static batch size.

* don't use any assign operation. TensorRT don't support assign/scatter.

* avoid to use any tensor create functions such as torch.zeros in forward code.

* write fused custom c++ jit operators for unsupported operations, also need to write a corresponding TensorRT plugin.

* don't add unused modules with weights to nn.Module. if you can't modify module code, use ```param_exclude``` in torch2trt to remove them.

* if your custom module have weights, you MUST use name contains "weight" or "bias", otherwise these weights will be filtered and cause error.

## Roadmap

- [ ] Deep integration between tensorrt and pytorch
  - [ ] Add a TensorRTModule to support train in pytorch, eval in tensorrt
- [ ] Deep integration between tvm and pytorch
  - [ ] Add TVM support
  - [ ] Add a TVMModule to support train in pytorch, eval in tvm (maybe impossible)

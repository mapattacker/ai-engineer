# Model Optimization

There are various ways to optimize a trained neural network model such that we can improve the inference performance with little impact on the precision.

We will concentrate on a few optimization techniques namely:

 * Quantization
 * CPU-GPU Flow
 * TensorRT

On the 2 popular libraries, pytorch & tensorflow. Keras is part of tensorflow's API already, so whatever tensorflow can do, keras should be able to too.

## Quantization

 * [Pytorch docs](https://pytorch.org/docs/stable/quantization.html)

## CPU Flow Control

Some operations in the neural network cannot run in GPU, hence data sometimes have to be transferred from CPU-GPU. This transfer if occurred many times, increases the latency. We can profile this by recording the events and time as a json file, and view in Chrome at the URL, `chrome://tracing` .

```python
import tensorflow as tf
import numpy as np

image = get_image_by_url("https://www.rover.com/blog/wp-content/uploads/2017/05/pug-tilt-960x540.jpg")
image_tensor = image.resize((300, 300))
image_tensor = np.array(image_tensor)
image_tensor = np.expand_dims(image_tensor, axis=0)

input_tensor_name = "image_tensor:0"
output_tensor_names = ['detection_boxes:0', 'detection_classes:0', 'detection_scores:0', 'num_detections:0']
ssd_mobilenet_v2_graph_def = load_graph_def(frozen_model_path)

with tf.Graph().as_default() as g:
    tf.import_graph_def(ssd_mobilenet_v2_graph_def, name='')
    input_tensor = g.get_tensor_by_name(input_tensor_name)
    output_tensors = [g.get_tensor_by_name(name) for name in output_tensor_names]

with tf.Session(graph=g) as sess:
    options = tf.RunOptions(trace_level=tf.RunOptions.FULL_TRACE)
    run_metadata = tf.RunMetadata()

    outputs = sess.run(output_tensors, feed_dict={input_tensor: image_tensor},
                       options=options, run_metadata=run_metadata)
    inference_time = (time.time()-start)*1000. # in ms
    
    # Write metadata
    fetched_timeline = timeline.Timeline(run_metadata.step_stats)
    chrome_trace = fetched_timeline.generate_chrome_trace_format()

with open('ssd_mobilenet_v2_coco_2018_03_29/exported_model/' + trace_filename, 'w') as f:
    f.write(chrome_trace)
```

For pytorch we can use the `torch.autograd.profiler`.

```python
import torch
import torchvision.models as models
import torch.autograd.profiler as profiler

model = models.resnet18().cuda()
inputs = torch.randn(5, 3, 224, 224).cuda()

with profiler.profile(record_shapes=True) as prof:
    with profiler.record_function("model_inference"):
        model(inputs)
        
prof.export_chrome_trace("trace.json")
```


We can specify certain nodes that are more CPU efficient, to run within CPU, thereby decreasing the data transfer and improving the inference performance. 

For example all the NonMaxSuppression are placed for CPU processing since most of the flow operations happen in this block.

```python
for node in ssd_mobilenet_v2_optimized_graph_def.node:
    if 'NonMaxSuppression' in node.name:
        node.device = '/device:CPU:0'
```



### References

* [Optimize NVIDIA GPU performance for efficient model inference](https://towardsdatascience.com/optimize-nvidia-gpu-performance-for-efficient-model-inference-f3e9874e9fdc)
* [HowTo profile TensorFlow](https://towardsdatascience.com/howto-profile-tensorflow-1a49fb18073d)
* [Find the bottleneck of your Keras model using TF trace](https://medium.com/@xianbao.qian/find-the-bottleneck-of-your-keras-model-using-tf-trace-2a6757aa372)
* [Pytorch profiler](https://pytorch.org/tutorials/recipes/recipes/profiler_recipe.html)
* [Tensorflow profiler](https://www.tensorflow.org/tensorboard/tensorboard_profiling_keras)

## Tensorflow Model Optimization

Tensorflow has has developed its own [library](https://www.tensorflow.org/model_optimization) for model optimization, which includes quantization, sparsity and pruning, and clustering. It can be installed via `pip install --user --upgrade tensorflow-model-optimization`.

## TensorRT

TensorFlow Integration for TensorRT (TF-TRT) is developed by Nvidia, which is a deep learning framework based on CUDA for inference acceleration. It optimizes and executes compatible subgraphs, allowing TensorFlow to execute the remaining graph. While you can still use TensorFlow's wide and flexible feature set, TensorRT will parse the model and apply optimizations to the portions of the graph wherever possible. 

![](https://github.com/mapattacker/ai-engineer/blob/master/images/tensorrt.png?raw=true)
Source: [Nvidia TensorRT](https://developer.nvidia.com/tensorrt)

Two of the most important optimizations are described below.

### Layer Fusion

During the TF-TRT optimization, TensorRT performs several important transformations and optimizations to the neural network graph. First, layers with unused output are eliminated to avoid unnecessary computation. 

Next, where possible, certain layers (such as convolution, bias, and ReLU) are fused to form a single layer. Another transformation is horizontal layer fusion, or layer aggregation, along with the required division of aggregated layers to their respective output. Horizontal layer fusion improves performance by combining layers that take the same source tensor and apply the same operations with similar parameters. 

![](https://github.com/mapattacker/ai-engineer/blob/master/images/tensorrt2.png?raw=true)
Source: [Speed up Tensorflow inference on GPUs - TensorRT](https://blog.tensorflow.org/2018/04/speed-up-tensorflow-inference-on-gpus-tensorRT.html)

### Quantization

Typically, model training is performed using 32-bit floating point (`FP32`) mathematics. Due to the backpropagation algorithm and weights updates, this high precision is necessary to allow for model convergence. Once trained, inference could be done in reduced precision (e.g. `FP16`) as the neural network architecture only requires a feed-forward network.

Reducing numerical precision allows for a smaller model with faster inferencing time, lower memory requirements, and more throughput.

There are certain requirements using quantization in TensorRT

 * `FP16` requires Nvidia GPUs that have hardware __tensor cores__
 * `INT8` is more [complex](https://on-demand.gputechconf.com/gtc/2017/presentation/s7310-8-bit-inference-with-tensorrt.pdf), and requires a __calibration process__ that minimizes the information loss when approximating the FP32 network with a limited 8-bit integer representation.

### Save Model

An example is used from a keras' model, and then saving it as a tensorflow protobuf model.

```python
import tensorflow as tf
from tensorflow.keras.applications.inception_v3 import InceptionV3

model = InceptionV3(weights='imagenet')
tf.saved_model.save(model, 'inceptionv3_saved_model')
```

We can view the model details using the `saved_model_cli`.

```bash
!saved_model_cli show --all --dir <model-directory>
```


### Benchmark Functions

To check that the new optimized model has faster inference & throughput, we want to prepare a function for loading the model... 

```python
def load_tf_saved_model_infer(input_saved_model_dir):
    """load model for inference"""
    print(f'Loading saved model {input_saved_model_dir}...')
    saved_model_loaded = tf.saved_model.load(input_saved_model_dir, tags=[tag_constants.SERVING])
    infer = saved_model.signatures['serving_default']
    print(infer.structured_outputs)
    return infer
```

We can use __batch inference__ to send many images to the GPU at once promotes parallel processing and improve throughput.

```python
def batch_input(batch_size=8):
    batched_input = np.zeros((batch_size, 299, 299, 3), dtype=np.float32)

    for i in range(batch_size):
        img_path = './data/img%d.JPG' % (i % 4)
        img = image.load_img(img_path, target_size=(299, 299))
        x = image.img_to_array(img)
        x = np.expand_dims(x, axis=0)
        x = preprocess_input(x)
        batched_input[i, :] = x

    batched_input = tf.constant(batched_input)
    return batched_input
```

... and lastly, a function for benchmarking the latency & throughput.

```python
def benchmark(batched_input, infer, N_warmup_run=50, N_run=1000):
    """benchmark latency & throughput
    
    Args
        batched_input: 
        infer (tf.float32): tensorflow model for inference
        N_warmup_run (int): no. of runs to warm up GPU
        N_run (int): no. of runs after warmup to benchmark
    
    Rets
        all_preds (list): predicted output
    """
    elapsed_time = []
    all_preds = []
    batch_size = batched_input.shape[0]

    for i in range(N_warmup_run):
        labeling = infer(batched_input)
        preds = labeling['predictions'].numpy()

    for i in range(N_run):
        start_time = time.time()

        labeling = infer(batched_input)
        preds = labeling['predictions'].numpy()

        end_time = time.time()
        elapsed_time = np.append(elapsed_time, end_time - start_time)

        all_preds.append(preds)

        if i % 50 == 0:
            print('Steps {}-{} average: {:4.1f}ms'.format(i, i+50, (elapsed_time[-50:].mean()) * 1000))

    print('Throughput: {:.0f} images/s'.format(N_run * batch_size / elapsed_time.sum()))
    return all_preds
```

### TRT Conversion

Tensorflow library has integrated Tensorrt, called [TrtGraphConverterV2](https://docs.nvidia.com/deeplearning/frameworks/tf-trt-user-guide/index.html) so we can call its API to convert the existing model. Below is a simple snippet on how to use it.

```python
from tensorflow.python.compiler.tensorrt import trt_convert as trt

converter = trt.TrtGraphConverterV2(
                input_saved_model_dir=None,
                conversion_params=TrtConversionParams(
                    precision_mode='FP32',
                    max_batch_size=1
                    minimum_segment_size=3,
                    max_workspace_size_bytes=8000000000,
                    use_calibration=True,
                    maximum_cached_engines=1,
                    is_dynamic_op=True,
                    rewriter_config_template=None,
                )
            )
converter.convert()
converter.save(output_saved_model_dir)
```

While below gives a function that allows more flexibility to change between various precision, and also calibrate the dataset when going to `int8`.

```python
from tensorflow.python.compiler.tensorrt import trt_convert as trt

def convert_to_trt_graph_and_save(precision_mode='float32',
                                  input_saved_model_dir='inceptionv3_saved_model',
                                  max_workspace_size_bytes=8000000000
                                  calibration_data=None):
    
    # select precision
    if precision_mode == 'float32':
        precision_mode = trt.TrtPrecisionMode.FP32
        converted_save_suffix = '_TFTRT_FP32'
    elif precision_mode == 'float16':
        precision_mode = trt.TrtPrecisionMode.FP16
        converted_save_suffix = '_TFTRT_FP16'
    elif precision_mode == 'int8':
        precision_mode = trt.TrtPrecisionMode.INT8
        converted_save_suffix = '_TFTRT_INT8'
        
    output_saved_model_dir = input_saved_model_dir + converted_save_suffix
    conversion_params = trt.DEFAULT_TRT_CONVERSION_PARAMS._replace(
        precision_mode=precision_mode, 
        max_workspace_size_bytes=max_workspace_size_bytes
    )
    converter = trt.TrtGraphConverterV2(
        input_saved_model_dir=input_saved_model_dir,
        conversion_params=conversion_params
    )

    # calibrate data if using int8
    if precision_mode == trt.TrtPrecisionMode.INT8:
        def calibration_input_fn():
            yield (calibration_data, )
        converter.convert(calibration_input_fn=calibration_input_fn)    
    else:
        converter.convert()

    # save tf-trt model
    converter.save(output_saved_model_dir=output_saved_model_dir)
```

We can check the signature of the new model again using the `saved_model_cli`.

```bash
!saved_model_cli show --all --dir <new-model-directory>
```

### Others

There are many other 3rd party optimization libraries, including:

 * [torch2trt](https://github.com/NVIDIA-AI-IOT/torch2trt)
 * [Keras inference time optimizer (KITO)](https://github.com/ZFTurbo/Keras-inference-time-optimizer)
 * [The Tensor Algebra SuperOptimizer (TASO)](https://github.com/jiazhihao/TASO)

### References

 * [Optimizing TensorFlow Serving performance with NVIDIA TensorRT](https://medium.com/tensorflow/optimizing-tensorflow-serving-performance-with-nvidia-tensorrt-6d8a2347869a)
---
layout: post
title: "Serving TensorFlow on 512MB Android Devices"
comments: true
description: "How we froze TF graphs, quantized weights, and shipped CNN/LSTM models on budget Android phones at Indus OS"
keywords: "tensorflow android, graph freezing, quantization, tf lite, mobile ml, 512mb, indus os"
---

At Indus OS we shipped ML models to Micromax and Lava phones with 512MB RAM. The OS itself consumed 300MB. That left roughly 200MB for all running apps. TensorFlow's default runtime was 20MB+ before loading a single model. We had to get inference working under 50MB total memory.

## Freezing Graphs and Stripping Ops

Training produces checkpoint files separate from the graph definition. For deployment you need a single self-contained file. The `freeze_graph` script merges the GraphDef with checkpoint weights by converting every `Variable` op into a `Const`.

```python
from tensorflow.python.tools import freeze_graph

freeze_graph.freeze_graph(
    input_graph='model.pb',
    input_checkpoint='model.ckpt',
    output_node_names='output/predictions',
    output_graph='frozen_model.pb',
    input_saver='',
    input_binary=True,
    restore_op_name='save/restore_all',
    filename_tensor_name='save/Const:0',
    clear_devices=True,
    initializer_nodes=''
)
```

After freezing, we ran `optimize_for_inference` to strip training-only nodes (dropout, batch norm updates, gradient ops). This alone cut our graph from 8MB to 3.2MB for the OCR model.

## Quantization and Selective Registration

Full float32 weights were too large. We quantized to 8-bit integers which cut model size by 4x with less than 1% accuracy loss on our Hindi OCR task.

```bash
# Quantize frozen graph
bazel-bin/tensorflow/tools/quantization/quantize_graph \
  --input=frozen_model.pb \
  --output=quantized_model.pb \
  --output_node_names=output/predictions \
  --mode=eightbit
```

The bigger win was selective op registration. Default TF Android builds include every op. Our models used maybe 30 ops out of 200+. We created a custom `ops_to_register.h` that only compiled the ops we needed. This brought the shared library from 12MB down to 4.1MB.

```bash
# Build with selective registration
bazel build -c opt --copt="-DSELECTIVE_REGISTRATION" \
  --copt="-DSUPPORT_SELECTIVE_REGISTRATION" \
  //tensorflow/contrib/android:libtensorflow_inference.so
```

## Memory Budget on a Real Device

Here is what our memory breakdown looked like on a Micromax Canvas Spark (512MB RAM, Mediatek MT6582):

| Component | Memory |
|-----------|--------|
| TF shared library | 4.1 MB |
| Quantized OCR model | 1.8 MB |
| Quantized TTS model | 6.2 MB |
| Input/output buffers | 3 MB |
| Runtime overhead | ~15 MB |
| **Total** | **~30 MB** |

## What Worked and What Didn't

Selective registration was the single biggest win. Without it, the project was dead on arrival. Quantization gave us the model size reduction we needed with acceptable accuracy.

What didn't work: TF Lite (still experimental at the time) crashed on several Mediatek chipsets. We also tried NNAPI but it simply wasn't available on Android 5.1 which was our target. We stuck with the native TF C API through JNI.

The lesson: on constrained devices, the framework overhead matters more than the model itself. We spent 70% of our optimization time on the runtime, not the model.

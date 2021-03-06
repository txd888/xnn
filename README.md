XNN: A C++ Prediction API that Wraps Caffe, MXNET and Python
=============================================================

Author: Wei Dong (wdong@wdong.org)

# Usage
```
#include <xnn.h>

...

xnn::set_mode(1); // 0 for CPU, 1 for GPU, doesn't affect theano
                  // use theanorc or env variable to tweak the
                  // behavior of theano.
int batch = 1;
xnn::Model *model = xnn::Model::create("model dir", batch);

cv::Mat image = cv::read("some.jpg", -1);
vector<float> out;

model->apply(image, &out);
model->apply(vector<cv::Mat>{image}, &out); 
```

At most batch images can be passed in each invokation of Model::apply.
If less than batch images are passed, the library internally pads the
input up to a whole batch for prediction, so the cost will be as if
batch images are predicted.
The returned vector will have the size of `(#category * batch)` or
`(out image size * #category * batch)`,
depending on whether the model does classification or
segmentation.


The library is still under development and doesn't yet coherently handle
classification and segmentation with different backends and the deployment
method is subject to changes.

# Building and Installation

The library depends on that Caffe, MXNet, Theano/Lasagne and other python
libraries are properly installed.  Use [these scripts](https://github.com/aaalgo/centos7-deep)
to install everything on a fresh CentOS 7 installation.

# Model Deployment
## Caffe
The model directory should contain the following files:
- caffe.model: copy the deploy.prototxt file.
- caffe.params: copy one of the ".caffemodel" file.
- caffe.blobs: text file containing the blob name to extract, usually "prob".  If multiple blob names are given, one on each line, all the blobs will be extracted and concatenated.
- caffe.mean[optional]: copy the xxx_mean.binaryproto file, or a text file containing 1-3 numbers providing the mean values of RGB channels.

If the input blob has an image size of (1, 1), then a segmentation
network is assumed, input images are not resized and output have
the same size as input except for number of channels.

Many pre-trained models can be found [here](https://github.com/BVLC/caffe/wiki/Model-Zoo).

## MXNet
The model directory should contain the following files:
- mxnet.symbol: the JSON model file.
- mxnet.params: model parameters.
- mxnet.meta: a JSON file specifying shape and mean information like the one below.
```
{"shape": 224, "mean": [123.68, 116.779, 103.939], "channels":3}
```

Many pre-trained models can be found [here](https://github.com/dmlc/mxnet-model-gallery).

## Python
The model directory is essentially a python module, which should contain the following files:
- `__init__.py`: typically empty.
- model.py: the python file containing the model.

The module model.py should provide two functions:

- `model.shape()`: returns (-1, channel, rows, cols)
- `model.load(shape)`: given shape, loads parameters and returns a function
that make prediction.

The following minimal script should be used to test that
a python model directory is properly deployed:

```
import numpy as np
import model

shape = model.shape()

bathc = 16
shape = (batch, shape[1], shape[2], shape[3])

pred_fn = model.load(shape)

input = np.zeros(shape, dtype=float)
output = pred_fn(input)
```

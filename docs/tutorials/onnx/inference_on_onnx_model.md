
# Running inference on MXNet/Gluon from an ONNX model

[Open Neural Network Exchange (ONNX)](https://github.com/onnx/onnx) provides an open source format for AI models. It defines an extensible computation graph model, as well as definitions of built-in operators and standard data types.

In this tutorial we will:
    
- learn how to load a pre-trained .onnx model file into MXNet/Gluon
- learn how to test this model using the sample input/output
- learn how to test the model on custom images

## Pre-requisite

To run the tutorial you will need to have installed the following python modules:
- [MXNet](http://mxnet.incubator.apache.org/install/index.html)
- [onnx](https://github.com/onnx/onnx) (follow the install guide)
- [onnx-mxnet](https://github.com/onnx/onnx-mxnet)
- matplotlib
- wget


```python
import numpy as np
import onnx_mxnet
import mxnet as mx
from mxnet import gluon, nd
%matplotlib inline
import matplotlib.pyplot as plt
import tarfile, os
import wget
import json
```

### Downloading supporting files
These are images and a vizualisation script


```python
image_folder = "images"
utils_file = "utils.py" # contain utils function to plot nice visualization
image_net_labels_file = "image_net_labels.json"
images = ['apron', 'hammerheadshark', 'dog', 'wrench', 'dolphin', 'lotus']
base_url = "https://raw.githubusercontent.com/dmlc/web-data/master/mxnet/doc/tutorials/onnx/{}?raw=true"

if not os.path.isdir(image_folder):
    os.makedirs(image_folder)
    for image in images:
        wget.download(base_url.format("{}/{}.jpg".format(image_folder, image)), image_folder)
if not os.path.isfile(utils_file):
    wget.download(base_url.format(utils_file))       
if not os.path.isfile(image_net_labels_file):
    wget.download(base_url.format(image_net_labels_file))  
```


```python
from utils import *
```

## Downloading a model from the ONNX model zoo

We download a pre-trained model, in our case the [vgg16](https://arxiv.org/abs/1409.1556) model, trained on [ImageNet](http://www.image-net.org/) from the [ONNX model zoo](https://github.com/onnx/models). The model comes packaged in an archive `tar.gz` file containing an `model.onnx` model file and some sample input/output data.


```python
base_url = "https://s3.amazonaws.com/download.onnx/models/" 
current_model = "vgg16"
model_folder = "model"
archive = "{}.tar.gz".format(current_model)
archive_file = os.path.join(model_folder, archive)
url = "{}{}".format(base_url, archive)
```

Create the model folder and download the zipped model


```python
os.makedirs(model_folder, exist_ok=True)
if not os.path.isfile(archive_file):  
    wget.download(url, model_folder)
```

Extract the model


```python
if not os.path.isdir(os.path.join(model_folder, current_model)):
    tar = tarfile.open(archive_file, "r:gz")
    tar.extractall(model_folder)
    tar.close()
```

The models have been pre-trained on ImageNet, let's load the label mapping of the 1000 classes.


```python
categories = json.load(open(image_net_labels_file, 'r'))
```

## Loading the model into MXNet Gluon


```python
onnx_path = os.path.join(model_folder, current_model, "model.onnx")
```

We get the symbol and parameter objects


```python
sym, params = onnx_mxnet.import_model(onnx_path)
```

We pick a context, CPU or GPU


```python
ctx = mx.cpu()
```

And load them into a MXNet Gluon symbol block. For ONNX models the default input name is `input_0`.


```python
net = gluon.nn.SymbolBlock(outputs=sym, inputs=mx.sym.var('input_0'))
net_params = net.collect_params()
for param in params:
    if param in net_params:
        net_params[param]._load_init(params[param], ctx=ctx)
```

We can now cache the computational graph through [hybridization](https://mxnet.incubator.apache.org/tutorials/gluon/hybrid.html) to gain some performance



```python
net.hybridize()
```

## Test using sample inputs and outputs
The model comes with sample input/output we can use to test that whether model is correctly loaded


```python
numpy_path = os.path.join(model_folder, current_model, 'test_data_0.npz')
sample = np.load(numpy_path, encoding='bytes')
inputs = sample['inputs']
outputs = sample['outputs']
```


```python
print("Input format: {}".format(inputs[0].shape))
print("Output format: {}".format(outputs[0].shape))
```

`Input format: (1, 3, 224, 224)` <!--notebook-skip-line-->


`Output format: (1, 1000)` <!--notebook-skip-line-->
    


We can visualize the network (requires graphviz installed)


```python
mx.visualization.plot_network(sym, shape={"input_0":inputs[0].shape}, node_attrs={"shape":"oval","fixedsize":"false"})
```




![png](https://raw.githubusercontent.com/dmlc/web-data/master/mxnet/doc/tutorials/onnx/network.png?raw=true)<!--notebook-skip-line-->



This is a helper function to run M batches of data of batch-size N through the net and collate the outputs into an array of shape (K, 1000) where K=MxN is the total number of examples (mumber of batches x batch-size) run through the network.


```python
def run_batch(net, data):
    results = []
    for batch in data:
        outputs = net(batch)
        results.extend([o for o in outputs.asnumpy()])
    return np.array(results)
```


```python
result = run_batch(net, nd.array([inputs[0]], ctx))
```


```python
print("Loaded model and sample output predict the same class: {}".format(np.argmax(result) == np.argmax(outputs[0])))
```

Loaded model and sample output predict the same class: True <!--notebook-skip-line-->


Good the sample output and our prediction match, now we can run against real data

## Test using real images


```python
TOP_P = 3 # How many top guesses we show in the visualization
```


Transform function to set the data into the format the network expects, (N, 3, 224, 224) where N is the batch size.


```python
def transform(img):
    return np.expand_dims(np.transpose(img, (2,0,1)),axis=0).astype(np.float32)
```


We load two sets of images in memory


```python
image_net_images = [plt.imread('images/{}.jpg'.format(path)) for path in ['apron', 'hammerheadshark','dog']]
caltech101_images = [plt.imread('images/{}.jpg'.format(path)) for path in ['wrench', 'dolphin','lotus']]
images = image_net_images + caltech101_images
```

And run them as a batch through the network to get the predictions

```python
batch = nd.array(np.concatenate([transform(img) for img in images], axis=0), ctx=ctx)
result = run_batch(net, [batch])
```


```python
plot_predictions(image_net_images, result[:3], categories, TOP_P)
```


![png](https://raw.githubusercontent.com/dmlc/web-data/master/mxnet/doc/tutorials/onnx/imagenet.png?raw=true)<!--notebook-skip-line-->


**Well done!** Looks like it is doing a pretty good job at classifying pictures when the category is a ImageNet label

Let's now see the results on the 3 other images


```python
plot_predictions(caltech101_images, result[3:7], categories, TOP_P)
```


![png](https://raw.githubusercontent.com/dmlc/web-data/master/mxnet/doc/tutorials/onnx/caltech101.png?raw=true)<!--notebook-skip-line-->


**Hmm, not so good...**  Even though predictions are close, they are not accurate, which is due to the fact that the ImageNet dataset does not contain `wrench`, `dolphin`, or `lotus` categories and our network has been trained on ImageNet.

Lucky for us, the [Caltech101 dataset](http://www.vision.caltech.edu/Image_Datasets/Caltech101/) has them, let's see how we can fine-tune our network to classify these categories correctly.

We show that in our next tutorials:

- Fine-tuning a ONNX Model using the modern imperative MXNet/Gluon API(Coming soon)
- Fine-tuning a ONNX Model using the symbolic MXNet/Module API(Coming soon)
    
<!-- INSERT SOURCE DOWNLOAD BUTTONS -->
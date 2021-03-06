# cnn-quantization

## Dependencies
- python3.x
- [pytorch](<http://www.pytorch.org>)
- [torchvision](<https://github.com/pytorch/vision>) to load the datasets, perform image transforms
- [pandas](<http://pandas.pydata.org/>) for logging to csv
- [bokeh](<http://bokeh.pydata.org>) for training visualization
- [scikit-learn](https://scikit-learn.org) for kmeans clustering
To install requirements run:
```
pip install torch torchvision bokeh pandas sklearn
```

## Data
- To run this code you need validation set from ILSVRC2012 data
- Configure your dataset path by providing --data "PATH_TO_ILSVRC" or copy ILSVRC dir to ~/datasets/ILSVRC2012.
- To get the ILSVRC2012 data, you should register on their site for access: <http://www.image-net.org/>

## Building cuda kernels for GEMMLOWP
To improve performance GEMMLOWP quantization was implemented in cuda and requires to compile kernels.

- Create virtual environment for python3 and activate:
```
virtualenv --system-site-packages -p python3 venv3
. ./venv3/bin/activate
```
- build kernels
```
cd kernels
./build_all.sh
```

## Prepare setup for Inference
We use a profiling data-set of approximately 256 images sampled from the training-set to collect statistics of activations at offline time. These are needed later to calculate the scale/clipping value of each activation tenor.
### Collect statistics
```
# Collect per layer statistics
python inference/inference_sim.py -a resnet50 -b 256 -sm collect -ac --qtype int4
# Collect per output channel statistics
python inference/inference_sim.py -a resnet50 -b 256 -sm collect -ac --qtype int4 -pcq_a
```
Statistics will be saved under ~/mxt_sim/statistics folder.

### Generate quantized model
```
python pytorch_quantizer/quantization/kmeans_quantization.py -a resnet50 -bits 4 -t quantize
```
Quantized model will be saved to ~/mxt_sim/models

### Run inference experiments
Post-training quantization of Res50 to 8-bit weights and 4-bit activations using the suggested quantization pipeline:
```
python inference/inference_sim.py -a resnet50 -b 512 -sm use --qtype int4 -pcq_w -pcq_a -c laplace
```
`* Prec@1 74.114 Prec@5 91.904`

Post-training quantization of Res50 to 4-bit weights and 8-bit activations using kmeans clustering:
```
python inference/inference_sim.py -a resnet50 -b 512 -sm use --qtype int8 -qm 4 -qw f32
```
`* Prec@1 74.242 Prec@5 91.764`

Post-training quantization of Res50 to 4-bit weights and 4-bit activations using the suggested quantization pipeline:
```
python inference/inference_sim.py -a resnet50 -b 512 -sm use --qtype int4 -pcq_w -pcq_a -c laplace -qm 4 -qw f32
```
`* Prec@1 72.182 Prec@5 90.634`

- Note that results could vary due to randomization of statistic gathering and kmeans initialization.  

## Solution for optimal clipping

To find optimal clipping values for the Laplace/Gaussian case, we numerically solve Equations (12)/ (A4)

[optimal_alpha.ipynb](optimal_alpha.ipynb)

Gaussian case, linear dependency
![Gaussian case](figures/opt_alpha_gaussian.png)

## Quantization with optimal clipping
In order to quantize tensor to M bit with optimal clipping we use GEMMLOWP quantization with small modification. We replace dynamic range in scale computation by 2*alpha where alpha is optimal clipping value.

Quantization code can be found here: 
[int_quantizer.py](pytorch_quantizer/quantization/qtypes/int_quantizer.py)

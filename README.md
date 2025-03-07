```text
    __     _    __     __             ____                          __  
   / /    (_)  / /_   / /_   ____    / __ )  ___    ____   _____   / /_ 
  / /    / /  / __/  / __ \ / __ \  / __  | / _ \  / __ \ / ___/  / __ \
 / /___ / /  / /_   / / / // /_/ / / /_/ / /  __/ / / / // /__   / / / /
/_____//_/   \__/  /_/ /_/ \____/ /_____/  \___/ /_/ /_/ \___/  /_/ /_/ 
                                                                        
```

# LithoBench: Benchmarking AI Computational Lithography for Semiconductor Manufacturing 

LithoBench dataset is a collection of circuit layout tiles for deep-learning-based lithography simulation and mask optimization. 
LithoBench consists of more than 120k tiles that are cropped from real circuit designs or synthesized according to the layout topologies of famous ILT testcases. 
The ground truths are generated by a famous lithography model in academia and an advanced ILT method. 
Based on the data, we provide a framework to design and evaluate deep neural networks (DNNs) with the data. 
The framework is used to benchmark state-of-the-art models on lithography simulation and mask optimization. 
We hope LithoBench can promote the research and development of computational lithography. 

## Installation

### Install Basic Dependencies

If you manage your python environments with anaconda, you can create a new environment with
```bash
conda create -n lithobench python=3.8
conda activate lithobench
```
To install the dependencies with pip, you can use
```bash
pip3 install -r requirements_pip.txt
```

You may install the dependencies with conda:
```bash
conda install --file requirements_conda.txt -c pytorch -c conda-forge
```
However, due to the complex environment solving, the process may be slow and the installed packages may be unsatisfactory. 
For example, you may get a CPU version of pytorch. 
Thus, if you want to use conda, you may install a GPU version of pytorch before you install other dependencies. 

Note that we develop LithoBench with python 3.8 and pytorch 1.10. 
We also tested LithoBench with pytorch 2.0. 
The system we use is Ubuntu 18 with Intel Xeon CPUs and NVIDIA GPUs. We also tested the program on CentOS 7. 

### Install adaptive-boxes

The python package *adaptive-boxes* is needed for shot counting. 
You can install the package in the *thirdparty/adaptive-boxes* folder. 
```
cd thirdparty/adaptive-boxes
pip3 install -e .
```

### Run ILT algorithms

You can test the ILT method in LithoBench with the following commands: 

*CurvMulti*
```
CUDA_VISIBLE_DEVICES=0 python3 pyilt/curvmulti.py
```

## Download the dataset

### LithoBench Data

Please download the data from

```
https://drive.google.com/file/d/1MzYiRRxi8Eu2L6WHCfZ1DtRnjVNOl4vu/view?usp=sharing
```

Put the lithomodels.tar.gz into the work/ directory and unzip it with: 

```
tar xvfz lithodata.tar.gz
```

### Pre-trained Models

Please download the pre-trained models from

```
https://drive.google.com/file/d/1N-VCv0gX49zzVWlwSs0yDqq2zKNQHKNB/view?usp=sharing
```

Put the lithomodels.tar.gz into the saved/ directory and unzip it with: 

```
tar xvfz lithomodels.tar.gz
```

### Code

You may also download the code from: 
```
https://drive.google.com/file/d/1zDc3s1DBlgoJ_yGTBpF0J5G35Aqs5l1-/view?usp=sharing
```

## Train and Test the Models

Please refer to scripts/runNeuralILT.sh. 

To train a model on MetalSet: 

```
python3 lithobench/train.py -m NeuralILT -s MetalSet -n 8 -b 12
```

>* "-m NeuralILT" specifies the NeuralILT model to train. 
>* "-s MetalSet" means training on MetalSet.
>* "-n" and "-b" decide the number of epochs and the batch size. 
>* Replacing "MetalSet" with "ViaSet" can train the model on ViaSet. 


To evaluate the model on MetalSet: 

```
python3 lithobench/test.py -m NeuralILT -s MetalSet -l saved/MetalSet_NeuralILT/net.pth --shots
```

>* By default, the trained model will be saved in saved/\<training set\>_\<model name\>/.
>* The sub-dataset specified by "-s" can be MetalSet, ViaSet, StdMetal, StdContact. 
>* Note that to evaluate the model on StdMetal, the trained model saved/MetalSet_NeuralILT/net.pth should also be used. To evaluate the model on StdContact, the trained model saved/ViaSet_NeuralILT/net.pth should also be used.
>* "--shots" enables the shot counting in the evaluation. It is slow so it is not enabled by default. 

LithoBench contains four models for lithography simulation, which located in lithobench/litho/. 
The four models for mask optimization are in lithobench/ilt/. 
The comments in the corresponding code show their performance on all sub-datasets. 

Due to the wierd behaviors of some complex-valued layers on multiple GPUs, it is suggested to run some models on a single GPU, e.g. CFNO, DOINN,etc. 

## Train and Test a New Model

lithobench/train.py and lithobench/test.py can train and test user-defined new models. 
For mask optimizatiom, a model is required to inherit the *lithobench.model.ModelILT* class. 
Similarly, a lithography simulation model should inherit the *lithobench.model.ModelLitho* class. 

For the pretraining and training of a model, users can implement the interfaces: 
```
pretrain(self, train_loader, val_loader, epochs=1)
train(self, train_loader, val_loader, epochs=1)
```
The *train_loader* and *val_loader* are PyTorch DataLoaders. 
For lithography simulation, the data loader outputs a batch of optimized masks, aerial images, and resist images. 
For mask optimization, the data loader outputs a batch of target images and reference optimized masks. 
It is okay to leave the *pretrain* function empty. 

For the evaluation of the results, users can use the interface: 
```
run(self, target)
```
For lithography simulation, this function should return two tensors containing the aerial and resist images. 
For mask optimization, it returns the tensor of optimized masks. 

To save and load the model weights, users should implement: 
```
save(self, filenames)
load(self, filenames)
```
By default, *filenames* is a file that contains the state dicts of the model. 

Users may use the reference lithography simulation model in their code by instantiate the following simulator: 
```
self.simLitho = pylitho.exact.LithoSim("./config/lithosimple.txt")
```
The lithography simulator accepts a batch of masks (shape=(B, 1, H, W)), and output the printed images at the maximum, nominal, and minimum cornors. 
For example, we can get the L2 loss and PVB loss by: 
```
printedNom, printedMax, printedMin = self.simLitho(mask)
l2loss = F.mse_loss(printedNom, target)
pvbloss = F.mse_loss(printedMax, printedMin)
loss = l2loss + pvbloss
```
*mask* is the output of the DNN model. *target* is the tensor of target images. 

## New Model -- an Example

Please refer to dev/swin.py and dev/run.sh. 

To train the new model "dev/swin.py" on MetalSet: 

```
python3 lithobench/train.py -m dev/swin.py -a SwinILT -i 512 -t ILT -o dev -s MetalSet -n 4 -b 4
```

>* "-m dev/swin.py" specifies path of the model. 
>* "-a SwinILT" indicated the alias and also the class name of the model
>* "-i 512" means that the image size is 512x512. 
>* "-t ILT" is for mask optimization. For lithography simulation, users can use "-t Litho". 
>* "-o dev" specifies the output directory of the training process
>* "-s MetalSet" specifies the dataset, which can be MetalSet and ViaSet. Note that StdMetal and StdContact are only for testing. 
>* "-n 4" is the number of epochs. 
>* "-b 4" is the batch size. 


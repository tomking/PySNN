# __PySNN__

Spiking neural network (SNN) framework written on top of PyTorch for efficient simulation of SNNs both on _**CPU**_ and _**GPU**_. The framework is intended for with correlation based learning methods. The library adheres to the highly modular and dynamic design of PyTorch, and does not require its user to learn a new framework like when using BindsNet. *This framework's power lies in the ease of defining and mixing new Neuron and Connection objects that seamlessly work together, even different versions, in a single network.*

PySNN is designed to truely be an engineering framework, providing mainly low level objects to its user that can be combined and mixed, just as in PyTorch. The biggest difference is that a network now consists of two types of modules, instead of the single nn.Module in regular PyTorch. These new modules are the pysnn.Neuron and pysnn.Connection. The Neuron object contains many of the methods that define a SNN. The Connection object really just is a small extension to a regular PyTorch Module that routes signals between different layers. More information can be found in the following sections.

<!-- Inspiration taken from [cuSNN](https://github.com/tudelft/cuSNN) and [bindsnet](https://github.com/Hananel-Hazan/bindsnet). -->

## __Installation__

Installation can be done with pip:

```bash
$ git clone https://github.com/BasBuller/PySNN
$ pip install PySNN/
```

If you want to make updates to the library without having to reinstall it, use the following install command instead:
```bash
$ pip install -e PySNN/
```

Code is formatted with [Black](https://github.com/psf/black) using a pre-commit hook. To configure it, run:

```bash
$ pre-commit install
```

## __Repository Structure__

Intention is to mirror most of the structure of PyTorch framework. Several core functions are defined in the functional module. 

## __Network Definition__

The overall structure of a network definition is the same as in PyTorch where possible. All newly defined object inherit from the nn.Module class. The biggest differences are as follows:

- Each layer consists out of a Connection and a Neuron object because they both implement different time based dynamics.
- Training does not use gradients.
- Neurons have a state that persists between consecutive timesteps.
- Networks inherit from a special pysnn.SNNNetwork class.

## __Neurons__

This object is the main difference with ANNs. Neurons have highly non-linear (and also non-differentiable) behaviour. They have an internal voltage, once that surpasses a threshold value it generates a binary spike (non-differentiable operation) which is then propagated to the following layer of Neurons through a Connection object. Defining a new Neuron class is rather simple, one only has to define new neuronal dynamics functions for the Neuron's voltage and trace. The supporting functions are (almost) all defined in the Neuron base class.

For an introduction to (biological) neuronal dynamics, and spiking neural networks the reader is referred to [Neuronal Dynamics](https://neuronaldynamics.epfl.ch/online/index.html) by Wulfram Gerstner, Werner M. Kistler, Richard Naud and Liam Paninski.

## __Connections__

It contains connection weights and routes signals between different layers. It only really differs with PyTorch layers in the fact that it has a state between iterations of its past activity, and the possibility of delaying signal transmission between layers.

### __Connection Shapes__

In order to keep track of traces and delays in information passing tensors an extra dimension is needed compared to the PyTorch conventions. 
Due to the addition of spike traces, each spiking tensor contains an extra trace dimension as the last dimension. The resulting dimension ordering is as follows for an image tensor (trace is indicated as R to not be confused with time for video data):

    [batch size, channels, height, width, traces] (B,C,H,W,R)

For fully connected layers the resulting tensor is as follows (free dimension can be used the same as in PyTorch):

    [batch size, free dimension, input elements, traces] (B,F,I,R)

Currently, no explicit 3D convolution is possible like is common within video-processing. Luckily, SNNs have a built-in temporal dimension and are (currently still theoretically) well suited for processing videos event by event, and thus not needing 3D convolution.

## __Traces__

Traces are stored both in the Neuron and Connection objects. Currently, Connection objects takes traces from their pre-synaptic Neurons and propagate the trace over time, meaning it does not do any further processing on the traces. If it is desired, one can implement separate trace processing in a custom Connection object.

Traces are stored in a tensor in each Connection, as well as the delay for each trace propagating through the Connection. Only one trace (or signal) can tracked through each synapse. In case delay times through a synapse become very long (longer than the refractory period of the pre-synaptic cell) it is possible for a new signal to enter the Connection before the previous one has travelled through it. In the current implementation the old signal will be overwritten, meaning the information is lost before it can be used!

    It is up to the user to assure refractory periods are just as long or longer than the synaptic delay in the following Connection!

## __Module definitions__

Make sure each module has a self.reset_state() method! It is called from the SNNNetwork class and is needed for proper simulation of multiple
inputs.

## __To do__

- Determine performance of the functions in pysnn.functional, they return the difference and using inplace operations in the Module that is
  calling the functional might provide better performance.
- Allow for having a local copy of a cell's entire trace history. Possibly also extending this to Connection objects. This will result in a large increase in memory usage.
- Change from using .uint8 to .bool datatypes with the introduction of PyTorch 1.2.

### __Learning rules__

- Adjust learning rule such that it is able to select which weights are learnable and which are not. 
- Adjust layer class such that the parameter __training__ is also used within a learning rule. Just make sure gradients are always turned off since those are not needed...
- Add support for convolutional Connections.

### __Connection classes__

- For connection class, make sure it can handle the transmission of multiple spike within the same synapse?

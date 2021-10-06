# OSIPI-DCE-DSC-Toolbox design

## General design ideas

The overall toolbox will be designed with the following elements:

1. The low-level library: implementing concepts in the official OSIPI DCE/DSC lexicon

2. Methods: these will be standalone python functions that utilise the low-level library to provide standard DCE/DSC functionality.

3. Pipelines: these combine methods into complete analyses, from loading data through to saving results

4. Tools: these provide a user-interface to pipelines, *eg* from the command-line or in a GUI app

These elements are discussed in more detail below.

### 1) Low-level library

This will be object-oriented (OO) python, with lexicon concepts mapped to classes, and functionality mapped to class methods. General users do not need to know about the details of the library (though it will of course be well documented!), but contributors will need to understand the structure and conventions we've used.

OO inheritance will be used to provide standard interfaces for abstract concepts. 

Example 1: We will probably have an abstract DCE-MRI tracer-kinetic model class (*eg* `DCE_TK_Model`) that prescribes methods for setting model parameters (and their bounds), computing modelled C(t), checking fitted parameter validity *etc*. Specific sub-classes of `DCE_TK_Model` then implement (*ie* define) their own instance of these methods. So we may have a class `DCE_TK_Model_ETM` that implements the extended-Tofts model.

Example 2: We probably want a `DCE_TimeSeries` (or `DCE_Voxel`? MB: I prefer Timeseries as more generic, *eg* applies to simulated Monte-Carlo data, but need to check what lexicon describes) class that holds a signal time-series (S(t)) and has a method for converting between signal and contrast-agent concentration (C(t)). In most DCE-MRI applications we will use a concrete sub-class `DCE_TimeSeries_SPGR` (indeed this may be all we implement - is this true? SPGR is all I've come across but my DCE-MRI knowledge isn't very widespread), but this leaves open the possibility for defining other more complex signal protocols.

Note multiple inheritance is allowed. So for example, we may have abstract classes *above* those discussed above to group classes into logical families, or further sub-classes *below* those discussed that further specialise individual concepts. In this way, the class inheritance structure of the library matches the logical structure of the lexicon.

For a good example of this type of design in a very commonly used, very well documented library, see [Qt's class structure](https://doc.qt.io/qt-5.15/classes.html). For example, the class `QDoubleSpinBox` (which implements user input through a spin-box taking decimal values) inherits from `QAbstractSpinBox` (as does `QSpinBox` used for integer only input), which in turn inherits from `QWidget` (the family that includes all other user-interface items like sliders and text input *etc*), which in turn inherits from `QObject`.

Also note, although we will need to write the structure (or skeleton) of this library largely scratch, the actual code (*eg* flesh) of a lot the class methods can be copied from the code collection repository (hopefully!).

### 2) OSIPI methods

The methods provide a mid-level between the low-level library and full data analysis pipelines. They allow you to do *eg*

> from osipi.dce.methods import fit_T1

They can be used by
+ OSIPI developers in the pipelines we provide (see below)
+ OSIPI users to write their own pipelines
+ OSIPI users for experimental/interactive analysis (eg Monte-Carlo simulations)

The main data inputs should duck-type standard numpy array inputs as well as our library classes, so that OSIPI users don't necessarily need detailed knowledge of the library to run the methods.

Flexibility/modularity is provided by using the inheritance structure of the underlying library and the [template method](https://en.wikipedia.org/wiki/Template_method_pattern) together with user options specifying what "flavour" they want to run.

Sensible candidates for methods might be:

+ Fitting baseline T1

+ Converting between contrast-agent concentration and signal time-series

+ Estimating bolus arrival time

+ Fitting a tracer-kinetic model to signal/concentration time-series

Example: Consider a method for converting signal to concentration. This might have a definition like:

> def signal_to_concentration(S_t, T_1, M_0, method='SPGR', **kwargs)

Here the signal time-series S_t can be input as a generic (N_samples x N_timepoints) numpy array, or as an object of our `DCE_TimeSeries` class (or sub-classes).

With the default method set to 'SPGR', a basic usage might be (using kwargs)

> C_t = signal_to_concentration(S_t, T_1, M_0, FA=FA, TR=TR, r1=r1)

In which case we use the input to construct a structure of `DCE_TimeSeries_SPGR` objects, on which we call their signal to concentration method, and then return C(t) from the main method. To the user, this looks like a standard method definition we have collected in the code collection repository.

But in our design, this is very easily extendable to other acquisition protocols. Imagine we have a factory that takes as input the method string (default 'SPGR') and returns the correct subclass of `DCE_TimeSeries`. Then rather than implementing the signal conversion directly, our method
1. Calls the Time-Series factory to return us the correct sub-class of `DCE_TimeSeries` objects, initialised with the user supplied inputs
2. Calls the objects own signal-to-concentration method

In this design, nothing needs to change in the main `signal_to_concentration` method when we add a new TimeSeries class. All we need to do is update our factory to return the relevant class (note any additional parameters it needs are covered by **kwargs, one of the lovely things about python's function argument design).

As discussed in the OO library section, the actual signal to concentration code (for SPGR) can be copied from the methods in the code collection repository - except rather than being copied directly into the high-level method, it is copied into the class method of `DCE_TimeSeries_SPGR`.

The same principle applies to *eg*, different methods of fitting baseline T1 (VFA, IR *etc*), or fitting different tracer-kinetic models. Note this is exactly how Madym provides flexibility for T1 mapping and tracer-kinetic model fitting. Madym doesn't use this approach for signal to concentration conversion because MB wasn't aware anything more complicated that SPGR existed for DCE-MRI!

### 3) OSIPI pipelines

Pipelines fit together methods into complete analyses, and also manage data I/O to read in input images and output parameter maps, summary statistics etc.

In this way they differ from methods, which work with data structures already existing in python memory, and return variables to memory (although the methods may optionally output logging info).

They are managed by a set of configuration options that define the exact analysis that will be run, thus a single pipeline can be used in many different ways. Suitable candidates for pipelines may include:

- T1 mapping

- AIF detection

- Bolus-arrival time

- Tracer-kinetic model fitting (which may comprise the above pipelines)


### 4) OSIPI tools

Tools sit at the level above pipelines and provide an interface to the user to set the options required by a pipeline. The reason to separate tools from pipelines is that it may be useful to have different interfaces call the same pipeline. For example, we may have separate command line tools for each pipeline, and then a single GUI app from which all pipelines can be run. Having the pipelines defined separately to the tool that calls them thus:

1. Allows code re-use between tools
2. Ensures consistency between the tools

Finally, we have discussed the idea of creating a GUI tool that enables pipeline creation through drag-N-drop of component modules. This is optional, so won't be discussed further in current plans, but worth flagging for future reference.


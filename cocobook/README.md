# CoCoMR Documentation Book

This document provides examples and documentation for CoCo practitioners following a by example approach.

The original document is here: [http://coco.readthedocs.io/en/latest/](http://coco.readthedocs.io/en/latest/)

The publication presenting CoCo is the following:

Ruffaldi E. & Brizzi F. \(2016\). **CoCo - A framework for multicore visuo-haptics in mixed reality**. In SALENTO AVR, 3rd International Conference on Augmented Reality, Virtual Reality and Computer Graphics \(pp. 339-357\). Springer. [doi:10.1007/978-3-319-40621-3\_24](http://dx.doi.org/10.1007/978-3-319-40621-3_24) isbn:978-3-319-40620-6

## Concepts

CoCo is based on the idea of **Composability** that is the possibility of interfacing components in different means depending on their needs, with the possibility of replacing given component with another that provides the same interface. This allows to replace a camera with another, or introduce debugging interface, or switch from live processing to offline processing.

CoCo Components are interfaced via Data-flow and Remote Procedures: the former allows the exchange of data between components with different rates and strategies, while the latter execute direct operations between components. The CoCo design is not excluding a Publisher-Subscriber approach, although at the moment it has not been implemented.

The other CoCo concept is the flexibility in the **Execution**  that is in the way a given Component is mapped to operating system Threads. The association of a CoCo Component to its Execution is decided when the component is first running.

## Execution patterns

CoCo is interested in several application scenarios in which there is a common aspect of soft real-time execution, high-computing requirements, possibly large data exchanged between components. In this scenario it is possible to identify several patterns of execution. 

First the simplest ones:

* Sink
* Filter
* Source

These simple cases can be aggregated in a general Computational Graph in which two patterns are common:

* Pipeline
* Farm

Starting from all these cases we can describe the type of execution partitioning that is provided by CoCo:






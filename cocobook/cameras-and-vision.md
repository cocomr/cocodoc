# Cameras and Vision

CoCo provides the interface to various types of Cameras and a series of internal vision based processing that are employed for the tracking and visualization of entities as in Augmented and Mixed Reality Scenarios.

Refer also to the Online Documentation: [http://coco.readthedocs.io/en/latest/computervision/](http://coco.readthedocs.io/en/latest/computervision/)

The Components in this Library are built around the exchange of video frames with the following features:

* Single Images
* Twin Images: e.g. stereo or RGB-D
* Reference frame: used for the transformations
* Camera Info: used for the calibration

The Images exchanged can be represented via Pixel Formats that comprise the most common without the need to being exhaustive. Pixel Format variability is relevant for reducing the data conversion from sensors, from standard compression libraries and toward given processing types that are interested mainly in some channels of the image.

## Pixel Formats

## RGB Camera Sources




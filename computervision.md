
# Components

CoCo provides some components for processing computer vision in 2D and 3D. These components have in common a data structure used for

# Data

* CameraBuffer
* CameraBuffer::Size
* CameraBuffer::Format
* RGBDBuffer
* CameraInfo: contains intrinsics of a 2D camera

Then for the AR part we have the following special: 
* MarkersData

# Source Components
## ImageSourceTask
This used OpenCV VideoCapture to produce a new RGB image
## KinectReaderTask
## Kinect2ReaderTask
## StreamingReceiverTask
This component produces an image received over the network via ZeroMQ
## CameraSharedMemoryTask
This is a special component that produces 

# Sink Components
## StreamingServerTask
This component encodes the image (RGB or RGBD) and sends it to the network via ZeroMQ using (address,port).

## GLImage (graphics)
This component renders the image in the associated OpenGL context

# Filtering Components

# MarkerTrackerTask
This is a ARUCO marker task that receives images and finds markers. Can take as input both formats. Requires the camera intrinsics to work.
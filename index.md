Compact Component Library: CoCo
===========================

A C++ framework for high-performance multi-thread shared memory applications.
Split your application in component that can run in parallel to each other or be encapsulated one in the other.
Components can run periodically or be triggered by events. 
CoCo provides also a ports system allowing the components to communicate with very low overhead without worrying of concurrency issues.

This framework has been developed in the Percro Laboratory of the Scuola Superiore Sant'Anna, Pisa by Filippo Brizzi and Emanuele Ruffaldi.

It has been widely used in robotics applications, ranging from Augmented Reality for teleoperation to haptic-rendering for virtual training. 

It is also compatible with ROS allowing it to be used in any kind of high-level control applications.

##Modules
[Computer Vision]

Why and What
------------

Orocos inspired, but:

* Lightweight library fully based on C++11.
* Timing oriented:
	* Real-time focused
	* Latency oriented
	* Low overhead communications
* ROS compliant

-> Simpler and Powerful

Folder Structure
---------------
* core: contains the core library features
* util: contains some utilities that are independent from the core files and can also be used standalone. These features are:
    * logging
    * profiling
* launcher: contains the executable that allows to launch a CoCo application
* ros_launcher: same as launcher but as ROS node
* extern: the xml parser library
* samples: contains two sample components togheter with the configuration file to launch them


Installation
-----------

CoCo comes with a cmake file.

`$ mkdir build && cd build
$ cmake .. -DCMAKE_INSTALL_PREFIX=prefix/to/install/directory
$ make -j
$ sudo make install`

To compile the doxygen documentation run
`$ make doc`
The html documentation will be available in `coco/doc/html/index.html`

The last command will install the binary `coco_launcher` that is the launcher of every CoCo application and `libcoco.so`.

To compile your application with CoCo add to your CMakeLists.txt file

`find_package(coco REQUIRED)
include_directories(${coco_INCLUDE_DIRS})
target_link_libraries(... ${coco_LIBS} ...)`

Usage
--------
A CoCo application is composed of two parts: C++ components and an XML configuration file.
The configuration file allows to specify at run-time how the components are connected to each other, their parallelism properties and eventual parameters.

### C++ Components ###

There are two types of CoCo nodes: TaskContext and PeerTask.

* TaskContex: is the main component
* PeerTask: has the same semantics of the TaskContex but cannot be stand alone and has to be encapsulated at run-time by a TaskContext to be executed

samples/component_1.cpp and component_2.cpp contain examples of how to implement a CoCo component or peer.

To create a TaskContext your class has to inherit from `coco::TaskContext` and you have to register your class with the macro `COCO_REGISTER(ClassName)`.

The class has to override the following method:

* `init() : void` called in the main thread before moving the component in its thread
* `onConfig() : void` first function called after been moved in the new thread
* `onUpdate() : void` main function called at every loop iteration

Optionally it is possible to override the `stop()` function that will be called when the application is terminated.

```cpp
class MyComponent : public coco::TaksContext
{
public:
	virtual void init()
	{}
	virtual void onConfig()
	{}
	virtual void onUpdate()
	{}
	virtual void stop()
	{}
};
COCO_REGISTER(MyCompoent)
```

Each component can instantiate the following objects:

* Attribute: allows to specify a class variable as an attribute, this allows to set its value from the configuration file
	* `coco::Attribute<T>(coco::TaskContext *owner, std::string name, T & var)`
		* owner = this
		* name  = attribute name (unique in the same class) used in the configuration file
		* var   = class variable that we want to attach to the attribute 
* Port: input or output it is used to communicate with the other components
	* `coco::InputPort<T>(coco::TaskContext *owner, std::string name, bool is_event = false)`
		* owner    = this
		* name     = port name (unique in the same class) used in the configuration file
		* is_event = specify wether the input port is an event port, this implies that when data is received the component is triggered
	* `coco::OutputPort<T>(coco::TaskContext *owner, std::string name)`
		* owner = this
		* name  = attribute name (unique in the same class) used in the configuration file
* Operation: allows to specify a function as an operation, this allows others to call this function (this is particulary usefull for peer component as we will see later)
	* `coco::Operation<Function::Signature>(coco::TaskContext *owner, std::string name, Function func, Class obj)`
		* owner = this
		* name  = name of the operation, not mandatory to be equal to the name of the function
		* func  = the real function
		* obj   = the object containing the function
	* How to use operation:
		* Enqueue it in the owner task pending operations list. Pending operations are executed before the `onUpdate()` function every time a component is awaken. This is available only for non peer component.
			`coco::TaskContext *task = COCO_TASK("TaskName")`
			`task->enqueueOperation<void(int)>("operation_name", parameters ...);`
			`// Enqueue the operation and associate a function to be called with the return value`
			`task->enqueueOperation<void(int)>(return_fx, "operation_name", parameters ...)`
			For more information about Operation see the [Member Functions](#member-functions) section
		* Call it directly:
			`coco::TaskContext *task = COCO_TASK("TaskName")`
			`task->operation<void(int)>("operation_name")(parameter);`
			
In addition to the base TaskContex it is possible to create a PeerTask.
A `coco::PeerTask` is a `coco::TaskContex` which cannot run alone but must be embedded at run-time inside a Task.

To create a peer just inherite from `coco::PeerTask` and register the peer `COCO_REGISTER(ClassName)`.

Your peer class has to to ovveride only `onConfig()`. The purpose of a PeerTask is to provide operations to whatever component needs them, allowing to increase encapsulation and code reusability.

See the examples in the folder *samples* for more informations.

#### Usefull Macro ####
`COCO_REGISTER("TaskName")`:
as seen above it is used to register either a TaskContext of a PeerTask class.

`COCO_TASK("TaskName")`: return a pointer to the the TaskContext object with name "TaskName" if it exists,  *nullptr* otherwise.

`COCO_NUM_TASK`: return the number of tasks (not peers) instantiated.

`COCO_CONFIGURATION_COMPLETED`: return whether all the components have completed the `onConfig` function.

`COCO_FIND_RESOURCE("what")`: look into the know paths for the resource *what*, and if it founds it return the complete path of the resource, otherwise an empty string. The know paths will be explained later.

`COCO_TERMINATE`: safety terminate the application calling `stop()` function for all the components.

#### <a name="member-functions"></a>Member Functions ####
Of class `coco::TaskContext`

* `std::string instantiationName()`
    * return the name given in the XML configuration file. If no name was given return the class name
* `template <class Sig, class ...Args>
	bool enqueueOperation(const std::string & name, Args... args)`, 
    * *name*: the name of the operation
    * *args*: the list of the arguments to be passed to the operations when called
    * Example of usage from *component_1.cpp*
* `template <class Sig, class ...Args>
	bool enqueueOperation(std::function<void(typename std::function<Sig>::result_type)> return_fx, std::string & name, Args... args)`
	* *return_fx*: the function that is called passing to it the return value of the operation
	* *name*: the name of the operation
	* *args*: the list of the arguments to be passed to the operations when called
* `TaskState 	state()`
	* return the current state of the task, the possible states are:
		* INIT = 0, the task is in initialization phase.
		* PRE_OPERATIONAL = 1,  the task is running the enqueued operations.
		* RUNNING = 2, the task is running the `onUpdate` function.
		* IDLE = 3, the task is idle, either waiting on a timeout to expire or on a trigger.
		* STOPPED = 4, the task has been stopped and it is waiting to terminate.
```cpp
coco::TaskContext *task = COCO_TASK("EzTask2") 
if (task)
	// Enqueue on task "EzTask2" the operation hello()
	// This works only if EzTask2 add hello as operation
	task->enqueueOperation<void(int)>("hello", 42);     
```
	
* `template <class Sig>
	std::function<Sig> operation(const std::string & name)`
    * return an handler for the function associated with the operation "name"

Of class `coco::PortBase`

* `bool isConnected()`
    * return true if the port is connected to at least another port
* `int connectionsCount()`
    * return the number of ports connected to this one

Of class `coco::InputPort`

* `FlowStatus read(T & output)`
    * store in *output* the value in the port
        * *T* is the type of the port
    * return a FlowStatus object containing the result status of the read
        * `enum FlowStatus { NO_DATA, OLD_DATA, NEW_DATA };`
    * in case the port is attached to multiple other ports it uses a round-robin schedule to go through all of them
* `FlowStatus readAll(std::vector<T> & output)`
    * same as above but in this case read from all the connections and store the result in the vector, no way to know from which connection comes which data
    * return a FlowStatus object containing the result status of the read

of class `coco::OutputPort`

* `void write(T & input)`
    * write *input* in all the connected ports
* `void write(T &input, const std::string &name)`
    * write *input* in the port with name *name*


### Launcher ###
Once you have created your componets and you have compiled them in shared libraries you have to specify to CoCo how this components have to be run.

To do so you have to create an xml file with the following specifications:

* Outher tag:

```xml
<package>
</package>
```

* `logconfig`: (more information on how to use the log utilities are provided in the [Logging](#logging) section)

```xml
<logconfig>
    <levels>0 1 2 3 4</levels> <!-- logging enabled levels -->
    <outfile>log2.txt</outfile> <!--file where to write the logging, if empty or missing no log file produced -->
    <types>debug err log</types> <!-- enabled types -->
</logconfig>
```

* `resourcepaths`: allows to specify the path where to find the component libraries and all the resources file passed to the attributes. The paths can be either absolute or relative; to improve code portability CoCo support the use of the environment variable `COCO_PREFIX_PATH`.  All the path in `resourcepaths` will be concatenated with the paths in the environment variable and will be used to retrieve all the resources needed by the applications. `COCO_PREFIX_PATH` supports multiple paths by simply concatenating them whith a semicolumn (:).
`COCO_FIND_RESOURCE("what")` uses these paths to retrieve the resource passed in input.

```xml
<resourcespaths>
	<librariespath>path/to/component/libraries</librariespath>
	<librariespath>path/to/more/component/libraries</librariespath>
	<path>path/to/resources</path>
	<path>path/to/more/resources</path>
</resourcespaths>
```
* `components`: list of *component*

```xml
<component>
	<task>task_name</task>
	<name>name_for_execution</name> 
	<library>libray_name</library>
	<attributes>
		<attribute name="attr1" value="2" /> 
		<attribute name="attr2" value="hello!" />
		<attribute name="resource_file" value="data.txt" type="file" /> 
	</attributes>
	<components> <!-- peers --> 
		<component>
			... 
		</component>
		...
	</components>
</component>
```
* `component`: represent a TaskContext specification
    * `task`: the name of the TaskContex or PeerTask class
    * `name`: Allows to give another identifier to this instantiation of the task. This name is mandatory only if we want to instantiate multiple components of the same type and must be unique
    * `libray_name`: Name of the library without prefix and suffix
    * `attributes`: list of attribues
        * `attribute`
            * *name*: the name of the attribute as specified in the class definition
            * *value*: the value, all the base C++ type are supported plus string
            * *type*: the supported types are:
                * *file*: when the value of the attribute is a resource name. In this way you specify to the launcher to look at the path specified above to find the resource and concat the whole path in the string before assign it to the attribute; this allow the configuration file to be more independent from the running pc.
    * `components`: List of PeerTask attached to this components. A component can have all the peers it wants. Also peers can have their own PeerTask going deeper as wanted.



* `connections`: specifies the connections between components ports.

```xml
<connections>
	<connection>
		...
	</connection>
	<connection>
		...
	</connection>
</connections>
```

* `connection`:

```xml
<connection data="BUFFER" policy="LOCKED" transport="LOCAL" buffersize="10">
	<src task="NameForExecution1" port="port_name_out"/>
	<dest task="NameForExecution2" port="port_name_in"/>
</connection>	
```
* *data*: type of port buffer
	* DATA: buffer lenght 1
	* BUFFER: FIFO buffer of lenght "buffersize"
	* CIRCULAR: circula FIFO buffer of lenght "buffersize"
* *policy*: policy of the locking system to be used
	* LOCKED: guarantee mutually exclusive access
	* UNSYNC: no synchronization applied
* *transport*: 
	* LOCAL: shared memory for thread
	* IPC: communication between processes (not yet implemented)


* `activities` contains the collections of activity.

```xml
<activities>
	<activity>
		...
	</activity>
	...
	<activity>
		...
	</activity>
</activities>
```
* `activity`: each *activity* is associated with a thread and can contains multiple *components*. For each activity we specify how it has to be executed and the *components* to execute

```xml
<activity>
	<schedulepolicy activity="parallel" type="periodic" value="1000" />
	<components>
		<component>task_name</component>
		<component>task_name</component>
		...			
	</components>
</activity>
```

* `schedulepolicy` has several attributes
    * `activity`
	    * *parallel*: executed in a new dedicated thread
		* *sequential*: executed in the main process thread. No more than one sequential activity is allowed per application
    * `type`
	    * *periodic*: the activity run periodically with period "value" expressed in millisecond
		* *triggered*: the activity run only when triggered by receiving data in an event port of one of its component
            * NOTE: if a triggered activity contains more than one component when it is triggered it will execute all the component inside, no matter for which component the triggered was ment.	
        * *value*: the period in millisecond.
            * It is ignored if *type* if *triggered*
		* *affinity*: id of the core where to pin the thread execution. Be sure that id < #cores 
		* *exclusive_affinity*: id of the core where to exclusive pin the thread execution. All other CoCo thread will be excluded to execute on that core.
          


	
* `includes`: allows to include others XML configuration files with the same structures and load the components and connections specified in the file. For each specified include file all the resource paths components and connections will be loaded. Any activity specifications or log configuration will be ignored. It is possible to specify connections between components described in different files, as well as associate them in activities. This is because all the actual loading is done after all the XML files have been parsed. Of course an included XML file can includes other xml files.
```xml
<includes>
	<include>path/to/xml/file</include>
	<include>path/to/xml/file</include>
	...
</include>
```
		
### Utils ###
####<a name="logging"></a> Logging ####
CoCo provides logging features.

* `COCO_LOG(level) << "this is a log " << i << ...`
	Base log utility. Will be displayed only if *level* is specified in in the configuration.
* `COCO_DEBUG("name") << "this a debug message " << i << ...`
	Used for debuggin messages. A *name* can be associate with it and will be displayed. `COCO_DEBUG` is disabled when the component is build in Release mode.
* `COCO_ERR() << "this is a not serious error " << error << ...`
	Used to raise an error that will not have the application to terminate.
* `COCO_FATAL() << "Fatal error, the application will be shut down " << error << ...`
 Print the message and immediately terminate the application.
* `COCO_SAMPLE(level, i) << "Print this msg every " << i << " calls " << ...`
	As `COCO_LOG`, but the message is printed only every *i* iterations. This is useful when the logging call is inside a high frequency loop.

#### Timing #####
CoCo provide a set of utilities to measure time.

* `COCO_START_TIMER("my_timer")`
	Starts a timer named *name*.
* `COCO_STOP_TIMER("my_timer")`
	Stops the timer named *my_timer*. This however will not reset the timer *my_timer*.
* `COCO_CLEAR_TIMER("my_timer")`
	Resets the timer named *my_timer*.
* `COCO_TIME_COUNT("my_timer")`
	Returns the number of intervals measured for timer *my_timer*.
* `COCO_TIME("my_timer")`
	Returns the total measured time for the timer *my_timer*.
* `COCO_TIME_MEAN("my_timer")`
  Returns the mean execution time of each iteration of for the timer *my_timer*.
* `COCO_TIME_VARIANCE("my_timer")`
  Returns the variance of the execution time of each iteration of for the timer *my_timer*.
* `COCO_SERVICE_TIME("my_timer")`
  Returns the mean service time of the timer, i.e. returns the mean time between to consecutive starts of the timer *my_timer*.
* `COCO_SERVICE_TIME_VARIANCE("my_timer")`
  Returns the variance of the service time of the timer *my_timer*.


###CoCo Launch###
To run a CoCo application it is possible to use either the `coco_launcher` executable or the  `coco_ros_launcher_node` ROS node. Both the executables accepts several inputs:

* -x [ --config_file ] arg
	Xml file with the configurations of the application.
* -p [ --profiling ] [=arg(=5)] 
	Enable the collection of statistics of the execution of each component. The information are printed every 5 seconds, to change this value pass a new time to the option.
* -g [ --graph ] arg          
	Create the graph of the various components and of their connections.
 


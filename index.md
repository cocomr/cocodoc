Compact Component Library: CoCo
===========================

A C++ framework for high-performance multi-thread shared memory applications.
CoCo allows to split your application in components and decide at run-time, through a configuration file, which one to instantiate, on how many thread and how they communicate with each other to exchange data.
The framework is mainly focused on bringing the paradigm of high-performance computing to soft real-time application allowing to exploit the power of multi-core architectures.

This framework has been developed in the Percro Laboratory of the Scuola Superiore Sant'Anna, Pisa by Filippo Brizzi and Emanuele Ruffaldi.

It has been widely used in robotics applications, ranging from augmented reality for teleoperation to haptic rendering for virtual training. 

It is also compatible with ROS allowing it to be used in any kind of high-level control applications.

##Modules
[Computer Vision]

Why and What
------------
CoCo desing has been inspired by the Orocos framework, but with a different aims. While the Orocos framework targets (hard) real-time application as low level robotic control, thus requiring special Linux kernel to be executed, CoCo aims at providing a framework for high level soft real-time application as robotic vision, haptic rendering and sensors integration.

The main characteristic of CoCo are:

* Lightweight library fully based on C++11.
* Possibility to configure at run-time:
	* Components and their attributes
	* Connections between components
	* Number of thread to be instantied
	* Mapping between threads and components
* Timing oriented:
	* Support for Linux real-time schedulers
	* Zero overhead communications via shared pointer and smart memory allocation
	* Embedded timing profiler
		* Automatic service time and execution time calculation for every component
		* Automatic latency calculation between two component
* ROS compliant
* High-performance computing features, configurable at run-time:
	* Pipeline 
	* Farm, number of workers specified at run-time
	* Farm of pipeline
* Web based visualization tool to visualize components connections, timing statistics via tables and graphics



Simple and Powerful!!

Folder Structure
---------------
* core: contains the core library features and the utilities
    * util: contains some utilities that are independent from the core files and can also be used standalone. These features are:
        * logging
        * profiling
        * memory
    * web_server
* launcher: contains the executable that allows to launch a CoCo application
* ros_launcher: same as launcher but as ROS node
* extern: the xml and json parser library and the moongose library to create the web server
* samples: contains some sample components together with the configuration files to launch them


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
For a complete class reference documentation look at the doxygen documentation contained in the library doc folder. To compile it just type `make doc`.

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
* `unsigned connectionsCount()`
    * return the number of ports connected to this one
* `unsigned queueLength()`
	* return the number of data blocks in the port. If the port has several connections, the length is equal to the sum of all the data in all the connections

Of class `coco::InputPort`

* `FlowStatus hasNewData()`
	* return whether the port contains new data
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
Once you have created your componets and you have compiled them in shared libraries you have to specify to CoCo how these components have to be run. The run-time choices to be made include: setting the values of the attributes, associating to task eventual peers, setting connections and allocating tasks over threads.

To do so you have to create an xml file with the following specifications:

* Outher tag:

```xml
<package>
</package>
```

* `log`: (more information on how to use the log utilities are provided in the [Logging](#logging) section)

```xml
<log>
    <levels>0 1 2 3 4</levels> <!-- logging enabled levels -->
    <outfile>log2.txt</outfile> <!--file where to write the logging, if empty or missing no log file produced -->
    <types>debug err log</types> <!-- enabled types -->
</log>
```

* `paths`: allows to specify the path where to find the component libraries and all the resources file passed to the attributes. The paths can be either absolute or relative; to improve code portability CoCo support the use of the environment variable `COCO_PREFIX_PATH`.  All the path in `paths` will be concatenated with the paths in the environment variable and will be used to retrieve all the resources needed by the applications. `COCO_PREFIX_PATH` supports multiple paths by simply concatenating them with a semi-column (:).
`COCO_FIND_RESOURCE("what")` uses these paths to retrieve the resource passed in input.

```xml
<paths>
	<path>path/to/component/libraries</librariespath>
	<path>path/to/more/component/libraries</librariespath>
	<path>path/to/resources</path>
	<path>path/to/more/resources</path>
</paths>
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
                * *file*: when the value of the attribute is a resource name. In this way you specify to the launcher to look at the path specified above to find the resource and concat the whole path in the string before assign it to the attribute; this allow the configuration file to be more independent from the 
           * Every component has a list of standard attribute already associated with them:
	           * *wait_all_trigger*: In case of a component with multiple *event* ports setting this attribute to 1 tells the run-time to trigger the execution of the component only when data is present in all the ports. In caso of a non triggered component or a peer this attribute has no effect
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



```xml
<connection data="BUFFER" policy="LOCKED" transport="LOCAL" buffersize="10">
	<src task="NameForExecution1" port="port_name_out"/>
	<dest task="NameForExecution2" port="port_name_in"/>
</connection>	
```
* `connection`:
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


```xml
<activity>
	<schedule activity="parallel" type="periodic" value="1000" />
	<components>
		<component name="task_name" />
		<component name="another_task_name" />
		...			
	</components>
</activity>
```
* `activity`: each *activity* is associated with a thread and can contains multiple *components*. For each activity we specify how it has to be executed and the *components* to execute
	* `schedule` has several attributes
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
          
```xml
<pipeline>
	<schedule activity="parallel/sequential"/>
	<components>
		<component name="task_name1" out="out_port"/>
		<component name="task_name2" in="in_port" out="out_port"/>
		<component name="task_name3" in="in_port"/>		
		...			
	</components>
</activity>
```
* `pipeline`: this tag allows to specify pipelines of tasks. It doesn't add semantic to the execution as it is possible to create a pipeline using *activity* and the proper connections. The use of this tag allows to be more clear and to avoid errors, such as forgetting connections or not setting an activity as triggered.
	* `schedule`: allows to specify whether the components will be parallel or executed sequential.
		* `activity`
			* `parallel`:  each component will run on a separate thread
			* `sequential`: all the components will run on the same thread, can be useful when the tasks have a computation time much lower than the service time

```xml
<farm>
    <schedule workers="3" />
    <source>
        <schedule type="periodic" value="500" exclusive_affinity="3" />
        <component name="Task1" out="value_OUT" />
    </source>
    <pipeline>
        <schedule activity="sequential" />
        <components>
            <component name="Task2" in="value_IN" out="value_OUT" />
            <component name="Task3" in="value_IN" out="value_OUT" />
            <component name="Task4" in="value_IN" out="value_OUT" />
        </components>
    </pipeline>
    <gather>
        <component name="Task5" in="value_IN" />
    </gather>
</farm>	
```
* `farm`: this tag allows the automatic creation of a farm of tasks. The farm  is a paradigm of the High-Performance Computing world that consists in instantiating multiple time the same node in order to be able to distribute the work on several of them. This comes in handy when the service time is much lower than the computation time, allowing to keep the source service time.
	* `schedule`: the number of workers, how many instantiation of the same tasks the farm has to do.
	* `source`: the source component that produce the data to be used by the workers.
		* `schedule` the source component can be periodic (with any period) or triggered and can be pinned on a core via affinity.
		* `out`: the output port that produces the data for the workers.
	* `pipeline`: here we instantiate the pipeline that operates as worker and can be composed by one or many tasks. The syntax is exactly as the standard pipeline presented above.
	* `gather`: it is the component that receives the data from the workers and it's execution is triggered by the data reception.
		* `in`: input port on which to receive data.

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
	Base log utility. Will be displayed only if *level* is specified in in the configuration. Automatically detect the component name executing the log command and record it; if not executed inside Task's method use
	* `COCO_LOG(level, name)`
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
* `COCO_TIME("my_timer")`
	Returns the last measured elapsed time for the timer *my_timer*
* `COCO_TIME_MEAN("my_timer")`
  Returns the mean execution time of each iteration of for the timer *my_timer*.
* `COCO_TIME_STATISTICS("my_timer")`
	  Returns an object *TimeStatistics* that contains all the information about the timer:
	  * Number of iterations
	  * Last measured elapsed time
	  * Mean and variance elapsed time
	  * Mean and variance service time
	  * Min and max elapsed time


###CoCo Launch###
To run a CoCo application it is possible to use either the `coco_launcher` executable or the  `coco_ros_launcher_node` ROS node. Both the executables accepts several inputs:
 
* -h [ --help ]
 Print help message and exit.
* -x [ --config_file ] arg
XML file with the configurations of the application.
* -d [ --disabled ] arg 
List of components that are going to be disabled in the execution
* -p [ --profiling ] [=arg(=5)] 
Enable the collection of statistics of the executions. Use only during debug as it slow down the performances.
* -g [ --graph ] arg
Create the graph of the various components and of their connections.
*  -t [ --xml_template ] arg         
Print the xml template for all the components contained in the library.
*  -w [ --web_server ] [=arg(=7707)]
 Instantiate a web server to the specified port that allows to view statics about the executions.
* -r [ --web_root ] arg
Set document root for web server, not needed if the the library has been installed.
* -l [ --latency ] arg 
Set the two task between which calculate the latency. Peer are not valid.

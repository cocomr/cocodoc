Compact Component Library: CoCo
===========================

A C++ framework for high-performance multi-thread shared memory applications.
Split your application in component that can run in parallel to each other or be encapsulated one in the other.
Components can run periodically or be triggered by events. 
CoCo provides also a port system allowing the components to communicate with very low overhead without worrying of concurrency issues.

This framework has been developed in the Percro Laboratory of the Scuola Superiore Sant'Anna, Pisa by Filippo Brizzi and Emanuele Ruffaldi.

It has been widely used in robotics applications, ranging from Augmented Reality for teleoperation to haptic-rendering for virtual training. 

It is also compatible with ROS allowing it to be used in any kind of high-level control applications.

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
* launcher: contains the executable that allows to launch a coco application
* ros_launcher: same as launcher but as ROS node
* extern: the xml parser library and
* samples: contains two sample component togheter with the configuration file to launch them


Installation
-----------

CoCo comes with a cmake file.
	$ mkdir build && cd build
	$ cmake .. -DCMAKE_INSTALL_PREFIX=prefix/to/install/directory
	$ make -j
	$ sudo make install

The last command will install the binary `coco_launcher` that is the launcher of every coco application and `libcoco.so`.

To compile your application with coco add to your CMakeLists.txt file
find_package(coco REQUIRED)
include_directories(coco_INCLUDE_DIRS)
target_link_libraries(... ${coco_LIBS} ...)

Usage
--------
The usage of the framework is composed of two parts the first is the real C++ code, while the second is the creation of the configuration file where to specify all the properties.

### C++ code ###

There are two types of CoCo nodes: TaskContext and PeerTask.

* TaskContex: is the main component
* PeerTask: has the same semantics of the TaskContex but cannot be stand alone and has to be encapsulated by a TaskContext to be executed

samples/component_1.cpp and component_2.cpp contain examples of how to implement a coco component or peer.

To create a TaskContext your class has to inherit from `coco::TaskContext` and you have to register your class with the macro `COCO_REGISTER(ClassName)`.

The class has to override the following method:

* `info() : std::string` returns a description of the component
* `init() : void` called in the main thread before moving the component in its thread
* `onConfig() : void` first function called after been moved in the new thread
* `onUpdate() : void` main function called at every loop iteration

```cpp
class MyComponent : public coco::TaksContext
{
public:
	virtual std::string info()
	{
		return "MyComponent does ...";
	}
	virtual void init()
	{}
	virtual void onConfig()
	{}
	virtual void onUpdate()
	{}
};
COCO_REGISTER(MyCompoent)
```

Each component can embedd the following objects:

* Attribute: allows to specify a class variable as an attribute, this allows to set its value from the configuration file
	* `coco::Attribute<T>(coco::TaskContext *owner, std::string name, T var)`
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
* Operation: allows to specify a function as an operation, this allows others to call your function (this is particulary usefull for peer component as we will see later)
	* `coco::Operation<Function::Signature>(coco::TaskContext *owner, std::string name, Function func, Class obj)`
		* owner = this
		* name  = name of the operation, not mandatory to be equal to the name of the function
		* func  = the real function
		* obj   = the object containing the function
	* How to use operation:
		* Enqueue it in the owner task pending operation:
			`coco::TaskContext *task = COCO_TASK("TaskName")`
			`task->enqueueOperation<void(int)>("operation_name", parameters ...);`
		* Call it directly:
			`coco::TaskContext *task = COCO_TASK("TaskName")`
			`task->getOperation<void(int)>("operation_name")(parameter);`
			
In addition to the base TaskContex it is possible to create also a PeerTask.
A `coco::PeerTask` is a `coco::TaskContex` which cannot run alone but must be embedded inside a Task.

To create a peer just inherite from `coco::PeerTask` and register the peer `COCO_REGISTER(ClassName)`.

Your peer class has to to ovveride only `info()` and `onConfig()`. The purpose of a PeerTask is to provide operations to whatever component needs it allowing to increase the encapsulation and code reusability.

See the examples in the folder *samples* for more informations.

#### Usefull Macro ####
`COCO_REGISTER("TaskName")`:
as seen above it is used to register either a TaskContext of a PeerTask class.

`COCO_TASK("TaskName")`: return a pointer to the the TaskContext object with name "TaskName" if it exists, *nullptr* otherwise.

#### Member Functions ####
Of class `coco::TaskContext`

* `std::string instantiationName()`
    * return the name given in the XML configuration file. If no name was give return the class name
* `template <class Sig, class ...Args>
	bool enqueueOperation(const std::string & name, Args... args)`, 
    * *name*: the name of the operation
    * *args*: the list of the arguments to be passed to the operations when called
    * Example of usage from *component_1.cpp*
	
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

* `logconfig`: (more information on how to use the log utilities will be provided later)

```xml
<logconfig>
    <levels>0 1 2 3 4</levels> <!-- logging enabled levels -->
    <outfile>log2.txt</outfile> <!--file where to write the logging, if empty or missing no log file produced -->
    <types>debug err log</types> <!-- enabled types -->
</logconfig>
```

* `resourcepaths`: allows to specify the path where to find the compoennt libraries and all the resources file passed to the attributes

```xml
<resourcespaths>
	<librariespath>path/to/component/libraries</librariespath>
	<path>path/to/resources</path>
	<path>path/to/more/resources</path>
</resourcespaths>
```

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
		<component>
			...
		</component>
		...
		<component>
			...
		</component>
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
* `components`: list of *component* that execute inside the same thread sequentially in the order specifed in the XML 
	* `component`: represent a TaskContext specification
	
```xml
<component>
	<task>TaskName</task>
	<name>NameForExecution</name> 
	<library>libray_name</library>
	<attributes>
		<attribute name="attr1" value="2" /> 
		<attribute name="attr2" value="hello!" />
		<attribute name="resource_file" value="data.txt" type="file" /> 
	</attributes>
	<components> 
		<component>
			...
		</component>
	</components>
</component>
```

* `components`
    * `task`: the name of the TaskContex class
    * `name`: Allows to give another identifier to this instantiation of the task. This name is mandatory only if we want to instantiate multiple components of the same type and must be unique
    * `libray_name`: Name of the library without prefix and suffix
    * `attributes`: list of attribues
        * `attribute`
            * *name*: the name of the attribute as specified in the class definition
            * *value*: the value, all the base C++ type are supported plus string
            * *type*: the supported types are:
                * *file*: when the value of the attribute is a resource name. In this way you specify to the launcher to look at the path specified above to find the resource and concat the whole path in the string before assign it to the attribute; this allow the configuration file to be more independent from the running pc.
    * `components`: List of PeerTask attached to this components. A component can have all the peers it wants. Also peers can have their own PeerTask going deeper as wanted
          

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
	* IPC: communication between processes
		
		
### Utils ###
#### Logging ####

#### Timing #####

	
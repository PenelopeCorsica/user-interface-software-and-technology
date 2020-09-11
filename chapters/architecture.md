While the previous chapter discussed many of the seminal interaction paradigms we have invented for interacting with computers, we've discussed little about how all of the widgets, interaction paradigms, and other user interface ideas are actually _implemented_ as software. This knowledge is obviously important for developers who implement buttons, scroll bars, gestures, and so on, but is this knowledge important for anyone else?

I argue yes. First, a precise understanding of user interface implementation allows designers to have a precise understanding of user interface behavior. This helps designers and engineers who think of user interface components as the building blocks of a user interface to analyze limitations of interfaces, predict edge cases in their behavior, and discuss their behavior precisely. Knowing, for example, that a button only invokes its command _after_ the mouse button is released allows one to reason about the assumptions a button makes about _ability_. The ability to hold a mouse button down, for example, isn't something that all people have, whether due to limited finger strength, motor tremors that lead to accidental button releases, or other motor-physical limitations. A user interface designer or engineer knowing these low-level implementation details is like violinist knowing whether a bow's strings are made from synthetic materials or Siberian horse tail. These details allow you to fully control what you make and how it behaves.

Knowledge of user interface implementation might also be important if you want to invent _new_ interface paradigms. A low-granularity of user interface implementation knowledge allows you to see exactly how current interfaces are limited, and empowers you to envision new interfaces that don't have those limitations. For example, when [Apple redesigned their keyboards to have shallower depth|ttps://techcrunch.com/2017/06/07/that-new-keyboard-is-the-key-to-apples-macbook-update/], their design team needed deeper knowledge than just "pushing a key sends a key code to the operating system." They needed to know the physical mechanisms that afford depressing a key, the tactile feedback those mechanisms provide, and the auditory feedback that users rely on to confirm they've pressed a key. Expertise in these physical qualities of the hardware interface of a keyboard was essential to designing a new keyboard experience.

Precise technical knowledge of user interface implementation also allows designers and engineers to have a shared vocabulary to _communicate_ about interfaces. Designers should feel empowered to converse about interface implementation with engineers, knowing enough to critique designs and convey alternatives. Without this vocabulary and a grasp of these concepts, engineers retain power over user interface design, even though they aren't trained to design interfaces.

# Architecture
What is this knowledge of implementation? I'm not proposing that everyone memorize and understand, for example, the source code implementations of all of Windows or macOS's widgets. I'm talking about a level of knowledge at the *architectural* level, which involves understandings patterns of control and data flow that govern user interface behavior.

To illustrate this notion of architecture, let's return to the example of a graphical user interface button. We've all used buttons, but rarely do we think about exactly how they work. Here is the _architecture_ of a very simple button:

|button-architecture.jpg|A diagram of two modes of a simple button: up and down, with a state transition on mouse down and a state transition on mouse up.|State transitions|Amy J. Ko|

This is a simple *state machine* with two states: _mouse up_ (on the left) and _mouse down_ (on the right). _Events_ cause the machine to transition between states: when the button receives a _mouse down_ event, it transitions to the down state.  When it receives a _mouse up_ event, it transitions to the up state and also executes its command.  This is about as simple as buttons get.

Of course, graphical user interface buttons in modern operating systems are actually much more complex. Consider, for example, what happens if the button is in a down state, but the mouse cursor moves outside the boundary of the button and _then_ a mouse up event occurs. When this happens, it transitions to the mouse up state, but does _not_ execute the button's command. (Try this on a touchscreen: tap on a button, slide your finger away from it, then release, and you'll see the button's command isn't executed). If the button can be disabled, then there's an entirely different state of disabled in which the button never changes states in response to events. If the button supports touch input, then a button's state machine also needs to handle touch events. All of these additional behaviors add more complexity to a button's state machine, but ultimately, it is still state machine.

All user interface widgets are implemented in similar ways, defining a set of states and events that cause transitions between those states. Scroll bar handles respond to mouse down, drag, and mouse up events, text boxes respond to keyboard events, links respond to mouse down and up events. Even text in a web browser or text editor response to mouse down, drag, and mouse up events to enter a "text selected state".

A nearly ubiquitous way to implement these state machines is to follow a *model-view-controller* (MVC) architecture. One can think of MVC as a division of responsibilities between storing data, showing data, and manging the interaction between the storage and the display. For example, think about the signs that are often displayed on gas station or movie theaters. Someone is responsible for writing and storing the content that will be shown on the signs. Someone else is responsible for putting the content on the signs. And someone is in charge of managing the communication between the two, such as a manager, who listens  decides when the sign will change. In the same way, MVC architectures in user interfaces take an individual component of an interface (e.g., a button, a form, a progress bar, or some other widget) and divide its implementation into three parts:

|mvc.jpg|An MVC architecture for a web form, showing a model that stores the username and password being entered, the controller that decides when to activate the login button, and the view, which displays the form data and buttons, and listens for clicks.|Model-view-controller architecture|Amy J. Ko|


* The *model* stores the _data_ that a user interface is presenting. For example, in the figure above, the model stores the username and password a user is currently entering. (In many user interface toolits, this wouldn't be stored in a separate database, but in the text field object in memory).
* The *view* visualizes the data in the model. 	For example, in the figure above, the view is the form itself and the controls on it, including the text fields and login button. The view's job is to render the controls, listen for input from the user (e.g., clicking the login button), and display any output the controller decides to provide (e.g., form validation feedback).
* The *controller* implements the logic of the user interface. In our login form example above, that includes validating the username and password (e.g., neither can be empty), and submitting the information when the login button is pressed. The controller gets and stores data when necessary, and tells the view to update itself as the model changes.

Now, if every individual widget in a user interface is its own self-contained model-view-controller architecture, how are all of these individual widgets composed together into a user interface? There are three big ideas that stitch together individual widgets into an entire interface.

First, all user interfaces are structured as *hierarchies* in which one widget can contain zero or more other "child" widgets, and each widget has a parent, except for the "root" widget (usually a window). For instance, here's the Facebook post UI we were discussing earlier and its corresponding hierarchy:
		
|ui-hierarchy.jpg|A diagram mapping the Facebook post user interface to the component hierarchy it is composed of, including a post, an avatar icon, an editor, a text box, a label, and an emoticon and photo upload widget|Component hierarchies|Amy J. Ko|

Notice how there are some components in the tree above that aren't visible in the UI (the "post", the "editor", the "special input" container). Each of these are essentially containers that group components together. This brings us to the second big idea of *layout*, in which the children of each component are organized spatially according to some layout rule. For example, the special input widgets are laid out in a horizontal row within the special inputs container and the special inputs container itself is laid out right aligned in the "editor" container. Each component has its own layout rules that govern the display of its children.

Finally, *event propagation* is the process by which user interface events move from a physical device to a particular user interface component. Each device has its own process, because it has its own semantics. For instance:

* A *mouse* emits mouse move events and button presses and releases. All of these are emitted as discrete events to operating system. Particular sequences events are aggregated into _synthetic_ events like a click (a mouse press followed by a mouse release). When the operating system receives them, it first decides which window will receive those events by comparing the position of the mouse to the position and layering of the windows, finding the topmost window that contains the mouse position. Then, the window decides which component will handle the event by finding the topmost component whose spatial boundaries contain the mouse.That event is then sent to that component. If the component doesn't handle the event (e.g., someone clicks on some text in a web browser that doesn't respond to clicks), the event may be _propagated_ to its parent, and to it's parent's parent, etc, seeing if any of the ancestors in the component hierarchy want to handle the event. Every user interface framework handles this propagation slightly differently, but most follow this basic pattern.
* A *keyboard* emits key down and key up events, each with a _key code_ that corresponds to the physical key that was pressed. 	As with a mouse, sequences are synthesized into other events (e.g., a key down followed by a key up with the same key is a key "press"). Whereas a mouse has a position, a keyboard does not, and so operating systems maintain a notion of _window focus_ to determine which window is receiving key events, and then each window maintains a notion of _keyboard focus_ to determine which component receives key events. 	Operating systems are then responsible for providing a visible indicator of which component has keyboard focus (perhaps giving it a border highlight and showing a blinking text caret). As with mouse events, if the component with focus does not handle a keyboard event, it may be propagated to its ancestors and handled by one of them. 	For example, when you press the escape key when a confirmation dialog is in focus, the button that has focus will ignore it, but the dialog window may interpret the escape key press as a "cancel".
* A *touch screen* emits a stream of touch events, segmented by start, move, and end events. Other events include touch cancel events, such as when you slide your finger off of a screen to indicate you no longer want to touch. This low-level stream of events is converted by operating systems and applications into touch gestures. Most operating systems recognize a class of gestures and emit events for them as well, allowing user interface controls to respond to them.
* Even *speech* interfaces emit events. For example, digital voice assistants are continuously listening for activation commands such as "Hey Siri" or "Alexa." After these are detected, they begin converting speech into text, which is then matched to one or more commands. Applications that expose a set of commands then receive events that trigger the application to process the command. Therefore, the notion of input events isn't inherently tactile; it's more generally about translating low-level inputs into high-level commands.
		
# Advances in User Interface Architecture

While the basic ideas presented above are now ubiquitous in desktop and mobile operating systems, the field of HCI has rapidly innovated beyond these original ideas. For instance, much of the research in the 1990's focused on building more robust, scalable, flexible, and powerful user interface toolkits for building desktop interfaces. The [Amulet toolkit|https://www.cs.cmu.edu/afs/cs/project/amulet/www/amulet-home.html] was one of the most notable of these, offering a unified framework for supporting graphical objects, animation, input, output, commands, and undo<myers97>. At the same time, there was extensive work on constraint systems, which would allow interface developers to declaratively express rules the interface must follow (e.g., this button should always be next to this other button)<bharat95,hudson96>. Other projects sought to make it easier to "skin" the visual appearance of interfaces without having to modify a user interface implementation<hudson97>. Research in the 2000's shifted to deepen these ideas. For example, some work investigated alternatives to component hierarchies such as scene graphs<huot04> and views across multiple machines<lecolinet03>, making it easier to build highly animated and connected interfaces. Some works deepened architectures for supporting undo and redo<edwards00>. Many of these ideas are now common in modern user interface toolkits, especially the web, in the form of CSS and it's support for constraints, animations, and layout separate from interface behavior.

Other research has looked beyond traditional WIMP interfaces, creating new architectures to support new media. The DART toolkit, for example, invented several abstractions for augmented reality applications<gandy14>. Researchers contributed architectures for digital ink applications<hong00>, zoomable interfaces<bederson00b>, peripheral displays that monitor user attention<matthews04>, data visualizations<bostock09>, tangible user interfaces made of physical components<greenberg01,klemmer04>, interfaces based on proximity between people and objects<marquardt11>, and multi-touch gestures<kin12>. Another parallel sequence of work explored the general problem of handling events that are uncertain or continuous, investigating novel architectures and error handling strategies to manage uncertainty<hudson92,mankoff00,schwarz10>. Each one of these toolkits contributed new types of events, event handling, event propagation, synthetic event processing, and model-view-controller architectures tailored to these inputs.

While much of the work in user interface architecture has sought to contribute new architectural ideas for user interface construction, some have focused on ways of _modifying_ user interfaces without modifying their underlying code. For example, one line of work has explored how to express interfaces abstractly, so these abstract specifications can be used to generate many possible interfaces depending on which device is being used<edwards94,nichols02,nichols06>. Other systems have invented ways to modify interface behavior by intercepting events at runtime and forcing applications to handle them differently<eagan11>. Some systems have explored ways of directly manipulating interface layout during use<stuerzlinger06> and transforming interface presentation<edwards97>. More recent techniques have taken interfaces as implemented, reverse engineered their underlying commands, and generated new, more accessible, more usable, and more powerful interfaces based on these reverse engineered models<swearngin17>.

A smaller but equally important body of work has investigated ways of making interfaces easier to test and debug. Some of these systems expose information about events, event handling, and finite state machine state<hudson97b>. Some have invented ways of recording and replaying interaction data with interfaces to help localize defects in user interface behavior<newman10,burg13>. Some have even investigated the importance of testing security vulnerabilities in user interfaces, as interfaces like copy and paste transact and manipulate sensitive information<roesner12>.
 
Considering this body of work as a whole, there are some patterns that become clear: 

* Model-view-controller is a ubiquitous architectural style in user interface implementation.
* User interface toolkits are essential to making it easy to implement interfaces.
* New input techniques require new user interface architectures, and therefore new user interface toolkits.
* Interfaces can be automatically generated, manipulated, inspected, and transformed, but only within the limits of the architecture in which they are implemented.
* The architecture an interface is built in determines what is difficult to test and debug.

These "laws" of user interface implementation can be useful for making predictions about the future. For example, if someone proposes incorporating a new sensor in a device, subtle details in the sensor's interactive potential may require new forms of testing and debugging, new architectures, and potentially new toolkits to fully leverage its potential. That's a powerful prediction to be able to make and one that many organizations overlook when they ship new devices.
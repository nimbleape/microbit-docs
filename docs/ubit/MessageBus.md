#uBit.messageBus

##Overview

The micro:bit has an eventing model that can notfy user code when specific things happen on the micro:bit. For 
example, the MicroBitAccelerometer will raise events to indicate that the
micro:bit has be been shaken, or is in freefall. MicroBitButton will send events on a range of button up, down, click and hold events. 
Programmers are also free (in fact, encouraged!) to send their own events whenever they feel it would be useful. 

## Registering an event handler

The MicroBitMessgeBus records which events your program is interested in, and delivers those MicroBitEvents to your program as they occur
through a defined **event handler**.  You do this through the MicroBitMessageBus ```listen``` function. This lets you attach a callback
to a funciton when a specified event (or events) occur. You can also control the queuing and threading model used for your callbck funciton
on a per event handler basis.

This may sound complex at first, but it is actually very simple. For example, to find out when button A is clicked, write some code like this:

```cpp
void onButtonA(MicroBitEvent e)
{
    uBit.display.print("A");
}

int main()
{
    uBit.messageBus.listen(MICROBIT_ID_BUTTON_A, MICROBIT_BUTTON_EVT_CLICK, onButtonA);
}
```

Now, whenever the MICROBIT_BUTTON_EVT_CLICK event is raise by MICROBIT_ID_BUTTON_A, your code inside function 'onButtonA' will be automatically run.  
You can call listen as many times as you want to attached functions to each of the events that are useful for your program. 

##Wildcard Events
Sometimes though, you want to capture all events genareted by some component. For example, you might want to know when any chages in a button has happened. 
In this case, there is a special event value called 'MICROBIT_EVT_ANY'. If you call listen with this value, then ALL events from the given source component will be delivered to your function.
You can find out which ones by looking at the MicroBitEvent delivered to your function - it contains the source and value variable of the event. For example, you could write
a program like this:

```cpp
void onButtonA(MicroBitEvent e)
{
    if (e.value == MICROBIT_BUTTON_EVT_CLICK)
        uBit.display.scroll("CLICK");

    if (e.value == MICROBIT_BUTTON_EVT_DOWN)
        uBit.display.scroll("DOWN");
}

int main()
{
    uBit.messageBus.listen(MICROBIT_ID_BUTTON_A, MICROBIT_EVT_ANY, onButtonA);
}
```

If you *REALLY* want even more events, there is also a MICROBIT_ID_ANY source, that allows you to attach a function to event generated from any component... Use this
sparingly though, as this could be quite a lot of events! e.g. The following code would attach the onEvent function to receive all the events from the whole runtime:


```cpp
void onEvent(MicroBitEvent e)
{
    uBit.display.scroll("SOMETHING HAPPENED!");
}

int main()
{
    uBit.messageBus.listen(MICROBIT_ID_ANY, MICROBIT_EVT_ANY, onEvent);
}
```

## Defining a Threading Mode

Whenever you register a listener, you may choose the threading mode used with that handler. Every event handler can have its own threading mode, that defines when your handler
will be executed, and how it will react to receiving multiple events. There are four permissable modes for event handlers. These are:

| Threading mode | Brief Description |
| ------------- |-------------|
| MESSAGE_BUS_LISTENER_IMMEDIATE | Handler is called directly from the code raising the event. Event handler is *not* permitted to block. |
| MESSAGE_BUS_LISTENER_DROP_IF_BUSY | Handler is executed through its own fiber. If another event arrives whilst the previous event is still being proessed, the new event will be silently dropped. |
| MESSAGE_BUS_LISTENER_QUEUE_IF_BUSY | Handler is executed though its own fiber. If another event arrives, it is queued, and the event handler will immediately be called again once processing is complete. (default) |
| MESSAGE_BUS_LISTENER_REENTRANT | Every event is executed in its own fiber. if another event arrives, it is handled concurrently in its own fiber. |

These various modes provide great flexibility in how the runtime can be used to support higher level languages and applications. For example, MESSAGE_BUS_LISTENER_IMMEDIATE is ideal for very simple, lightweight handlers, as this will provide very timely response to events with a low processing overhead. However, it is easy to cause side effects on other parts of the code if it doesn not return promptly. MESSAGE_BUS_LISTENER_DROP_IF_BUSY provide semantics identical to the Scratch programming language, so can be used to build easy to understand, asynchronous environmnets. MESSAGE_BUS_LISTENER_QUEUE_IF_BUSY provides similar semantics, but with tolerance to avoiding loss of high frequency events. MESSAGE_BUS_LISTENER_REENTRANT provides guaranteed causal ordering and improved concurrency, but at the cost of additonal complexity and RAM.

You can define the threading mode you want to use on a per event handler basis as an optional final parameter to the listen funciton. For example:

```cpp
bool pressed = false;

void onButtonA(MicroBitEvent e)
{
    pressed = true;
}

int main()
{
    uBit.messageBus.listen(MICROBIT_ID_BUTTON_A, MICROBIT_BUTTON_EVT_CLICK, onButtonA, MESSAGE_BUS_LISTENER_IMMEDIATE);
}
```

## C++ Event Handlers

It is also posible to write event handlers as C++ member functions. If you don't know what this means, then don't worry, as that also means you won't need it. :-)
For those programmers who do like to write C++, you can use a variation of the ```listen``` funciton to register you member funciton event handler. This takes the same form as
the examples above, but with an adiditonal parameter to specify the object to call the method on. You also need to specifiy your event handler using legal C++ syntax. 

For example, you can write code like this to register an event handler in your own class:

```c++
MyCoolObject::onButtonPressed(MicroBitEvent e)
{
    uBit.display.print("A");
}

MyCoolObject::MyCoolObject()
{
    uBit.messageBus.listen(MICROBIT_ID_BUTTON_A, MICROBIT_BUTTON_EVT_CLICK, this, &MyCoolObject::onButtonPressed);
}
```

Again, it is also possible to a threading mode as an optional final parameter:

```c++
MyCoolObject::onButtonPressed(MicroBitEvent e)
{
    uBit.display.print("A");
}

MyCoolObject::MyCoolObject()
{
    uBit.messageBus.listen(MICROBIT_ID_BUTTON_A, MICROBIT_BUTTON_EVT_CLICK, this, &MyCoolObject::onButtonPressed, MESSAGE_BUS_LISTENER_IMMEDIATE);
}
```

## Removing Event Handlers

Event handlers can be dynamically removed from the message bus as well as added. To do this, use the ```ignore``` function. This takes precisely the same parameters as the
listen function, except that the threading mode argument is never used.

For example, to remove the event handlers shown above:

```c++
    uBit.messageBus.ignore(MICROBIT_ID_BUTTON_A, MICROBIT_BUTTON_EVT_CLICK, onButtonA);
    uBit.messageBus.ignore(MICROBIT_ID_BUTTON_A, MICROBIT_BUTTON_EVT_CLICK, this, &MyCoolObject::onButtonPressed);
```

##Message Bus ID
| Constant | Value |
| ------------- |-------------|
| MICROBIT_ID_MESSAGE_BUS_LISTENER | 1021 |

* The message bus will send a MICROBIT_ID_MESSAGE_BUS_LISTENER event whenever a new listener is added to the message bus. This event allows other parts of the system to detect when interactions are
taking place with a component. This is primarily used as a power management mechanism - allowing on demand activation of hardware when necessary.

##Message Bus Events
| Constant | Value |
| ------------- |-------------|
| Message Bus ID of listener | 1-65535 |


##API
[comment]: <> ({"className":"MicroBitMessageBus"})
##Constructor
<br/>
####MicroBitMessageBus()
#####Description
Default constructor.
##send
<br/>
####<div style='color:#FF69B4; display:inline-block'>int</div> send( <div style='color:#008080; display:inline-block'>MicroBitEvent</div> evt)
#####Description
Queues the given event to be sent to all registered recipients.
#####Parameters

>  <div style='color:#008080; display:inline-block'>MicroBitEvent</div> *evt* - The event to send.
#####Example
```cpp
 MicroBitMessageBus bus; 
 
 // Creates and sends the MicroBitEvent using bus. 
 MicrobitEvent evt(MICROBIT_ID_BUTTON_A, MICROBIT_BUTTON_EVT_CLICK); 
 
 // Creates the MicrobitEvent, but delays the sending of that event. 
 MicrobitEvent evt1(MICROBIT_ID_BUTTON_A, MICROBIT_BUTTON_EVT_CLICK, CREATE_ONLY); 
 
 bus.send(evt1); 
 
 // This has the same effect! 
 evt1.fire() 
```
##process
<br/>
####<div style='color:#FF69B4; display:inline-block'>int</div> process( <div style='color:#008080; display:inline-block'>MicroBitEvent  &</div> evt)
#####Description
Internal function, used to deliver the given event to all relevant recipients. Normally, this is called once an event has been removed from the event queue.
#####Parameters

>  <div style='color:#008080; display:inline-block'>MicroBitEvent  &</div> *evt* - The event to send.
#####Returns
1 if all matching listeners were processed, 0 if further processing is required.

!!! note
    It is recommended that all external code uses the  send()  function instead of this function, or the constructors provided by MicrobitEvent. 

<br/>
####<div style='color:#FF69B4; display:inline-block'>int</div> process( <div style='color:#008080; display:inline-block'>MicroBitEvent  &</div> evt,  <div style='color:#008080; display:inline-block'>bool</div> urgent)
#####Description
Internal function, used to deliver the given event to all relevant recipients. Normally, this is called once an event has been removed from the event queue.
#####Parameters

>  <div style='color:#008080; display:inline-block'>MicroBitEvent  &</div> *evt* - The event to send.

>  <div style='color:#008080; display:inline-block'>bool</div> *urgent* - The type of listeners to process (optional). If set to true, only listeners defined as urgent and non-blocking will be processed otherwise, all other (standard) listeners will be processed. Defaults to false.
#####Returns
1 if all matching listeners were processed, 0 if further processing is required.

!!! note
    It is recommended that all external code uses the  send()  function instead of this function, or the constructors provided by MicrobitEvent. 

##elementAt
<br/>
####<div style='color:#FF69B4; display:inline-block'>MicroBitListener</div> elementAt( <div style='color:#008080; display:inline-block'>int</div> n)
#####Description
Returns the microBitListener with the given position in our list.
#####Parameters

>  <div style='color:#008080; display:inline-block'>int</div> *n* - The position in the list to return.
#####Returns
the  MicroBitListener  at postion n in the list, or NULL if the position is invalid. 
##add
<br/>
####<div style='color:#FF69B4; display:inline-block'>int</div> add( <div style='color:#008080; display:inline-block'>MicroBitListener  *</div> newListener)
#####Description
Add the given  MicroBitListener
#####Parameters

>  <div style='color:#008080; display:inline-block'>MicroBitListener  *</div> *newListener*
#####Returns
MICROBIT_OK if the listener is valid, MICROBIT_INVALID_PARAMETER otherwise. 
##remove
<br/>
####<div style='color:#FF69B4; display:inline-block'>int</div> remove( <div style='color:#008080; display:inline-block'>MicroBitListener  *</div> newListener)
#####Description
Remove the given  MicroBitListener
#####Parameters

>  <div style='color:#008080; display:inline-block'>MicroBitListener  *</div> *newListener*
#####Returns
MICROBIT_OK if the listener is valid, MICROBIT_INVALID_PARAMETER otherwise. 
____
[comment]: <> ({"end":"MicroBitMessageBus"})
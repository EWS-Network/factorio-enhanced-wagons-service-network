Extended Wagons System Network
======================

This page is here to explain how the Factorio EWS Network works. It has been sightly inspired by the VTN which at the time of finishing all my blueprints I did not know, so I borrowed the bit shifting given there is no bitmasking capability in the combinators.


TLDR; `https://youtu.be/3yMbtP663Yw`_


Global Network Strucure
=======================

The idea was simple: I wanted to find a way in Vanilla mode to implement what the Logic Train Network mod does. I could have just used the LTN but I do not like mods in a game.
So, I accepted the fact that some things will need "hard-coding" : the Station names and the train's paths.

I was doing these blueprints in preparation of a new map in which all parts of the base are using trains (belts only are local to convey / unload the train payload and do something with it, so no main bus).


So without further a due, let's dive into it.


The Supply Chain
================

I am no Logistic genius, so I looked at how distribution networks IRL work: Supplier, Dispatcher, Carrier, Consumer (which I called requester in my templates).

Supplier
--------

The Suppliers would be:

* The mining outposts, far remote locations which I do not need to build much but to provide power for and some ammo and walls to fight the biters
* The factories building the main components that would ingest into a MALL (plates, steel, circuits, liquids).

In order to have an asynchronous system which allows us to turn on and off these outposts.


At the time of writting this (14th Auh. 2019) I have not spent any time doing blueprints for these stations as most people have their own way of doing it.


Dispatcher
----------

The dispatcher stations are placed in periphery of your base: they gather the resources from the suppliers and fill up. If we have 8 suppliers and 4 Dispatcher stations for ores for example, chances are that the dispatchers will always have enough resources coming in to fill up trains.


The dispatchers logic is based on the length of the capacity that 1 Wagon can contain is currently stored in the station. We will come to more details about that in the standby.


Standby
-------

For lack of better wording, this is a stacker station in which trains are going to wait for a Dispatcher station to have enough resources in order to fill itself up entirely.
At this point in my blueprints, I have not been able to find a smart way (yet) to have the train be routed to the station that can indeed load it up as opposed to the sum of resources across for all wagons.


Carrier station
---------------

Once a train has gone through the Dispatcher, it is routed to a stacker station called `Carrier`. Effectively, this train is going to take the resources down to the requester/consumer station that needs that resource.

For those used to the VTN network, instead of making the train wait at the `Out` station, full, it is waiting in a stack station already full. So we can have more than one train completely full and ready to go.

Typically you would want to have the Carrier station as close as possible to the stations that are going to need the resources.


Requester / Consumer Station
----------------------------

As it names says, this is the station that is going to get their stuff delivered by the carrier. We will dive into the logic of the station circuit in a minute, but essentially, if not considered "full" the station will broadcast how many wagons worth of resources it wants to have.


How the circuits network works
==============================


Before looking into the stations, we need to look at the "brain" of the system which is what splits from the wire network the supply and demand numbers.
As everything is carried out via a red wire across the entire base, we need the brain to split and distinguish those two.

We want it to be close to the stackers for standby and carriers, because the trains will exit these stations based on the numbers the brain will output (ie. carrier goes when a dispatcher broadcasts its needs).


The brain logic
---------------

The dot. signal
^^^^^^^^^^^^^^^

Because I hate defining values staticly, but I have to in some places, I then placed the dot signal in the brain. (You must be sure that only one constant combinator is connected to the global network if you are going to have multiple brains otherwise the value associated with the dot signal will be added up together.

By default, I have it set to 16.

What does the Dot. signal do ? Well, it regulates globally the bit shifting for each and every stations that is going to use bit shifting: basically, bit shifting is going to take your decimal number, convert it to binary, and add `0` after it. So for example, 7(10), which is 0111(2), will become 0111 0000 0000 0000 0000(2) which equals 458752(10).


The redwire will carry the negative amount of requested resources (so the demand number) as well as the postive amount, non shifted, of a given resource available. The number is the number of wagons worth that the network has to offer, not the amount of resources itself (18 means that the amount is wagons * stacks in wagon * stack size, so for iron, 18*40*50=8k, for green circuits it would be 18*40*200=32k).

If at some point in the game you have gigantic values for these resources with a shift of 16, you can decrease the number so the game will be able to compute the values (the max value in the game is a signed int in C from errors I am getting. Types and values `here<https://www.tutorialspoint.com/cprogramming/c_data_types.htm>`_).

The maths
^^^^^^^^^

First we are going to retrieve the number of wagons for each type of resources that the network is requesting.

We also want to retrieve the number of wagons available on the network. We then have to negate the bitshifted value of requested resources, and add it up to the input, which is going to leave only the positive value out (the difference).

Why ? So let's say we have 48 wagons worth of ore available and 19 requested.
On the network we will have:

* requested = 19 * -1 = -19 << 16 (dot signal) = -1245184

* supply = 48

Total = −1245136 (-1245184 + 48)

Which would be 0001 0010 1111 1111 1101 0000(2). (plus the sign bit which makes the difference between positive and negative values).

After we bit shifted the first time, our 48 has "disapeared" from, and we only have 19. If we bit shift 19 again to get 1245184, and add add it to −1245136, we get our 48 value back.


I am sure there is a much nicer way to explain it, but that is it in a nutshell.


Stations logic
==============

Common design
-------------

The design and logic of the stations is pretty much identical for the dispatcher and the supplier, at the difference than one sends the resources and the other one gets them and therefore have inverted output logic.

Balanced Loading/Unloading
^^^^^^^^^^^^^^^^^^^^^^^^^^

On Reddit I found lots of people having issues with their loading and unloading not being balanced (most of the time the unloading), based on their design of belts that take the ore out to the factories/smelters. And I had that problem too which was wrecking my ratios.

The design in place is used by many and is not new, it's been done in many ways by a lot of people before: take the number of chests / inserters, the quantity of resources in the chest globally, divide by the inserters, which gives you an average. Closely connect the inserters together on 1 wire (red here) and with the other color wire (green here) connect to the chest it is emptying. If the difference between the average and the quantity in the chest is below 0, activate.

I have very successfully used that system in many bases and when the resources are gone, they are all gone at once.


To achieve that, here is the logic

Static combinator -> outputs the inserter signal with the value of how many inserters per wagon are in play on one side (usually designs go for 4 or 6 of these).
Static combinator -> outputs the rail signal which is either -1 or -2 which you set depending on whether you are unloading / loading from 1 side or both sides of the wagon.

And that is all you have to change yourselves if you want to.

Now for the maths of it:

D = Divider = length of the station (measured by the number of train signals, red + green) * inserters * rail signal. So for a 4 wagons train, on one side, with 6 inserters, we get 6 * 4 * -1 = -24.
The output of all the chests chained together will give us the quantity of resources in the chests.

It is then divided by D, we remove D from the exit signal to output only the chests content and feed that average value back to the inserters.


Resources count
---------------

Instead of a long and boring paragraph trying to explain the logic of it, a simple diagram will do the job far better

.. image:: station_maths_logic.jpeg



Dispatcher station
------------------

The way the blueprints have been made, you can chain tile *extension* after the head of the station, and close up with the station *tail*.
The minimum size for a station is 1 locomotive and 1 wagon.

.. note::

   at the time of doing the blueprints, I only designed those for X locomotive at the front of the train and.

Each time you put down a *loader extension* (which comes in single sided and double sided versions), the maths will automatically measure how many extensions you have put down to measure how much wagons this station is worth of resources.

The idea is that your dispatcher station can very well be 8 wagons long if you so desire to dispatch gigantic trains for that resource type, or 2 of length if you feel like 1 train worth of that resource is already plenty (ie. red circuits, 2 wagons train is already 16k worth). So in that way you can actually use the train station size to make up your ratios.

The *tail* of the station is here to force stop trains because we will be using all other signals to measure the train in station, and this train might be shorter than the station itself (I am sure people will find use-cases for those).


Variables in the Constant Combinator
------------------------------------

* Iron-Chest : 32: The number of storage slots in the iron chest

* Steel Chest: 48: The number of storage slots in the steel chest

* Wagon : 40: Number of stacks in a wagon

* S: Stack Size: Size of the resource stored in a single slot (ie. 50 for ores, 100 for plates, 200 for Green Circuits).

* Rail : 1/2: Whether your station is one or two sided for load/unloader

* Inserter: 1-6: The number of unloader per wagon on a **single side** of it. (usually would be 4 or 6)

Blueprints names and codes
--------------------------

Some of the templates have an extension with groups of letters and numbers. Here is the decode:

<Loader|Unloader>

<0-9>W - number of cargo wagons

<1S|2S> - single sided or dual sided (from the train track)

<FI|SI|FFI|FSI> - Fast Inseter | Stack Inserter | Filter Fast Inserter | Filter Stack Inserter

<X>C - Number of chests on one side of the wagon (refer to `Inserter` above)

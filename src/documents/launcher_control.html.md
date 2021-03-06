---
title: "Node Rockets Launcher Control System"
layout: "default"
isPage: true
---

Launcher Control System
=======================

Goals
-----

The goals for the launcher control system are so far pretty simple:

1. Measure the air pressure inside the rocket.
2. Pressurize the rocket automatically.
3. Launch the rocket automatically.

Plans
-----

This is noderocket, so we know we're going to use nodejs.  For the launcher, we decided to use an Arduino based platform running johnny-five for the interface to the hardware.  Arduino gives us a series of analog and digital input/output pins which we can use to measure or to turn things on and off.

### Measuring Air Pressure

We spent some time looking for an inexpensive, easy to find, pressure sensor that can measure up to about 100 PSI.  We found a great fit in the oil pressure sensors in our cars:

![oil pressure sensor](images/oilpresssender.jpg)

An oil pressure sensor varies its electrical resistance in response to the amount of pressure pushing against the sensor.  If we can measure the resistance of the pressure sensor we can calculate the pressure pushing against it.  In electronics, the way to measure an unknown resistance is to use a [voltage divider circuit](http://en.wikipedia.org/wiki/Voltage_divider).

![voltage divider circuit](images/Impedance_Voltage_divider.png)

In this case, we will use a known V<sub>in</sub> and Z<sub>1</sub>, we will measure V<sub>out</sub> by connecting it to an analog pin on the Arduino, and we can then calculate Z<sub>2</sub> using the following equation:

![voltage divider equation](images/voltage_divider_equation.gif)

Our V<sub>in</sub> will be the +5 V from the Arduino.  We need to pick a reasonable Z<sub>1</sub>.  The resistance of the pressure sensor ranges from close to nothing to between 150 and 200 Ω.  So we have to balance two needs:

1. We want to minimize the current we consume ([Ohm's Law](http://en.wikipedia.org/wiki/Ohms_law): I = V/R), which means we want a high Z<sub>1</sub>.
2. We want to maximize the voltage change as the sensor resistance changes, which means we want a low Z<sub>1</sub>.

The larger we make Z<sub>1</sub>, the less of a voltage change we get from a change in resistance on the sensor.  The Arduino reads voltage from 0 to 5 V as digital values 0 to 1023, where 0 is 0 V and 1023 is 5 V.  This mean that the Arduino reads in increments of 0.00488 V.  A Z<sub>1</sub> of around 300 Ω will give us a voltage range of 0 V to 2 V, assuming the sensor is 0 Ω at 0 PSI and 200 Ω at 100 PSI.  This means we will read pressure in increments of about 0.244 PSI, which is good enough for our rocket launcher.  The maximum current we will consume is 16 mA, which isn't great, but we can live with it.

** Calculating Pressure From Resistance **

The pressure sensor resistance changes close enough to linearly that we're going to use a simple [linear equation](http://en.wikipedia.org/wiki/Linear_equation) to determine the pressure from the resistance.  We have to determine the parameters of that equation experimentally.  Using a digital multimeter we measure the resistance of the sensor at two known pressures, then we determine the slope and y-intercept from those numbers. When I measured the resistance of my sensor I got the following readings:
- At 10 PSI resistance was 22.31648 Ω.
- At 80 PSI resistance was 150.077 Ω

Using amazing algebra skills, we get an equation that looks like this:

> psi = (Z<sub>2</sub> * 0.5479) - 2.2272;

### Pressurizing and Launching

Pressurizing the rocket is just a matter of opening a valve to let pressurized air into the rocket, then closing it again.

![air valve](images/air_valve.png)

We considered several options for triggering the rocket launch.  Our current solution uses a pneumatic cylinder to move a piece of material holding the rocket to the launcher.  The pneumatic cylinder is powered by air pressure, which is controlled by a valve.  So our control circuit for launching the rocket will be the same as the control circuit for pressurizing the rocket.

![air cylinder](images/air_cylinder.png)

** Controlling the Valves **

The air valves operate at 12 V and draw about 1 A when first opened, dropping to a sustained 500 mA or so for as long as they stay open.  This is a lot more power than the Arduino can provide.  The oil pressure sensor, since it comes from a car, expects to operate in a common ground environment.  This means that all of our components should normally be connected to ground, and the valves should be switched on by switching operating voltage of 12 V to them.

We're going to use transistors to do the switching for us.  Since we want the valves to be connected to ground and switch on power, we will need to use [P channel transistors](http://en.wikipedia.org/wiki/PNP_transistor#PNP) as the switch to the valve.  Since the valve has a relatively high power consumption, we'll need to use high powered transistors, like [MOSFETs](http://en.wikipedia.org/wiki/Mosfet).

The Arduino digital output pin effectively has three states: low (0 V), high (5 V), and high impedence (no value, like during startup).  In order to make sure the valve stays closed unless switched on we want a small NPN transistor between the Arduino and the MOSFET.  This transistor will make sure that if the Arduino gives a weak or indeterminate value on the pin, the MOSFET still gets a strong "off" signal.  We'll be using this circuit:

<a href="images/launcher_valve_schematic.png"><img src="images/launcher_valve_schematic.png" alt="Valve Schematic" style="width: 500px;"/></a>

Our LOAD is the air valve.  There is what's called a "pull down" resistor on the NPN transistor that makes sure that if the Arduino IO isn't giving a strong value, the NPN input is tied to ground, which turns off an NPN transistor.  The P MOSFET has a "pull up" resistor, which does the same thing if the NPN transistor is giving a weak output.  A P channel transistor is off if its input is high.

Since our LOAD is a solenoid, which is a type of inductor coil, it will discharge current in a backwards direction when it's turned off.  In order to protect the transistors we will include a diode across the load.  A diode allows current to flow in only one direction.  We will point the diode so that when the transistor turns the valve on, no current will flow through it, but any current created by the valve when it turns off will flow through the diode, protecting the transistor.

### Building the Circuits

** Parts List **

We will create two valve control circuits, and the pressure measurement circuit.  We'll need the following parts:
- [An Arduino Kit (Uno)](http://www.amazon.com/Starter-Kit-Newsite-Uno-Breadboard/dp/B0051QHPJM/ref=sr_1_cc_1?s=aps&ie=UTF8&qid=1397853336&sr=1-1-catcorr&keywords=arduino+holder)
   - Arduino Uno
   - USB cable
   - Breadboard
   - plenty of jumper wires
- [6 10k Ω resistors](https://www.sparkfun.com/products/11508)
- [1 270 Ω resistor](https://www.sparkfun.com/products/11507)
- [2 NPN transistors (5 V input)](https://www.sparkfun.com/products/12852)
- [2 P channel MOSFETs (5 V input)](https://www.sparkfun.com/products/10349)
- [2 diodes](https://www.sparkfun.com/products/8589)
- [1 DC barrel jack adapter](https://www.sparkfun.com/products/10811)
- [1 15V (or so) power supply](http://www.amazon.com/gp/product/B0027C16H0/ref=oh_details_o01_s00_i00?ie=UTF8&psc=1)
- [1 CAT6 keystone jack](http://www.monoprice.com/Product?c_id=105&cp_id=10513&cs_id=1051303&p_id=5622&seq=1&format=2)
- about 6 inches of CAT5 or CAT6 cable, preferably solid core (bulk cable is usually solid, patch cables are usually stranded - solid plugs into a breadboard more easily)

In order to make it easy to connect to the valves and sensor, we included a CAT6 keystone jack.  The circuit board will be connected to the control assembly (valves and sensor) through a network cable.  This makes it easier to set up and tear down the launcher.

**** WARNINGS ****
- The direction of the diodes matters.  If you put them in backwards you are short circuiting your transistors.
- Don't plug the 15 V into the Arduino power jack.  It fits, but it's way too much power.  You could damage the Arduino.
- Different transistors may use different pins than the ones in the diagrams/pictures.  If you aren't using the same parts listed above, make sure the pins match for base, collector, and emitter (or gate, source, and drain).  If they don't match, adjust your wiring accordingly.

Put all together, it should look like this:

[![project circuit layout](images/launcher_circuit_small.png)](images/noderocket-launcher_electronics_bb.png)

[Larger Version](images/noderocket-launcher_electronics_bb.png)

[Fritzing File](images/noderocket-launcher_electronics.fzz)

And here is a real life version:

<a href="images/launcher_control_board.jpg"><img src="images/launcher_control_board.jpg" alt="Spark Core" style="width: 500px;"/></a>
[Larger Version](images/launcher_control_board.jpg)

### Using a Spark Core

<img src="images/spark.jpg" alt="Spark Core" style="width: 500px;"/>

The [Spark Core](https://www.spark.io/) is a small, Arduino compatible board with Wifi built in.  It's close to a drop in replacement for the Arduino in our control system circuit, with a couple key differences:

- The Spark Core is a 3.3V system, not a 5V system.
- The data connection is WiFi, not USB.

** Dealing with 3.3V **

Our circuit is expecting 5V - that's what works best with our transistors.  The Spark Core is powered by USB, which is 5V.  The Vin pin outputs 5V.

The analog pins can't handle 5V, only 3.3V.  That means we need to make sure our pressure sensor never gives the analog pin more than 3.3V.  There are two ways to do this:

- Use a 3.3V pin as the power supply to the voltage divider (only the voltage divider).
- Power the voltage divider from 5V, but make sure max that can reach the analog pin is 3.3V or under.

I'm using the first solution, which is easy to adjust for in the software.

Here's the circuit with a spark device:

<img src="images/spark_circuit.jpg" alt="Spark Core Circuit" style="width: 500px;"/>

** Connecting without USB **

This is mostly handled with a software change.  The noderocket-launcher project includes a "spark" branch that has the updates needed to connect to a spark device.  Here's what you need to do:

- Register your spark device at [https://www.spark.io](https://www.spark.io).
- Prepare the spark device following the instructions for [the spark-io project](https://github.com/rwaldron/spark-io/).
- Clone the [noderocket-launcher project](https://github.com/noderockets/noderocket-launcher) and switch to the spark branch.
- Find your Spark Access Token at [https://www.spark.io/build](https://www.spark.io/build) under the Settings section.
- Find your spark device's Device ID at [https://www.spark.io/build](https://www.spark.io/build) under the Cores section.
- Start the launcher server using:

        node server.js --token <your token> --deviceId <your device id>

### Building the Valve Assembly

We need the valves and pressure sensor connected such that:
1. We can connect an air pressure source.
2. We can connect a rocket launch platform.
3. The fill valve controls air into the rocket.
4. The pressure sensor senses the air pressure of the rocket.
5. The launch valve has pressure when the rocket is ready to launch.
6. The launch valve loses pressure when the rocket is launched.

** Parts List **

We'll need the following parts:
- [2 12 V air valves](http://www.amazon.com/Electric-Solenoid-Replacement-Pipelines-Applications/dp/B00827FP26/ref=sr_1_sc_2?ie=UTF8&qid=1395697129&sr=8-2-spell)
- [1 oil pressure sensor](http://www.amazon.com/gp/product/B00029JXMA/ref=oh_details_o00_s00_i00?ie=UTF8&psc=1)
   - 1 1/8 NPT female to 1/4 NPT male adapter (comes with the pressure sensor linked)
- [2 Quick connect starter sets](http://www.harborfreight.com/piece-brass-industrialmilton-uick-connect-starter-set-68237.html)
   - 2 Female quick connect to 1/4 NPT female
   - 2 Male quick connect to 1/4 NPT male
- [2 1/4 NPT tees](http://www.harborfreight.com/14-female-brass-t-connector-68197.html)
- [1 1/4 NPT female coupling](http://www.harborfreight.com/14-x-14-Female-Brass-Pipe-Coupling-68200.html)
- [3 1/4 NPT male coupling](http://www.harborfreight.com/14-in-x-14-in-male-brass-pipe-coupling-68198.html)
- [1 1/8 NPT male to 1/4 NPT female coupling](http://www.amazon.com/dp/B000BOAA6O/ref=biss_dp_t_asn)
- [1 pneumatic cylinder](http://www.amazon.com/gp/product/B005JGR0GM/ref=oh_details_o00_s00_i00?ie=UTF8&psc=1)
- [1-2 foot hose 1/4 NPT male on both ends to connect valve to cylinder](http://www.amazon.com/K-Tool-International-KTI72000-Rubber-Whip/dp/B00BEEF52E/ref=sr_1_2?s=hi&ie=UTF8&qid=1397851143&sr=1-2)
- [Electrical tape](http://www.harborfreight.com/60-ft-x-34-in-industrial-grade-electrical-tape-69570.html)
- [M5 screw to connect the sensor ground to a valve body](http://www.amazon.com/Machine-Plated-Phillips-Threaded-Imported/dp/B00F32EKMO/ref=sr_1_1?ie=UTF8&qid=1401656736&sr=8-1&keywords=m5+screw+10mm)
- [pipe thread tape](http://www.harborfreight.com/1-2-half-inch-x-260-inch-plumbers-thread-seal-tape-39625.html)
- [1 CAT6 keystone jack](http://www.monoprice.com/Product?c_id=105&cp_id=10513&cs_id=1051303&p_id=5622&seq=1&format=2)
- about 6 inches of CAT5 or CAT6 cable, preferably solid core (bulk cable is usually solid, patch cables are usually stranded - solid plugs into a breadboard more easily)

![launcher control parts](images/launcher_control_parts.png)

Assemble them as shown in the picture - make sure to apply tape to the threads of each male piece before assembling.  Tighten the connections.  It should look like this when assembled:

![launcher control assembly](images/launcher_control_assembly.png)

The pneumatic cylinder isn't pictured as it's bolted on to the launch platform.  It has a male quick connect and the hose connecting it to the valve has a female quick connect so that the control assembly is removed from the launch platform via two quick connects - one to the rocket fill and the other to the pneumatic cylinder.

And both sides assembled with CAT6 keystone jacks - a network cable completes the connection between the two:

![launcher control and circuit](images/launcher_control_cat6.png)
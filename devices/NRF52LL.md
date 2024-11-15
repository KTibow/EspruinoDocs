<!--- Copyright (c) 2017 Gordon Williams, Pur3 Ltd. See the file LICENSE for copying permission. -->
nRF52 Low Level Interface Library
=================================

<span style="color:red">:warning: **Please view the correctly rendered version of this page at https://www.espruino.com/NRF52LL. Links, lists, videos, search, and other features will not work correctly when viewed on GitHub** :warning:</span>

* KEYWORDS: Module,nRF52,nRF5x,nRF52832,Low Level,Hardware,Differential,SAADC,GPIOTE,LPCOMP,PPI,TIMER
* USES: Puck.js,Pixl.js,Jolt.js,MDBT42Q,nRF52832,nRF52

The nRF52 microcontroller used in [Puck.js](/Puck.js), [Pixl.js](/Pixl.js) and [MDBT42Q](/MDBT42Q) has a load of really interesting peripherals built-in, not all of which are exposed by Espruino. The microcontroller also contains something called PPI - the "Programmable Peripheral Interconnect". This allows you to 'wire' peripherals together internally.

PPI lets you connect an `event` (eg. a pin changing state) to a `task` (eg. increment the counter). All of this is done without the processor being involved, allowing for very fast and also very power efficient peripheral use.

Check out [the chip's reference manual](http://infocenter.nordicsemi.com/pdf/nRF52832_PS_v1.1.pdf) for more information.

This library ([[NRF52LL.js]]) provides a low level interface to PPI and some of the nRF52's peripherals.

**Note:** Failure to 'shut down' peripherals when not in use could drastically increase the nRF52's power consumption.


Basic Usage
-----------

* Initialise a peripheral to create events
* Initialise a peripheral you want to send tasks to
* Set up and Enable a PPI to wire the two together

The following are some examples:

### Count the number of times the BTN pin changes state

Uses GPIO and counter timer:

```JS
var ll = require("NRF52LL");
// Source of events - the button
var btn = ll.gpiote(7, {type:"event",pin:BTN,lo2hi:1,hi2lo:1});
// A place to recieve Tasks - a counter
var ctr = ll.timer(3,{type:"counter"});
// Set up and enable PPI
ll.ppiEnable(0, btn.eIn, ctr.tCount);
/* This function triggers a Task by hand to 'capture' the counter's
value. It can then be read back from the relevant `cc` register */
function getCtr() {
  poke32(ctr.tCapture[0],1);
  return peek32(ctr.cc[0]);
}
```

### Create a square wave on pin `D0`, with the inverse of the square wave on `D1`

Uses GPIO and counter timer:

```JS
var ll = require("NRF52LL");
// set up D0 and D1 as outputs
digitalWrite(D0,0);
digitalWrite(D1,0);
// create two 'toggle' tasks, one for each pin
var t0 = ll.gpiote(7, {type:"task",pin:D0,lo2hi:1,hi2lo:1,initialState:0});
var t1 = ll.gpiote(6, {type:"task",pin:D1,lo2hi:1,hi2lo:1,initialState:1});
// create a timer that counts up to 1000 and back at full speed
var tmr = ll.timer(3,{cc:[1000],cc0clear:1});
// use two PPI to trigger toggle events
ll.ppiEnable(0, tmr.eCompare[0], t0.tOut);
ll.ppiEnable(1, tmr.eCompare[0], t1.tOut);
// Manually trigger a task to start the timer
poke32(tmr.tStart,1);
```

### Toggle `LED` every time `D31`'s analog value goes above `VCC/2`

Uses low power comparator + GPIO:

```JS
var ll = require("NRF52LL");
// set up LED as an output
digitalWrite(LED,0);
// create a 'toggle' task for the LED
var tog = ll.gpiote(7, {type:"task",pin:LED,lo2hi:1,hi2lo:1,initialState:0});
// compare D31 against vref/2
var comp = ll.lpcomp({pin:D31,vref:8});
// use a PPI to trigger the toggle event
ll.ppiEnable(0, comp.eCross, tog.tOut);
```

**Note:** As of Espruino 2v25 you can set up the comparator to create a JS event with [`E.setComparator`](https://www.espruino.com/Reference#l_E_setComparator)

### Count how many times `D31` crosses `VCC/2` in 10 seconds

Uses low power comparator + counter timer:

```JS
var ll = require("NRF52LL");
// source of events - compare D31 against vref/2
var comp = ll.lpcomp({pin:D31,vref:8});
// A place to recieve events - a counter
var ctr = ll.timer(3,{type:"counter"});
// Set up and enable PPI
ll.ppiEnable(0, comp.eCross, ctr.tCount);
/* This function triggers a Task by hand to 'capture' the counter's value. It can then clear it and read back the relevant `cc` register */
function getCtr() {
  poke32(ctr.tCapture[0],1);
  poke32(ctr.tClear,1); // reset it
  return peek32(ctr.cc[0]);
}
// Every 10 seconds, wake and print out the number of crosses
setInterval(function() {
  print(getCtr());
}, 10000);
```

**Note:** As of Espruino 2v25 you can set up the comparator to create a JS event with [`E.setComparator`](https://www.espruino.com/Reference#l_E_setComparator)

### Use LED1 on Puck.js to sense a change in light level

[LED1 in Puck.js can be a light sensor](/Puck.js#light-sensor), and we
can use the low power comparator with this to detect a state change.

To make this work we have to use one IO pin (in this case D1) so that we
can toggle it with each change, and then watch it with `setWatch` for changes.

**Note:** On the MBDT42 breakout board, LED1 isn't attached to an analog
pin so this won't work. However LED2 **is**, so can still be used in this way.

```JS
var ll = require("NRF52LL");
var togglePin = D1;
analogRead(LED1);
digitalWrite(togglePin,0);
// create a 'toggle' task for togglePin
var tog = ll.gpiote(7, {type:"task",pin:togglePin,lo2hi:1,hi2lo:1,initialState:0});
// compare LED1 against 3/16 vref (vref is in 1/16 ths)
var comp = ll.lpcomp({pin:LED1,vref:3,hyst:true});
// use a PPI to trigger the toggle event
ll.ppiEnable(0, comp.eCross, tog.tOut);

// Detect a change on togglePin
setWatch(function() {
  // called twice per 'flash' (for light on and off)
  print("Light level changed");
}, togglePin, {repeat:true});
```

**Note:** As of Espruino 2v25 you can set up the comparator to create a JS event with [`E.setComparator`](https://www.espruino.com/Reference#l_E_setComparator)

### Make one reading from the ADC:

Uses the ADC (much line `analogRead` but with more options)

```JS
var ll = require("NRF52LL");
var saadc = ll.saadc({
  channels : [ { // channel 0
    pin:D31,
    gain:1/4,
    tacq:40,
    refvdd:true,
  } ]
});
print(saadc.sample()[0]);
saadc.stop(); // deconfigure so analogRead works again (use saadc.start() to redo)
```

### Make a differential from the ADC:

Use the ADC to measure the voltage difference between D30 and D31,
with the maximum gain and oversampling provided by the hardware.

```JS
var ll = require("NRF52LL");
var saadc = ll.saadc({
  channels : [ { // channel 0
    pin:D30, npin:D31,
    gain:4,
    tacq:40,
    refvdd:true,
  } ],
  oversample : 8
});
print(saadc.sample()[0]);
saadc.stop(); // deconfigure so analogRead works again (use saadc.start() to redo)
```

### Read a buffer of data from the ADC

Uses ADC.

It's also possible to use `.sample(...)` for this, but this example
shows you how to use it in more detail.

The ADC will automatically sample at the given sample rate.

```JS
var ll = require("NRF52LL");
// Buffer to fill with data
var buf = new Int16Array(128);
// source of events - compare D31 against vref/2
var saadc = ll.saadc({
  channels : [ { // channel 0
    pin:D31,
    gain:1/4,
    tacq:40,
    refvdd:true,
  } ],
  samplerate:2047, // 16Mhz / 2047 = 7816 Hz auto-sampling
  dma:{ptr:E.getAddressOf(buf,true), cnt:buf.length},
});
// Start sampling until the buffer is full
poke32(saadc.eEnd,0); // clear flag so we can test
poke32(saadc.tStart,1);
poke32(saadc.tSample,1); // start!
while (!peek32(saadc.eEnd)); // wait until it ends
poke32(saadc.tStop,1);
print("Done!", buf);
saadc.stop(); // deconfigure so analogRead works again (use saadc.start() to redo)
```

### Read a buffer of data from the ADC, alternating between 2 pins

Uses ADC and counter timer.

The NRF52 doesn't support using `samplerate` (as in the last example)
with more than one channel, so you have to use another timer to
trigger the `tSample` task.

```JS
var ll = require("NRF52LL");
// Buffer to fill with data
var buf = new Int16Array(128);
// ADC
var saadc = ll.saadc({
  channels : [ {
    pin:D31, // channel 0
    gain:1/4,
    refvdd:true
  }, {
    pin:D30, // channel 1
    gain:1/4,
    refvdd:true
  } ],
  dma:{ptr:E.getAddressOf(buf,true), cnt:buf.length},
});
// create a timer that counts up to 1000 and back at full speed
var tmr = ll.timer(3,{cc:[1000],cc0clear:1});
// use two PPI to trigger toggle events
ll.ppiEnable(0, tmr.eCompare[0], saadc.tSample);
// Start sampling until the buffer is full
poke32(saadc.eEnd,0); // clear flag so we can test
poke32(saadc.tStart,1);
// start the timer
poke32(tmr.tStart,1);
while (!peek32(saadc.eEnd)); // wait until sampling ends
poke32(tmr.tStop,1);
poke32(saadc.tStop,1);
print("Done!", buf);
saadc.stop(); // deconfigure so analogRead works again (use saadc.start() to redo)
```

### Use the RTC to toggle the state of a LED

Uses RTC, GPIO:

```JS
var ll = require("NRF52LL");

// set up LED as an output
digitalWrite(LED,0);
// create a 'toggle' task for the LED
var tog = ll.gpiote(7, {type:"task",pin:LED,lo2hi:1,hi2lo:1,initialState:0});

// set up the rtc
var rtc = ll.rtc(2);
poke32(rtc.prescaler, 4095); // 32kHz / 4095 = 8 Hz
rtc.enableEvent("eTick");
poke32(rtc.tStart,1); // start RTC
// use a PPI to trigger the toggle event
ll.ppiEnable(0, rtc.eTick, tog.tOut);
```

### Use the RTC to measure how long a button has been held down for:

Uses RTC, GPIO:

```JS
var ll = require("NRF52LL");
// Source of events - the button
// Note: this depends on the polarity of the physical button (this assumes that 0=pressed)
var btnu = ll.gpiote(7, {type:"event",pin:BTN,lo2hi:1,hi2lo:0});
var btnd = ll.gpiote(6, {type:"event",pin:BTN,lo2hi:0,hi2lo:1});
// A place to recieve Tasks - the RTC
var rtc = ll.rtc(2);
poke32(rtc.prescaler, 0); // no prescaler, 32 kHz
poke32(rtc.tStop, 1); // ensure RTC is stopped
// Set up and enable PPI to start and stop the RTC
ll.ppiEnable(0, btnd.eIn, rtc.tStart);
ll.ppiEnable(1, btnu.eIn, rtc.tStop);
// Every so often, check the RTC and report the result
setInterval(function() {
  print(peek32(rtc.counter));
  poke32(rtc.tClear, 1);
}, 5000);
```

### Hardware capacitive sense on two pins

Uses GPIO, counter timer:

**Note:** the counter timer has 6 capture/compare registers. We use 1 to produce the PWM
and 2 for the two capacitive sense pins - the remaining 3 could be used for 3 more
capacitive sense lines.

```JS
// connect one 100k resistor between PINDRV and PIN1
// and one 100k resistor between PINDRV and PIN2
function capSense2(PINDRV, PIN1, PIN2) {
  var ll = require("NRF52LL");
  digitalWrite(PINDRV,0);
  digitalRead([PIN1,PIN2]);
  // create a 'toggle' task for output
  var t0 = ll.gpiote(7, {type:"task",pin:PINDRV,lo2hi:1,hi2lo:1,initialState:0});
  // two input tasks, one for each cap sense input
  var e1 = ll.gpiote(6, {type:"event",pin:PIN1,lo2hi:1,hi2lo:0});
  var e2 = ll.gpiote(5, {type:"event",pin:PIN2,lo2hi:1,hi2lo:0});
  // create a timer that counts up to 1000 and back at full speed
  var tmr = ll.timer(3,{cc:[1000],cc0clear:1});
  // use a PPI to trigger toggle events
  ll.ppiEnable(0, tmr.eCompare[0], t0.tOut);
  // use 2 more to 'capture' the current timer value when a pin changes from low to high
  ll.ppiEnable(1, e1.eIn, tmr.tCapture[1]);
  ll.ppiEnable(2, e2.eIn, tmr.tCapture[2]);
  // Manually trigger a task to clear and start the timer
  poke32(tmr.cc[0],0); // compare with 0 for PWM
  poke32(tmr.tClear,1);
  poke32(tmr.tStart,1);
  return { read : function() {
    return [ peek32(tmr.cc[1]), peek32(tmr.cc[2]) ];
  } };
}

var cap = capSense2(D25, D31, D5);

setInterval(function() {
  console.log(cap.read());
},500);
```

Reference
---------

* APPEND_JSDOC: NRF52LL.js


Interrupts
----------

Espruino doesn't allow you to react to interrupts from the internal peripherals
directly, however you can change the state of an external pin (see the
examples above) and can then also use that as an input with `setWatch`.

'Use LED1 on Puck.js to sense a change in light level' above is a good example
of that.

**Note:** `setWatch` uses a **GPIOTE** peripheral for each watch, starting
with GPIOTE 0 - so be careful not to overlap them!


LPCOMP
------

LPCOMP is a low-power comparator. You can use it as follows:

```JS
var ll = require("NRF52LL");
// Compare D31 with 8/16 of vref (half voltage)
o = ll.lpcomp({pin:D31,vref:8});
// or {pin:D31,vref:D2} to compare with pin D2

// Read the current value of the comparator
console.log(o.sample());
// Return an object {up,down,cross} showing how
// the state changed since the last call
console.log(o.cross());
// eg { up: 1, down: 0, cross: 1 }
```

**Note:** As of Espruino 2v25 you can set up the comparator to create a JS event with [`E.setComparator`](https://www.espruino.com/Reference#l_E_setComparator)

```JS
E.setComparator(D31, 8/16);
E.on("comparator", e => {
  // e==1 -> up
  // e==-1 -> down
});
```
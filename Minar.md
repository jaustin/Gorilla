# Introduction to using MINAR

MINAR is a component in mbed OS for scheduling code to run later, either after a timeout or periodically. The act of scheduling code to be run later is called **posting** a callback.

Typically on embedded systems, you might use interrupts and timers to do this, but then your code runs in interrupt context and tasks are not serialised, adding complexities around locking and exclusion. Using MINAR, code runs in thread mode:

 * Cooperatively multi-tasking, MINAR is not an RTOS.
 * Tasks don’t interrupt each other.
 * Tasks run to completion.
 * Tasks can be posted from interrupt handlers and other tasks.


The starting point for scheduling code to run later is the `postCallback` function
```c
minar::Scheduler::postCallback(callbackTask);
```

## Run code some fixed time later

The following code example runs a function five seconds after initializing:

```c
#include "mbed-drivers/mbed.h"


static void blinky(void) {
    printf("5 second later!");
}   

void app_start(int, char**){
    printf("Hi");

    // Run a task in 5 seconds
    minar::Scheduler::postCallback(blinky).delay(minar::milliseconds(5000));
}   
```

Warning: MINAR is not an RTOS, so by default your task will not necessarily run at exactly the time you specified. Instead, it will run within a timewindow either side of the specified point in time. The size of this window is called *the tolerance* and can be specified in the same way as the delay.

By default the tolerance is 10ms:

```c
minar::Scheduler::postCallback(blinky)
	.delay(minar::milliseconds(5000)     // the postCallback method returns an object that lets us set more parameters
	.tolerance(minar::milliseconds(100)); // so now we can also set the tolerance
```

## Run code periodically

The following example, which is the same as the [mbed OS Blinky example](https://github.com/ARMmbed/example-mbedos-blinky), uses `minar` to run a callback periodically:

```c
#include "mbed-drivers/mbed.h"

// If we use the Scheduler class we simplify our code later on
using minar::Scheduler;

DigitalOut led(LED1);

static void blinky(void) {
    led = !led; 
}   

void app_start(int, char**){

    // we don't need minar:: here because of the 'using' line above'
    Scheduler::postCallback(blinky).period(minar::milliseconds(500));

}   

```

## Perform a task when a button is pressed

This example uses the InterruptIn class from mbed, but defers all the work to MINAR in thread mode:

```c
#include "mbed-drivers/mbed.h"

InterruptIn button(BUTTON1);
DigitalOut  led(LED1);

// If we use the Schedur class from the minar namespace we simplify our code later on
using minar::Scheduler;

void buttonTask() {
    printf("Hello mbed!");
    led = !led;
}

void buttonISR() {
    Scheduler::postCallback(buttonTask);
}

void app_start(int, char *[]) {
    button.fall(buttonISR);
}
```

## Cancel a callback

In order to cancel a callback that we've previously scheduled, we need a handle to it. We can do this by calling the getHandle() method on the object returned by postCallback. You can read the details of this implementation in [the MINAR docs](https://github.com/ARMmbed/minar).

```C++
#include "mbed-drivers/mbed.h"

using minar::Scheduler;

InterruptIn button(p17);
DigitalOut  led(p22);

static minar::callback_handle_t handle = 0;

static void blinky(void) {
    led = !led;
}

void buttonTask() {
    /* minar doesn't break if the handle is stale, so we can call this >once */
    Scheduler::cancelCallback(handle);
}

void buttonISR() {
    Scheduler::postCallback(buttonTask);
}

void app_start(int, char**){
    button.fall(buttonISR);
    handle = Scheduler::postCallback(blinky)
              .period(minar::milliseconds(500))
              .getHandle();
}
```

**Tip:** If you want all the dirt on MINAR, see [the mbed OS User Guide](https://docs.mbed.com/docs/getting-started-mbed-os/en/latest/Full_Guide/MINAR/).

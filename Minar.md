# Introduction to using MINAR

MINAR is a component in mbed OS for scheduling code to run later, either after a timeout or periodically. The act of scheduling code to be run later is called **posting** a callback.

Typically on embedded systems, you might use interrupts and timers to do this, but your code runs in interrupt context and tasks are not serialised, adding complexities around locking and exclusion. Using MINAR, code runs in thread mode.

 * Cooperatively multi-tasking, MINAR is not an RTOS
 * Tasks don’t interrupt each other
 * Tasks run to completion
 * Tasks can be posted from interrupt handlers and other tasks


The starting point for scheduling code to run later is the `postCallback` function
```c
minar::Scheduler::postCallback(callbackTask);
```

## Run code some fixed time later

The following code example will run a function after 5 seconds

```c
#include "mbed-drivers/mbed.h"


static void blinky(void) {
    printf("1 second later!");
}   

void app_start(int, char**){
    printf("Hi");

    // Run a task in 1 second
    minar::Scheduler::postCallback(blinky).delay(minar::milliseconds(5000));
}   
```

Warning: MINAR is not an RTOS, so by default your task will not necessarily run at exactly the time you specified: instead, it will run within a timewindow either side of the specified point in time. The size of this window is called the tolerance and can be specified in the same way as the delay.

By default the tolerance is 10ms

```c
minar::Scheduler::postCallback(blinky)
	.delay(minar::milliseconds(5000)     // the postCallback method returns an object that lets us set more parameters
	.tolerance(minar::milliseconds(100)); // so now we can also set the tolerance
```

## Run code periodically

The following example, which is the same as the mbed OS blinky, will use `minar` to run a callback periodically

```c
#include "mbed-drivers/mbed.h"

// If we use the Scheduler namespace we simplify our code later on
using minar::Scheduler;

DigitalOut led(LED1);

static void blinky(void) {
    led = !led; 
}   

void app_start(int, char**){

    // we don't need minar::Scheduler here because of the 'using' line above'
    postCallback(blinky).period(minar::milliseconds(500));

}   

```

## Perform a task when a button is pressed

This example uses the InterruptIn class from mbed, but defers all the work to MINAR in thread mode

```c
#include "mbed-drivers/mbed.h"

InterruptIn button(BUTTON1);
DigitalOut  led(LED1);

// If we use the Scheduler namespace we simplify our code later on
using minar::Scheduler;

void buttonTask() {
    printf("Hello mbed!");
    led = !led;
}

void buttonISR() {
    postCallback(buttonTask);
}

void app_start(int, char *[]) {
    button.fall(buttonISR);
}
```


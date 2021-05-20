# ResponsiveAnalogRead (for RP2040/Raspberry Pi Pico)

![ResponsiveAnalogRead](https://user-images.githubusercontent.com/345320/50956817-c4631a80-1510-11e9-806a-27583707ca91.jpg)

ResponsiveAnalogRead is an C++ library for eliminating noise in analogRead inputs without decreasing responsiveness. It sets out to achieve the following:

1. Be able to reduce large amounts of noise when reading a signal. So if a voltage is unchanging aside from noise, the values returned should never change due to noise alone.
2. Be extremely responsive (i.e. not sluggish) when the voltage changes quickly.
3. Have the option to be responsive when a voltage *stops* changing - when enabled the values returned must stop changing almost immediately after. When this option is enabled, a very small sacrifice in accuracy is permitted.
4. The returned values must avoid 'jumping' up several numbers at once, especially when the input signal changes very slowly. It's better to transition smoothly as long as that smooth transition is short.

You can preview the way the algorithm works with [sleep enabled](http://codepen.io/dxinteractive/pen/zBEbpP) (minimising the time spend transitioning between values) and with [sleep disabled](http://codepen.io/dxinteractive/pen/ezdJxL) (transitioning responsively and accurately but smooth).

An article discussing the design of the algorithm can be found [here](http://damienclarke.me/code/posts/writing-a-better-noise-reducing-analogread).

This port of the library makes it function on a Raspberry Pi Pico (and, likely, any RP2040-based board)

## How to use

Here's a basic example:

```cpp
#include "pico/stdlib.h"
#include "hardware/gpio.h"
#include "hardware/adc.h"
// include the ResponsiveAnalogRead library. Make sure you've copied the code into your program folder, and added it to CMakeLists.txt
#include <ResponsiveAnalogRead.h>

// define the ADC you want to use (0-3, which will be mapped to pins 26-29 on the Pico/RP2040)
const int ADC = 0;

// make a ResponsiveAnalogRead object, pass in the pin, and either true or false depending on if you want sleep enabled
// enabling sleep will cause values to take less time to stop changing and potentially stop changing more abruptly,
// where as disabling sleep will cause values to ease into their correct position smoothly and with slightly greater accuracy
ResponsiveAnalogRead analog(ADC, true);

// the next optional argument is snapMultiplier, which is set to 0.01 by default
// you can pass it a value from 0 to 1 that controls the amount of easing
// increase this to lessen the amount of easing (such as 0.1) and make the responsive values more responsive
// but doing so may cause more noise to seep through if sleep is not enabled


void main() {
  analog.setAnalogResolution(1 << 12); // 12 bit ADC on the RP2040
  analog.setActivityThreshold(16);     // ... so increase activity threshold to match to 1 << 12-8

  while (true) {
    analog.update();
    if(analog.hasChanged()) {
      int value = analog.getValue();
      // ... do something with value
    }
  }
}
```

### Using your own ADC

```cpp
#include "pico/stdlib.h"
#include "hardware/gpio.h"
#include "hardware/adc.h
#include <ResponsiveAnalogRead.h>

ResponsiveAnalogRead analog(0, true);

void main() {
  adc_gpio_init(26)

  analog.setAnalogResolution(1 << 12); // 12 bit ADC on the RP2040
  analog.setActivityThreshold(16);     // ... so increase activity threshold to match to 1 << 12-8

  while(true) {
    // read from your ADC
    adc_select_input(adc);
    int reading = adc_read();
    // update the ResponsiveAnalogRead object every loop
    analog.update(reading);

    if(analog.hasChanged()) {
      // .. do something with it
    }
  }
}
```

### Smoothing multiple inputs

```cpp
#include "pico/stdlib.h"
#include "hardware/gpio.h"
#include "hardware/adc.h
#include <ResponsiveAnalogRead.h>

ResponsiveAnalogRead analogOne(0, true);
ResponsiveAnalogRead analogTwo(1, true);

void main() {
  analog.setAnalogResolution(1 << 12); // 12 bit ADC on the RP2040
  analog.setActivityThreshold(16);     // ... so increase activity threshold to match to 1 << 12-8

  while(true) {
    // update the ResponsiveAnalogRead objects every loop
    analogOne.update();
    analogTwo.update();
    // ...do something with their new values
  }
  
}
```

## How to install

* Copy the contents of `src` into your CPP program.
* Add ResponsiveAnalogRead.cpp to the list of files in the `add_executable` directive of the CMake file
* Include `ResponsiveAnalogRead.h` as necessary

## Constructor arguments

- `adc` - int, the adc to read (0-3, which are physical pins 26-29 respectively on a Pi Pico).
- `sleepEnable` - boolean, sets whether sleep is enabled. Defaults to true. Enabling sleep will cause values to take less time to stop changing and potentially stop changing more abruptly, where as disabling sleep will cause values to ease into their correct position smoothly.
- `snapMultiplier` - float, a value from 0 to 1 that controls the amount of easing. Defaults to 0.01. Increase this to lessen the amount of easing (such as 0.1) and make the responsive values more responsive, but doing so may cause more noise to seep through if sleep is not enabled.

## Basic methods

- `int getValue() // get the responsive value from last update`
- `int getRawValue() // get the raw analogRead() value from last update`
- `bool hasChanged() // returns true if the responsive value has changed during the last update`
- `void update(); // updates the value by performing an analogRead() and calculating a responsive value based off it`
- `void update(int rawValue); // updates the value by accepting a raw value and calculating a responsive value based off it (version 1.1.0+)`
- `bool isSleeping() // returns true if the algorithm is in sleep mode (version 1.1.0+)`

## Other methods (settings)

### Sleep

- `void enableSleep()`
- `void disableSleep()`

Sleep allows you to minimise the amount of responsive value changes over time. Increasingly small changes in the output value to be ignored, so instead of having the responsiveValue slide into position over a couple of seconds, it stops when it's "close enough". It's enabled by default. Here's a summary of how it works:

1. "Sleep" is when the output value decides to ignore increasingly small changes.
2. When it sleeps, it is less likely to start moving again, but a large enough nudge will wake it up and begin responding as normal.
3. It classifies changes in the input voltage as being "active" or not. A lack of activity tells it to sleep.

### Activity threshold
- `void setActivityThreshold(float newThreshold) // the amount of movement that must take place for it to register as activity and start moving the output value. Defaults to 4.0. (version 1.1+)`

### Snap multiplier
- `void setSnapMultiplier(float newMultiplier)`

SnapMultiplier is a value from 0 to 1 that controls the amount of easing. Increase this to lessen the amount of easing (such as 0.1) and make the responsive values more responsive, but doing so may cause more noise to seep through when sleep is not enabled.

### Edge snapping
- `void enableEdgeSnap() // edge snap ensures that values at the edges of the spectrum (0 and 1023) can be easily reached when sleep is enabled`

### Analog resolution
- `void setAnalogResolution(int resolution)`

If your ADC is something other than 10bit (1024), set that using this.

## License

Licensed under the MIT License (MIT)

Copyright (c) 2016, Damien Clarke, 2021 Tom Armitage

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

# Sound-Reactive NeoPixel LED Strip

An Arduino project that reads an analogue sensor signal and uses the
readings to control a 60-pixel NeoPixel LED strip. The sketch counts how
many of 100 sensor samples cross a threshold, adjusts the number of
illuminated pixels, and selects a colour using a colour-wheel function.

> **Note:** This README describes the supplied sketch as written. A few
> details in the code should be checked before using it on hardware; see
> [Important code notes](#important-code-notes).

## Project overview

The Arduino repeatedly:

1.  Reads the sensor connected to analogue input `A0` 100 times.
2.  Counts how many readings are greater than `100`.
3.  Uses that count to update `outputValue`, which represents the
    intended number of illuminated LEDs.
4.  Occasionally generates a random colour position.
5.  Clears the pixel buffer, sets the selected pixels to the colour
    returned by `Wheel()`, and sends the update to the strip.

The behaviour is sound-responsive if the sensor connected to `A0` is a
microphone or sound sensor. The sketch itself only reads an analogue
signal, so another analogue sensor could also be used.

## Hardware

The code indicates the following setup:

-   Arduino board with an analogue input `A0`
-   60-pixel addressable NeoPixel-compatible LED strip
-   Analogue sensor connected to `A0`
-   LED strip data input connected to digital pin `6`
-   Suitable power supply for the LED strip

### Connections

  Component                Arduino connection
  ------------------------ ------------------------------------------------
  Sensor analogue output   `A0`
  LED strip data input     Digital pin `6`
  Sensor ground            Arduino GND
  LED strip ground         Arduino GND / common ground
  LED strip power          Suitable external supply, matched to the strip

Connect the Arduino ground and LED power-supply ground together. Do not
assume the Arduino 5 V pin can safely power all 60 LEDs; use a suitable
supply for the strip and follow the strip manufacturer's wiring
guidance.

## Software requirements

-   Arduino IDE
-   `Adafruit NeoPixel` library

Install the library through **Arduino IDE → Library Manager**:

1.  Open Library Manager.
2.  Search for **Adafruit NeoPixel**.
3.  Install the library.
4.  Open `pixel_vu_meter_original.ino`.
5.  Select the correct board and port.
6.  Compile and upload the sketch.

## Configuration

The main hardware settings are defined near the top of the sketch:

``` cpp
#define PIN 6
#define NUMPIXELS 60
```

-   `PIN` is the Arduino data pin connected to the LED strip.
-   `NUMPIXELS` is the number of LEDs in the strip.

The strip is initialised with:

``` cpp
Adafruit_NeoPixel pixels =
  Adafruit_NeoPixel(NUMPIXELS, PIN, NEO_GRB + NEO_KHZ800);
```

This configures a GRB colour order and an 800 kHz signalling rate. Check
the LED strip's required colour order and protocol if its colours appear
incorrect.

## How the code works

### 1. Initialisation --- `setup()`

-   `pixels.begin()` initialises the NeoPixel strip.
-   `randomSeed(analogRead(0))` seeds the random-number generator using
    an analogue reading.
-   `Serial.begin(9600)` starts serial communication at 9600 baud. The
    current sketch does not print any serial messages.

### 2. Sensor sampling --- `loop()`

The sketch takes 100 readings from `A0`:

``` cpp
for (int i = 0; i < 100; i++) {
  sensorValue = analogRead(A0);
  if (sensorValue > 100) counter++;
}
```

Each reading greater than `100` increments `counter`. Therefore,
`counter` can range from `0` to `100`.

### 3. LED count adjustment

`outputValue` is used as the intended number of illuminated pixels. The
sketch attempts to increase it based on the sensor count and gradually
decrease it on selected loop iterations. It then limits the value to the
range `0`--`60`.

### 4. Colour selection --- `Wheel()`

`Wheel()` accepts a byte-sized position and returns a colour value. Its
three sections blend red, green, and blue components to create a
colour-wheel effect.

A random position is generated when `loopCounter` is divisible by `100`:

``` cpp
if (loopCounter % 100 == 0) randNumber = random(255);
```

### 5. Updating the strip

The sketch first sets every pixel in the software buffer to a dark
value. If `outputValue` differs from `rememberOutputValue`, it sets the
selected pixels to the colour returned by `Wheel()` and calls:

``` cpp
pixels.show();
```

`pixels.show()` transmits the buffered pixel colours to the physical LED
strip.

## Important code notes

Review these points in the original sketch before relying on it:

1.  **Pixel index is out of range:** The loop starts at `i = 60`, but a
    60-pixel strip has valid indices `0` through `59`. The first
    `setPixelColor()` call therefore uses an invalid index. A typical
    corrected loop for lighting the last `outputValue` pixels would
    start at `59` and continue while `i >= 60 - outputValue`.

2.  **The first `map()` call always returns 80:**\
    `map(counter, 10, 60, 80, 80)` has identical output limits, so its
    result is always `80`. The condition is effectively
    `80 > outputValue`; it does not scale the sensor count.

3.  **The second `map()` is effectively an identity mapping:**\
    `map(counter, 0, 40, 0, 40)` maps the range onto itself. Values
    outside the stated input range can also produce values outside
    `0`--`40`, before the later clamp to `0`--`60`.

4.  **The four-argument colour calls need checking:**\
    The strip is configured as `NEO_GRB`, which describes a
    three-channel RGB strip. The sketch calls `pixels.Color()` with four
    arguments. Check the installed Adafruit NeoPixel library version and
    the actual strip type; for a standard RGB strip, three colour
    components are normally used.

5.  **Colour changes may not be shown immediately:**\
    `pixels.show()` is called only when `outputValue` changes. A new
    random colour may be selected while the LED count remains unchanged,
    in which case the new colour is not sent to the strip at that
    moment.

6.  **The random range is 0--254:**\
    `random(255)` does not generate `255`. Use `random(256)` if the full
    byte range `0`--`255` is wanted.

7.  **The special `outputValue == 20` loop condition is unusual:**\
    The condition `(outputValue == 20 && i == 0)` is not needed for the
    normal last-N-pixels pattern and does not correct the invalid
    starting index.

These are observations about the supplied source, not changes made to
it.

## Main variables

  -----------------------------------------------------------------------
  Variable                            Purpose
  ----------------------------------- -----------------------------------
  `outputValue`                       Intended number of LEDs to
                                      illuminate

  `rememberOutputValue`               Previous output value, used to
                                      detect changes

  `randNumber`                        Random colour-wheel position

  `counter`                           Number of the 100 sensor samples
                                      greater than `100`

  `loopCounter`                       Counts executions of `loop()`
  -----------------------------------------------------------------------

## Repository contents

``` text
.
├── pixel_vu_meter_original.ino
└── README.md
```

## Possible future improvements

-   Correct the LED index so it stays within `0`--`59`.
-   Choose meaningful mapping limits for sensor count and LED count.
-   Add serial output to help observe sensor readings while testing.
-   Make the threshold configurable.
-   Update the strip when the colour changes, even if the LED count
    stays the same.
-   Add a short delay or a defined sampling interval if a steadier
    visual response is needed.

## License

No license information was included with the supplied sketch. Add a
license before redistributing the project if you are the owner or have
permission to do so.

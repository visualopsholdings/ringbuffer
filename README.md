
# Using I2C to control an Arduino

## Context

Read: https://github.com/visualopsholdings/i2c

### Ring buffer

A Ring Buffer can be written to AND read from at the same time without the reader and
writer sharing any thing between them apart from the actual buffer.

As long as the reader keeps up with the writer, as the writer get's to the end and wraps around
the reader can happily consume the buffer.

The C++ class in ringbuffer.hpp and ringbuffer.cpp achieves this. To use it, just
include the header and then declare one and use it.

Now your code above can look like this:

```
RingBuffer buffer;

void setup() {
  pinMode(7, OUTPUT);
  digitalWrite(7, LOW);
  Wire.begin(8);
  Wire.onReceive(receiveEvent);
}
void receiveEvent(int howMany) {
  while (Wire.available()) {
    buffer.write(Wire.read());
  }
}
void loop() {
  if (buffer.length() > 0) {
    if (buffer.read() == 1) {
      digitalWrite(7, HIGH);
    }
    else {
      digitalWrite(7, LOW);
    }
  }
}
```

OK! So now we can send various I2C words and it will happily do all the different
things sending a 1 or a 0 can do to an LED :-)

## Sending I2C from a PI

To actually send i2c commands to your Arduino, you can use a raspberry PI and then wire
up the various pins between your Arduino and the PI like this:

Power:
- 5v pin on the PI to the 5v pin on your Arduino
- GRound pin on the PI to (you guessed it) the ground pin on the Arduino

I2C:
- Wire the SDA and SCL ins together

On a PI, all of these pins are right up the end AWAY from the USB/Ethernet ports in a tight
cluster so easy to make a simple little 4 pin jumper for that end.

Looking down at the PI header it's this:

| <!-- --> | <!-- --> | <!-- -->  | <!-- --> |
| ------ | ------ | ------ | ------ |
|        | SCL    | SDA    |        |
|        | GROUND | VIN    |        |
| <!-- --> | <!-- --> | <!-- -->  | <!-- --> |

And on an Arduino, all the pins are usually labelled so that's easy, but there are 2 sets
of pins usually. The SDA and SCL are right at the end of the long header and the power 
is at the start of the second short header on an Arduino.

While you program your Arduino, unplug the power from the PI to the Arduino while it's
connected to your computer and then unplug your Arduino from your computer and plug it
into the PI and yoru good to go.

On the PI, you can do this:

```
$ sudo apt-get install -y i2c-tools
```

And then you can see your Arduino with:

```
$ i2cdetect -y 1
```

It draws a nice graphical map of the i2c bus, yours will be number 8 on the first one.

To send it an "ON;"

```
$ i2ctransfer -y 1 w3@0x08 0x4F 0x4E 0x3B
```

"OFF;"
```
$ i2ctransfer -y 1 w4@0x08 0x4F 0x46 0x46 0x3B
```

"GO;"
```
$ i2ctransfer -y 1 w3@0x08 0x47 0x4F 0x3B
```

You can work out those codes in here:

https://www.rapidtables.com/convert/number/ascii-to-hex.html

## Long delay() considered harmful

For simplicity I just used delays between led flashes (sort of to prove my point), but
this is considered bad practice on an arduino since long delays can stop anything else
happening.

Here is the correct code you would use to flash an LED with varying delays between on and 
off.

```
void setup() {
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
}

int state = 0;
int lasttime = 0;

void loop() {
  if (!lasttime) {
    lasttime = millis();
    return;
  }
  int now = millis();
  int diff = now - lasttime;
  if (state == 0 && diff > 1000) {
    ledOn();
    state = 1;
  }
  else if (state == 1 && diff > 100) {
    ledOff();
    state = 0;
  }
  else {
    return;
  }
  lasttime = millis();
}

```

## Using the examples

Each example is named appropriately for what it does, and to get them all to build, put this
whole i2c folder into the "libraries" folder in your Arduion Sketch Directory. You can find
this in Preferences, or Settings on the macintosh.

## Development

The development process for all of this code used a normal Linux environment with the BOOST
libraries and a C++ compiler to write and run unit tests to test all of the various
things and then use that code in the Arduino IDE to flash the Arduino.

It's written in a subset of C++ that happily builds on an Arduino:

  - No dynamic memory (no new, malloc etc. Everything just static)
  - No inheritence (see above)
  - No STL (see above)

BUT for the actual development, including a fairly complex debugger for the LOGO stuff,
we use all of these things and just don't build that bit for the Arduino.

So on your Linux (or mac using Homebrew etc), get all you need:

```
$ sudo apt-get -y install g++ gcc make cmake boost
```

And then inside the "test" folder on your machine, run:

```
$ mkdir build
$ cd build
$ cmake ..
$ make
$ make test
```

And then you can go in and fiddle and change the code in lgtestsketch.cpp to change what
your sketch might look like on in your .ino and then do

```
$ make && ./RBTest
```

And it will just run your test in your Dev environment! Just stub out all your builtin words
like "ledOn()" etc to output a string.

For an M1 or M2 mac, use this command for cmake

```
$ cmake -DCMAKE_OSX_ARCHITECTURES="arm64" ..
```

To turn on all the debugging for the various things, each header has something like:

```
//#define RINGBUF_DEBUG
```

That you can comment out BUT while it's commented out the code won't build on the Arduino.

This stuff is just for the development environment.
  
## Related discussions

https://forum.arduino.cc/t/how-to-properly-use-wire-onreceive/891195/12

## Change Log

24 May 2023
- Moved ringbuffer to it's own repo
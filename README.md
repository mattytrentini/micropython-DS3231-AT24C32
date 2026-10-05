# micropython-DS3231-AT24C32
MicroPython driver for the DS3231 RTC found on DS3231/AT24C32 modules.

This repository does not implement an AT24C32 EEPROM driver.

## DS3231N
Driver for the Dallas DS3231, a highly accurate RTC IC.

I wrote this driver to make an easier to use and complete driver to access all the functions the DS3231 has. The driver has been designed for use with an ESP8266 but should work on fine on other devices as long as an I2C instance can be given.

Some care has been put into memory use, with most variables buffered when constructing. Checking the alarms and setting the SQW/INT doesn't use memory allocation in the heap and can be used in ISR. This does not imply methods are thread safe.

This driver implements both alarms and all available match triggers, interrupts and checks.

Oscillator Stop Flag (OSF) is checked when reading the time, with a warning printed if it is set. Check `ds.OSF()` before trusting the clock: `True` means the oscillator stopped, or has stopped in the past, and the time may be invalid.

Square wave output and also the crystal output can be set.

The temperature readout and aging offset options have not been implemented yet. I see no priority to implement temperature data since I can't come up with a plausible use case, but requests are welcome. As soon as I get access to a decent oscilloscope I will start playing with the aging offset features.

## Installation

Install the driver from the `mattytrentini` fork using `mpremote` on your host:

```sh
mpremote connect /dev/ttyACM0 mip install github:mattytrentini/micropython-DS3231-AT24C32
```

Replace `/dev/ttyACM0` with your board's serial port. Alternatively, on a
network-connected MicroPython board with `mip` available:

```python
import mip
mip.install("github:mattytrentini/micropython-DS3231-AT24C32")
```

The package installs only `ds3231.py`, has no external dependencies, and is
imported with `from ds3231 import DS3231`. The driver retains its upstream API
and runtime behavior.

## Usage

### Create instance
Create an instance of the DS3231 class:

```python
from machine import Pin, I2C
from ds3231 import DS3231

i2c = I2C(sda=Pin(4), scl=Pin(5))

ds = DS3231(i2c)
```

For a KinCony CO16, use I2C0 with SDA GPIO8, SCL GPIO18, and the RTC at `0x68`:

```python
from machine import Pin, I2C
from ds3231 import DS3231

i2c = I2C(0, sda=Pin(8), scl=Pin(18), freq=100_000)
ds = DS3231(i2c, addr=0x68)
if ds.OSF():
    print("RTC time is not trustworthy; set it from a known-good source.")
else:
    print(ds.datetime())
```

This example only reads the RTC; it does not set the clock or clear its status.
### Time functions

Example of setting the date and time:
```python
year = 2020 # Can be yyyy or yy format
month = 10
mday = 3
hour = 13 # 24 hour format only
minute = 55
second = 30 # Optional
weekday = 6 # Optional

datetime = (year, month, mday, hour, minute, second, weekday)
ds.datetime(datetime)
```

The getter `ds.datetime()` returns an eight-element tuple in ESP8266 RTC order:
`(2020, 10, 3, 6, 13, 55, 30, 0) # (year, month, mday, weekday, hour, minute, second, 0)`

**The setter and getter use different tuple orders.** The setter accepts
`(year, month, mday, hour, minute[, second[, weekday]])`, not the getter's
tuple. Do not pass a getter result directly back to `ds.datetime(...)`.
Omitted seconds and weekday are written as zero; an explicitly supplied
weekday follows the DS3231's 1–7 convention. Setting the clock clears OSF,
so first check `ds.OSF()` and use a known-good time source when setting it.

Set the ESP8266 internal RTC using the DS3231
```python
import machine
rtc = machine.RTC()
if not ds.OSF():
    rtc.datetime(ds.datetime())
else:
    print("RTC time is not trustworthy; internal RTC left unchanged.")
```

#### Get the date and time
Call `ds.datetime()` to get the current date and time. It prints a warning when
the Oscillator Stop Flag (OSF) is set, but still returns a tuple; that tuple
must not be treated as reliable solely because it was returned. Read
`ds.OSF()` before using the time, including before copying it to the internal
RTC. Reading OSF or the date/time does not clear OSF.

### Alarm functions

The DS3231 has 2 internal alarms with can each be set independently to different match conditions.
The alarm has match options ranging from every second to every month.

Call an alarm without arguments and it will return the current alarm setting register.

#### alarm 1
Set alarm 1:
`DS3231.alarm1([time[, match[, int_en[, weekday]]]])`

time    : tuple, (second,[ minute[, hour[, day]]])
weekday : bool, select mday (False) or wday (True)
match   : int, match const
int_en  : bool, enable interrupt on alarm match on SQW/INT pin (disables SQW output)


Alarm 1 has the following match options:
```python
ds.AL1_EVERY_S      # Alarm every second
ds.AL1_MATCH_S      # Alarm when seconds match (every minute)
ds.AL1_MATCH_MS     # Alarm when minutes, seconds match (every hour)
ds.AL1_MATCH_HMS    # Alarm when hours, minutes, seconds match (every day)
ds.AL1_MATCH_DHMS   # Alarm when day|wday, hour, min, sec match (specific wday / mday) (once per month/week)
```

Example:
```python
# Set alarm 1 for 16:10:15 every day
ds.alarm1((15, 10, 16), match=ds.AL1_MATCH_HMS)
```

#### alarm 2
Set alarm 2:
`DS3231.alarm2([time[, match[, int_en[, weekday]]]])`

time    : tuple, (minute[, hour[, day]])
weekday : bool, select mday (False) or wday (True)
match   : int, match const
int_en  : bool, enable interrupt on alarm match on SQW/INT pin (disables SQW output)


Alarm 2 has the following match options:
```python
ds.AL2_EVERY_M # Alarm every minute on 00 seconds
ds.AL2_MATCH_M # Alarm when minutes match (every hour)
ds.AL2_MATCH_HM # Alarm when hours and minutes match (every day)
ds.AL2_MATCH_DHM # Alarm when day|wday match (once per month/week)
```

Example:
```python
# Set alarm 1 for 19:30 every Tuesday
ds.alarm2((30, 19, 2), match=ds.AL2_MATCH_DHM, weekday=True)
```

#### Checking and clearing alarms
You can manually check to see if an alarm has triggered. This will return True when the alarm has been triggered and False otherwise. The act of checking clears the alarm register automatically.

`DS3231.check_alarm(alarm)`

alarm : alarm number, can be 1 or 2

### Square wave output
The DS3231 can output a square wave on the SQW/INT output pin. The following frequencies are availalbe:
* `FREQ_1`       1 Hz
* `FREQ_1024` 1024 Hz
* `FREQ_4096` 4096 Hz
* `FREQ_8192` 8192 Hz

Example:
```python
ds.square_wave(freq=ds.FREQ_1024) # Enable 1024 Hz output
ds.square_wave(freq=False) # Disable SQW output
```

Note:
* When using the SQW/INT pin to output a square wave, the alarm interrupt is not avaialable. Checking for alarms will have to be done manually.
* When disabling the SQW output signal, the alarm interrupts (even when set before) remain disabled. They can to be set manually afterwards, if desired with `ds.alarm_int()`.

The DS3231 can also output the crystal frequency at 32768 Hz on a dedicated pin (32K). This output is enabled by default on powerup and can me changed as follows:

```python
ds.output_32kHz() # Enable 32 kHz output
ds.output_32kHz(False) # Disable 32 kHz output
```

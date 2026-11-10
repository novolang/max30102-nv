# max30102-nv

A driver for the MAX30102, a pulse oximeter and heart-rate sensor from
Maxim Integrated (now Analog Devices) with an I2C interface, specified in
the data sheet "MAX30102: High-Sensitivity Pulse Oximeter and Heart-Rate
Sensor for Wearable Health", document 19-7740, Rev 1, 10/18
([analog.com/en/products/max30102](https://www.analog.com/en/products/max30102)).
The driver implements the `HeartRate` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv). It delivers the
raw red and infrared counts of the sensor's FIFO as its primary
interface. The package also holds a simple beat detector over those
counts, `ppg_beat`, which is a starting point and not a medical
algorithm.

**Hardware verification: NOT VERIFIED.** No board on the bench carries
this sensor, so the driver has run only against a double of the part, in
the host suite, and the beat detector only over synthesised waveforms. The
device program below builds for the nRF52840 DK and the nRF54LM20 DK and
has not been flashed.

## Install

```sh
novo pkg add max30102-nv
```

The device program is `examples/pulse_reader.nv`.  Make it the `main` of a package that
depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/pulse_reader.nv` brings the sensor up, drains its FIFO ten times
a second, feeds the infrared counts to the beat detector and prints the
newest counts and the rate once a second over the board's UART:

```
ir 78234 red 65012 bpm 72
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio, BoardTimer }
use dev_ppg.{ PpgSample, HeartRate }
use dev_bus
use max30102.{ Max30102 }
use ppg_beat

// Writes `n`, 0 or more, in decimal.  Formatting into a string would
// allocate, which @tier(embedded) refuses, so each digit goes out as a
// byte.  An Int has at most 19 decimal digits, which bounds both loops.
@tier(embedded)
fn digits(n: Int) [hw]
    var div = 1
    for _k in 0..18
        if div <= n / 10
            div = div * 10
    for _k in 0..19
        if div > 0
            hal.uart.write_byte(48 + n / div % 10)
            div = div / 10

// Writes the line for the newest sample and the detector.
@tier(embedded)
fn report(s: PpgSample, bpm: Int) [hw]
    hal.uart.write("ir ")
    digits(s.ir)
    hal.uart.write(" red ")
    digits(s.red)
    hal.uart.write(" bpm ")
    digits(bpm)
    hal.uart.write("\n")

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    let sensor = Max30102.attach(BoardI2c { id: 0 }, BoardTimer { id: 0 }, BoardGpio { id: 0 },
                                 dev_bus.wiring_i2c(0, 0x57))
    // The sensor needs its supplies up; the program waits 20 ms.
    hal.timer.delay_ms(20)
    if not sensor.begin()
        hal.uart.write("no answer from the sensor at 0x57\n")
    // 100 samples a second reach the detector (400 sps averaged by 4).
    let cfg = ppg_beat.beat_config_default()
    var det = ppg_beat.beat_new()
    var newest = PpgSample { red: 0, ir: 0, green: None, at_us: 0 }
    var round = 0
    while true
        var out: Vec[PpgSample; 32] = Vec.new()
        let n = sensor.ppg_drain(var out)
        if n < 0
            hal.uart.write("no answer\n")
        else
            for i in 0..n
                det = ppg_beat.beat_feed(det, ppg_beat.signal_of(out[i]), cfg)
                newest = out[i]
        round = round + 1
        if round >= 10
            round = 0
            report(newest, det.bpm)
        hal.timer.delay_ms(100)
```

The program is `examples/pulse_reader.nv`. A board package that depends
on this package builds it as its `main`. `tests/device_build.sh` does so
from a scratch package for the two boards below and for the emulated
board, where it runs the image and checks that the program reports that
the sensor does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 18,312 B | 5,706 B | 2,644 B |
| `nrf54lm20-dk` | 19,608 B | 6,002 B | 2,644 B |

The numbers are from the build's memory report. Flash also holds the
runtime, the compiler's support library and the program's tables. The
arena, 65,536 B on both boards, is the runtime's heap and is reserved
whether or not the program uses it; the driver allocates nothing in it.
The driver's whole state is a `Max30102` value of 96 bytes, the detector
a `Beat` value of 112 bytes, and the 32-sample buffer `Vec[PpgSample; 32]`
1,288 bytes, all in the program's frame.

Neither Nordic board package names a driver for its two-wire interface
(`@hal(i2c)`), so `BoardI2c` is absent hardware on both: every transfer is
refused, the program prints `no answer from the sensor at 0x57`, and the
build removes the code that follows the first bus read, which makes the
rows above smaller than the image of a board with a bus.

The wiring: SDA and SCL to the board's first I2C bus, the breakout's
supplies to 3.3 V (the sensor itself runs its core at 1.8 V and its LEDs
at 3.1 V to 5 V, which the breakouts regulate), and INT either open or on
a free pin of the board, which the wiring record names.

### The effects

Every function of the driver that reaches the bus, a pin or the timer
declares `[hw]`, and nothing else. The driver performs no input or output
of its own and allocates nothing of its own. The bus functions of
devices-nv build one short frame per transaction inside a `system` block
and release it when the call returns. The module checks at every tier,
`embedded` included. The beat detector is integer arithmetic on values and
declares no effect.

## What the sensor is

The MAX30102 is a reflective optical module: a red LED (660 nm), an
infrared LED (880 nm) and a photodiode behind an 18-bit ADC. The light of
the LEDs enters the skin of a fingertip; each heartbeat pushes blood
through the tissue, which absorbs more light, so the count at the
photodiode rises and falls with the pulse. That signal is a
**photoplethysmogram** (PPG). The same count at two wavelengths also gives
blood oxygen, which this package does not compute.

The sensor answers on the I2C address 0x57 (the sheet writes 0xAE and
0xAF). PART_ID, register 0xFF, reads 0x15; REV_ID, 0xFE, is a number the
maker assigns.

| Setting | Values | Register |
|---|---|---|
| Mode | heart rate (red only), SpO2 (red and infrared), multi-LED | MODE_CONFIG 0x09 |
| LED pulse amplitude | 0 to 51 mA in steps of 0.2 mA, per LED | LED1_PA 0x0C (red), LED2_PA 0x0D (infrared) |
| Pulse width | 69, 118, 215 or 411 µs, giving 15, 16, 17 or 18 bits | SPO2_CONFIG 0x0A |
| ADC full scale | 2048, 4096, 8192 or 16384 nA | SPO2_CONFIG 0x0A |
| Sample rate | 50 to 3200 sps, limited by the pulse width | SPO2_CONFIG 0x0A |
| Averaging | 1, 2, 4, 8, 16 or 32 samples per FIFO sample | FIFO_CONFIG 0x08 |
| FIFO roll-over | off, or on | FIFO_CONFIG 0x08 |
| Almost-full level | the empty slots left when the interrupt fires, 0 to 15 | FIFO_CONFIG 0x08 |

The sample rate is limited by the pulse width, as Tables 11 and 12 give:

| Pulse width | 69 µs | 118 µs | 215 µs | 411 µs |
|---|---|---|---|---|
| SpO2 mode, at most | 1600 sps | 1000 sps | 800 sps | 400 sps |
| Heart rate mode, at most | 3200 sps | 1600 sps | 1600 sps | 1000 sps |

The part programs the highest rate allowed when a higher one is asked
for. The driver clamps first and reads the register back, so the rate it
answers is the part's. With averaging the FIFO fills at the ADC rate over
the averaging: the default configuration runs the ADC at 400 sps with
averaging 4, and the FIFO fills at 100 sps.

The FIFO holds 32 samples. A sample is one 3-byte count per active
channel, red first. The 18-bit ADC word is **left-justified** in the 24
bits (Table 1): the most significant ADC bit is always bit 17, so a 15-bit
conversion leaves bits 2:0 at zero. The driver shifts the word right by
the missing bits, as Zephyr's driver and the `max3010x` crate do, so a
count is the ADC's own value, 15 to 18 bits. Counts at two pulse widths
are therefore not on the same scale.

## What the package contains

| Module | What is in it |
|---|---|
| `max30102` | The type `Max30102` that holds a board's bus, timer and pins, the trait `Max30102Port`, the configuration `Max30102Config`, the register conversions, the FIFO drain, the temperature and the bring-up |
| `ppg_beat` | The beat detector `Beat` with `beat_feed` for one count and `beat_run` for a `HeartRate` |

## How to choose an entry point

A program on a board uses `Max30102`, built by `attach` with the board's
I2C bus, timer and GPIO and the wiring, as a board's device table builds
it. The value holds no state, so every method takes a plain `self`, and a
function written against `HeartRate` can call them on a `let`.

| Method | What it does |
|---|---|
| `attach(bus, timer, pins, w)` | the part at `w.addr`; nothing is sent |
| `begin()`, `begin_with(c)` | brings the part up with the default configuration or one of the caller's |
| `ppg_drain(var out)` | `HeartRate`: replaces `out` with the samples waiting, oldest first, red and infrared counts and a stamp; 0 for an empty FIFO, -1 when the part did not answer |
| `ppg_set_rate_hz(hz)` | `HeartRate`: the least FIFO rate at or above `hz` that the pulse width allows, with the averaging kept; the rate the part took |
| `ppg_set_led_ma(led, ma)` | `HeartRate`: the largest step at or below `ma`, never above it; the current set, 0 for green |
| `ppg_lost()` | `HeartRate`: the overflow counter |
| `device_id()` | `Identifiable`: 0x15, with REV_ID as `rev` |
| `fifo_count()`, `clear_fifo()`, `set_almost_full(n)` | the samples waiting, an empty FIFO, the almost-full level |
| `interrupts_enable(mask)`, `interrupts_read()` | enable `INT_A_FULL`, `INT_PPG_RDY`, `INT_ALC_OVF` and `INT_DIE_TEMP`; read and clear the flags |
| `temperature_milli_c()` | one die temperature reading, thousandths of a degree Celsius |
| `shutdown(on)` | power-save mode, with the registers kept |
| `reg(r)`, `set_reg(r, v)` | one register |

A program that holds its own bus, or a test with a double of the part,
implements `Max30102Port` over its bus and calls the module's functions,
which take the port: `max30102.start(port, config)`,
`max30102.drain(port, var out)`.

### Where the driver differs from the trait's description

The sheet's FIFO carries no time stamps, and the overflow counter clears
when a sample is read. `HeartRate` describes `at_us` as the time since
sampling started and `ppg_lost` as the samples lost since then, which a
driver can only supply by keeping state between calls, and a driver that
changes its own state takes `var self`, which a generic function can
call only through a `var` parameter. This driver keeps no state, so:

- `at_us` of the newest sample is the board timer's tick at the drain, and
  each older sample is one FIFO period before the next, so the stamps are
  evenly spaced, within one period of the instant each sample was taken,
  and continue across drains on the timer's clock, not from zero.
- `ppg_lost` is the overflow counter, the samples lost since the FIFO
  was last read, saturating at 31.

A program that needs the total keeps it from successive `ppg_lost` calls
before each drain.

## The beat detector

`ppg_beat` finds the rhythm of a still fingertip on a good signal. It
does not reject motion artefacts, does not tell a heartbeat from a
tremor, and has not been tried on a wearer. **It is a starting point.** A
product that reports a heart rate to a person needs an algorithm of its
own, tuned and validated for its sensor, its LED current and its use. The
algorithm, in integers, one sample at a time:

1. A count below `min_signal` means no finger, and the detector resets.
2. The baseline is an exponential average of the counts with a time
   constant of 1.5 s; the pulse is the count less the baseline, negated
   when `invert` is set (a reflective sensor on a finger sees the light fall
   with each pulse of blood).
3. The pulse is smoothed over 4 samples.
4. An envelope follows the peaks of the smoothed pulse and decays with a
   time constant of 2 s.
5. A beat is the smoothed pulse rising through half the envelope, after it
   has fallen below a quarter of it, and not within 250 ms of the last beat.
6. The interval since the last beat is kept when it is between 30 and 220
   beats a minute. The rate is 60 × fs over the mean of the last four
   intervals, from the second interval on. An interval outside the range,
   or three seconds without a beat, clears them and the rate goes to 0.

`beat_feed(d, count, config)` takes one count and answers the next
detector. `beat_run(h, d, config)` drains any `HeartRate` and feeds every
sample to the detector, with the infrared count, or the red one for a part
in heart rate mode. The tests run it over sines from 45 to 180 beats a
minute, with noise, with a dip in place of a peak, with a second hump in the
pulse, over a change of tempo, a finger that leaves and comes back, and a
flat signal. They check that it finds the rate on those signals. They say
nothing about a wearer.

## The rules a user needs

1. Reset the part before configuring it. `start` does: RESET is set, and
   the driver waits for the part to clear it, at most 100 ms. A reset
   raises no PWR_RDY.
2. After power-up PWR_RDY holds INT low until INT_STATUS_1 is read. `start`
   reads both status registers last.
3. The mode is written last, after the FIFO has been cleared. Changing the
   mode does not clear the FIFO or change any other setting.
4. Reading FIFO_DATA does not advance the register address. A burst read
   of 0x07 returns the bytes of consecutive samples. Any burst that crosses
   0x07 pops a sample; the driver reads the pointers 0x04 to 0x06 and the
   configuration 0x08 to 0x0A as separate reads.
5. A drain reads the pointers first and never asks for more than the FIFO
   holds. When the pointers meet, the FIFO is empty unless samples were
   lost, in which case it is full: a full FIFO with roll-over off stops
   taking samples until it is read, and the `max3010x` crate reads that as
   empty and never drains it.
6. A drain reads at most 30 bytes at a time, which is five samples in SpO2
   mode and ten in heart rate mode. If a burst fails, the read pointer is
   written back so the samples can be read again.
7. A FIFO that holds exactly 32 samples without having lost one looks
   empty for one sample interval: the pointers meet and the overflow counter
   is still 0. The next sample is lost, the counter moves, and the next
   drain reads all 32. Drain before the FIFO is full: the default
   almost-full level is 22 samples unread, 220 ms at 100 sps.
8. LED current is set in 0.2 mA steps and never above the request. The
   sheet says the actual current "can vary widely due to the trimming
   methodology".
9. Heart rate mode has the red LED only, so `ir` is 0 in its samples.
10. The sample rate and pulse width are limited as the table above gives;
    asking for more is clamped, and the answer is the rate the part took.
11. A temperature reading takes up to 29 ms (the sheet's Figure 4);
    `temperature_milli_c` waits up to 100 ms. The integer part is two's
    complement and the fraction adds even when the integer is negative.
12. The beat detector wants about 100 samples a second. Its `min_signal`,
    20000 counts, is a guess for the default configuration that has not
    been checked on a finger: set it from a reading with and without a
    finger. Another rate needs `fs_hz` set to match, and another pulse
    width rescales the counts.

## What is not included

NOT IMPLEMENTED:

- **The SpO2 calculation.** The package delivers the red and infrared
  counts and does no oxygen-saturation arithmetic. The conversion from the
  ratio of the two counts to a percentage needs a calibration curve that
  depends on the optics and the LEDs, and the algorithms that exist are
  the maker's or the application's own.
- The multi-LED mode with its four time slots (SLOT1 to SLOT4), and the
  heart rate mode on the infrared LED.
- The MAX30101's green LED and its extra channels. The register map is the
  same, so the driver may drive a MAX30101 in the modes above; it has not
  been tried, and `PpgLed.PpgGreen` is refused.
- The async form, `HeartRateAsync`, which a handle cannot implement
  (SPEC section 14.5), and the interrupt-driven wake on the almost-full
  interrupt.
- A total of samples lost, and time stamps counted from the start of
  sampling (see above).
- Motion compensation, signal-quality checks and every algorithm beyond
  the beat detector above.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the
  `HeartRate` and `Identifiable` traits this driver implements, the wiring
  record and the register helpers.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus` and `GpioOut` traits and the board handles.

## Tests

`novo test` runs the module over a double of the part: a register file
with the part ID, the self-clearing reset and temperature bits, interrupt
flags that a read clears, and a 32-slot FIFO with its pointers and
overflow counter, whose FIFO_DATA register returns consecutive bytes on a
burst. The suite checks every conversion against the sheet's tables, the
configuration bytes, the framing of the bus, the reset and its timeout,
the pointers and the samples waiting around the wrap and when full, a
drain of one channel and of two, of three samples, of nine in two bursts
and of a full FIFO in seven, the shift for each pulse width, a failed burst
and the read pointer written back, the LED currents and rates and what the
pulse width clamps, the interrupts, the temperature in both signs, the
bring-up, and the type over the host's board handles. The drain, the
temperature and the bring-up each run once for every transaction of a good
run with that transaction failing, and must answer that they failed. The
beat tests are described above. The suites cover every line of `src/`:

```sh
NOVO=bin/novo bash tests/coverage.sh
```

**Hardware: NOT VERIFIED.** See the device program section.

The sources read for this driver, for their fixes and not ported, are the
data sheet above, Zephyr's `drivers/sensor/maxim/max30101` (whose binding
also covers the MAX30102) and the `max3010x` crate of eldruin. Each fix
kept is named in `src/max30102.nv` where it is kept.

## Licence

Apache-2.0.

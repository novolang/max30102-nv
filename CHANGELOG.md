# Changelog

## 0.1.0

The first release: a driver for the MAX30102 pulse oximeter and heart-rate
sensor on an I2C bus, with a beat detector over its infrared counts.

- `Max30102`, the type a board's device table builds from the board's I2C
  bus, timer and GPIO handles and the wiring, implementing devices-nv's
  `HeartRate` and `Identifiable`. The value keeps no state, so every method
  takes a plain `self`. `Max30102Port` is the trait a test or a program
  implements to run the module's functions over its own bus.
- The part ID check, the reset with its wait, shutdown, and the bring-up
  `start`: reset, configuration, FIFO cleared, mode written last, both
  interrupt status registers read so that INT is released.
- The configuration `Max30102Config`: heart-rate or SpO2 mode, sample
  averaging, FIFO roll-over, the almost-full level, the sample rate clamped
  to what the pulse width allows (Tables 11 and 12), the pulse width, the ADC
  range and the two LED amplitudes.
- The FIFO: the pointers and the overflow counter read in one read that
  stops before FIFO_DATA, the samples waiting around the wrap and when the
  FIFO is full, a drain in bursts of at most 30 bytes, counts shifted by the
  bits the pulse width leaves at zero, stamps one FIFO period apart, and the
  read pointer written back when a burst fails.
- LED currents in 0.2 mA steps, never above the request; the interrupt
  enables and the status read; the die temperature in thousandths of a
  degree Celsius.
- `ppg_beat`: a beat detector over a stream of counts, `beat_feed` for one
  count and `beat_run` for a `HeartRate`. A starting point, tested over
  synthesised waveforms.
- The device program `examples/pulse_reader.nv`, which prints the raw counts
  and the rate. Built for the nRF52840-DK it is 24,100 B of flash and for
  the nRF54LM20-DK 25,540 B; before the Nordic buses landed, with the reading
  path left out, it was 18,312 B and 19,608 B.

### Known

- Hardware verification: NOT VERIFIED. No board on the bench carries the
  sensor. The nRF52840 DK drives I2C with TWIM0 and the nRF54LM20 DK with
  TWIM22, so the device program's images carry the reading path; a reply from
  the sensor over either bus is NOT VERIFIED. The beat detector has not seen a
  wearer's signal.
- The data sheet read is document 19-7740 Rev 1 (10/18), the revision that
  could be fetched.
- `ppg_lost` answers the overflow counter and `at_us` counts on the board's
  timer, not the total lost and the time since sampling started, because
  those need state and a driver with state cannot be called on a `let` by a
  function over `HeartRate`.
- Not implemented: the SpO2 calculation, the multi-LED mode, the MAX30101's
  green LED, `HeartRateAsync` and an interrupt-driven wake, a total of
  samples lost.
- `devices-nv` is the registry range `^0.3.3`, the first release with the
  `HeartRate` trait. `embedded-hal-nv` is the registry range `^0.3.2`.
- The `[tier]` of the manifest is `embedded`, the strictest, so the module
  also checks at `rt` and `system`.

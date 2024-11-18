# SPI for DxCore
The SPI library implements all of the standard functionality described in the [Arduino SPI library reference](https://www.arduino.cc/en/reference/SPI) applies here. Also, like all of the "big three" third-party cores for post-2016 AVR devices, this version of SPI.h supports the `swap()` and `pins()` methods to make use of the PORTMUX feature of the chips. However, there is an additional complication described below; this has finally been addressed in an (IMO) satisfactory way in 1.3.0.

## Pins
| Pin Mapping   | Pins        | Parts           | Swap-level name   |   Synonym  | Value | Note                      |
|---------------|-------------|-----------------|-------------------|------------|-------|---------------------------|
| SPI0 DEFAULT  | PA4-PA7     | 20+ pin         | SPI0_SWAP_DEFAULT | SPI0_SWAP0 | 0x00  | Core default `#`        - |
| SPI0 ALT1     | PE0-PE3     | 48+ pin         | SPI0_SWAP_ALT1    | SPI0_SWAP1 | 0x01  |                         - |
| SPI0 ALT2     | PG4-PG7     | DA/DB 64        | SPI0_SWAP_ALT2    | SPI0_SWAP2 | 0x02  |                         - |
| SPI0 ALT3     | PA0-1,PC0-1 | DD28+/EA/EB only| SPI0_SWAP_ALT3    | SPI0_SWAP3 | 0x03  | New mapping DD+ only `*$` |
| SPI0 ALT4     | PD4-PD7     | DD/EA/EB only   | SPI0_SWAP_ALT4    | SPI0_SWAP4 | 0x04* | New mapping DD+ only `*@` |
| SPI0 ALT5     | PC0-PC3     | DD/EA/EB only   | SPI0_SWAP_ALT5    | SPI0_SWAP5 | 0x05* | New mapping DD+ only `*$` |
| SPI0 ALT6     | PC1-PC3,PF7 | DD/EA/EB only   | SPI0_SWAP_ALT6    | SPI0_SWAP6 | 0x06* | New mapping DD+ only `*%` |
| SPI1 DEFAULT  | PC0-PC3     | DA/DB only      | SPI1_SWAP_DEFAULT | SPI1_SWAP0 | 0x80  | Only DA/DB Have SPI1      |
| SPI1 ALT1     | PC4-PC7     | DA/DB 48/64     | SPI1_SWAP_ALT1    | SPI1_SWAP1 | 0x84* | Only DA/DB Have SPI1      |
| SPI1 ALT2     | PB4-PB7     | DA/DB 64        | SPI1_SWAP_ALT2    | SPI1_SWAP2 | 0x88* | Not on 48-pin parts    `!`|

Notes:
* Parts with SPI1 (at least those currently announced) haven't had more than 3 total pin mapping options represented by just 2 bits. The DD-series and EA-series have more complicated mappings. When they release a part with >3 pin options and >1 SPI interface, this library will need to be adjusted, which is why you should always use the names.
* `*` Added on DD and later series of parts only.
* `@`SPI0 ALT4 is the default on 14-pin DD-series, as it is the lowest number they have with all pins.
* `$` SPI0 ALT5 likely works only on 28+ pin parts with a PC0, but it's not explicitly clear. Keep an eye on DD errata, where the issue with SPI Alt 2 on DA/DB 48 was
* `%` SPI0 ALT6 on DD-series parts is clearly to provide a way to get the important SPI lines onto the MVIO pins
* `!` SPI1 ALT2 - apparently - is supposed to be able to limp onwards despite not having SCK or SS pins one the DX-series 48 pin parts. That is supposedly busted. We were way ahead of the errata and never expected it to work.

## SPI pin swap methods
`SPI.swap(swap_level)` will set the the mapping to the specified pin swapping level. It will return true if this is a valid option for the part you're using, and false if it is not (you don't need to check this, but it may be useful during development). If an invalid option is specified, it will be set to SPI0 on the default pins.

`SPI.pins(MOSIpin, MISOpin, SCKpin);` or `SPI.pins(MOSIpin, MISOpin, SCKpin, SSpin);` can also be used - this will set the mapping to whichever mapping has the specified pins. MISO, MOSI, and SCK must be specified correctly; the SS pin is ignored (the option to specify it is provided only for backwards compatibility. It was decided that requiring it to be passed, when it's functionality would require a different library and is explicitly disabled, didn't really make much sense. If the pins passed are not a valid mapping option, it will return false and set the mapping to the default. This uses more flash than `SPI.swap();` - there is no advantage, and consequently SPI.swap() is recommended.

When `SPI.swap()` or `SPI.pins()` is called, assuming it was called with a valid option, the pin mapping requested is saved. When `SPI.begin()` is called, this stored value will specify changes to `PORTMUX.SPIROUTEA` and `PORTx` registers. Thus, if a non-default pin mapping is required, you must set it before before calling `SPI.begin()`. To change the pin mapping after `SPI.begin()` you must turn off SPI with`SPI.end()`, call `SPI.swap()` or `SPI.pins()` and then `SPI.begin()` again.

## Two SPI ports
The AVR DA/DB-series parts have two hardware SPI ports. On parts with more pins, they can be pin-swapped to different sets of pins (up to three sets of pins per SPI peripheral). The AVR DD-series has only a single SPI port - but it has a far more pin options than the DA/DB-series parts do. Originally, it was expected that two libraries could be created like is done for the few classic AVRs with multiple SPI ports (eg, ATmega328PB) and the many 32-bit architectures with multiple SPI ports; however, it was discovered in 1.2.0 (which attempted to implement this) that the existing libraries with which we desire compatibility (an SPI library that you need to modify everything you use with it is hardly satisfactory) were more challenging to work with than expected. In order to work with existing libraries, we need only guarantee that our instances of SPI_class have names matching the convention; that sounds like a low bar - and indeed, it is: the only way it could be a problem is if one of those key names happened to already be used for something, and not just any something, but something which had a greater authority to be naming things than anything the core or core libraries did.

### The problem
As noted above the `SPI_class` object for the second SPI port would have to match the convention - first SPI port is named `SPI`, second is `SPI1` and so on. You may recall that Microchip names their peripherals (as defined in the io header files included by all sketches) as a small number of capital letters, followed by an integer, starting with 0. The first SPI peripheral, and the struct with references to all of it's registers, is named `SPI0`... And the second is named `SPI1`.  It was discovered that in the field, the standard library solution is that if a board claims it has 2 SPI interfaces, SPI-using libraries will expect the second one to be an instance of SPI_class named `SPI1`, and if this was not the case, that would generate a compile error. But here, we can't use SPI1 as the name, because it's already taken! Well - what do we do? It, of course, is possible to override the io headers by #undefining it - I considered this, but I think it sets a poor precedent to override "real" Microchip-declared peripheral names for the sake of making Arduino-land stuff work better. There was just no graceful way to have multiple SPI instances here.

### The solution
However, on reflection - I realized that... does it really even make sense to have two instances of SPI_class?
1. It is rare to need two physical SPI ports - SPI is a bus, and many parts can share the same MISO, MOSI and SCK lines.
1. The standard SPI library API does not support slave mode, so the plausible case of a part being both a master to sensors or other simple devices, while something more computing-heavy like a Raspberry Pi controls it over SPI is not an issue. Because SPI.h has nothing to do with SPI slave mode, and another library must be used for that, all that we require of SPI.h in order for it to be no worse than the stock SPI library is that it leave whichever SPI port it's *not* using alone.
1. The most compelling reason to make use of the second SPI port and why the original attempt at a second SPI library was such a high priority was due to the highly constrained pin options on the 28-pin and 32-pin parts - only one set of pins is available for each SPI interface. The available pins for SPI0 are PA4-PA7. When those are the only SPI pins available, using SPI becomes mutually exclusive with using some of the most powerful and important pins on the part: There are many useful peripherals limited to those pins and those alone (TCD0 until we get a fix for the errata, and the only option for Serial0 when external clock sources are used, among others). The pins of SPI1 are more convenient - if only you could make SPI.h use those. Once it was recognized that only a single instance was needed to support realistic use cases, the route to implement this was clear, and quite simple - we just have a member variable pointing to either SPI0 or SPI1 that we use in place of the hardcoded references, and set the swap level and the pointer at the same time: since swap level itself it at most 3 bits, so it's not a problem to use the MSB of the swap level to represent the SPI instance, and we have plenty of room to scale this.

As of 1.3.0, the version of SPI.h included with DxCore allows all SPI0 and SPI1 pin mappings to be used via the SPI.swap() and SPI.pins() functions described below. Unlike other peripheral libraries that provide a similar `swap()` method, the SPI library defines constants to pass to `SPI.swap()` - two names for each are shown on the table at the top of this page; the naming of the pin mappings ("DEFAULT", "ALT1", "ALT2") matches what Microchip calls them, and is hence our recommendation. For convenience the numeric values are also listed - though as always, we strongly discourage users from passing numeric values or setting registers to them when named constants are available. Your code is more readable with the constants, and it helps future proof your code.

## UsingInterrupt() and the new attachInterrupt implementation
1.3.8 introduced a new attachInterrupt implementation which increases flexibility and allows manually defined pin interrupts. It was soon reported that this was not compatible with SPI.h. 1.3.9 introduces a workaround:

Unless the old attachInterrupt implementation is selected, `SPI.usingInterrupt(number)` will cause it to globally disable interrupts while in a transaction. This is not ideal, however *you should not have interrupts performing SPI transactions, period* - SPI transactions are not fast; interrupts should be fast. This is here for compatibility ONLY. The standard implementation of this does not map cleanly onto the new attachInterrupt(). Furthermore, the standard implementation has several logic gaps and race conditions which, together, could still result in the same sort of behavior it was aimed at preventing.
Similarly, `SPI.notUsingInterrupt(number)` will set it back to the normal mode. This is also not ideal, but we needed a timely fix and as I said, this functionality only, as supplied, approximated correct behavior. I'll come back to this at some point, but it is no longer an urgent - not least because you should never use these functions, because you should not be using SPI from interrupts! The one exception to this is the case where the number passed is NOT_AN_INTERRUPT (-1 or 255), in which case we match the old behavior and simply return.

## SPI.attachInterrupt and SPI.detachInterrupt
These methods, as far as I can tell were never supported for the official or third party AVR boards, only third party extensions where some other API was used to support SPI slave mode functionality. They were marked as something that should never be called, and there was no sign of code that made use of them within the core. In fact, there was no sign of use of them on the wider internet - except for slave mode extensions for different architectures; hence, I feel safe ditching these.

## SS (Slave Select) pin
On the Dx-series parts, the SS pin can be configured to - when driven low -  switch the SPI peripheral into Slave mode, where it acts as an SPI slave (the same feature was present on the classic AVR parts) - however, this library does not support slave mode (nor - as far as I am aware, has any official SPI library. It would seem that the Arduino userbase is much more enthusiastic about SPI master mode than slave mode; that is not particularly surprising. A basic TWI slave device has a lot of advantages, both in terms of how the code is written, and
This core disables the SS pin when running in SPI master mode. This means that the "SS" pin can be used for whatever purpose you want - unlike classic AVRs, where the "slave-select" functionality of the SS pin could not be disabled (on classic AVRs, if that pin was input, and it went low - SPI was now in slave mode, whether you like it or not! And within Arduino circles "not" was pretty much universal, since SPI.h doesn't support slave mode.

## Slave implementation
As noted above, SPI.h does not support SPI slave mode, never has, and never will. While a library to implement this is conceivable, SPI is in fact FAR simpler than I2C/TWI - there are a grand total of 5 registers, including the data register; implementing an SPI slave device can be done comparatively easy by directly configuring the registers, and there has been very little interest on forums in SPI slave devices, and a great deal of interest in I2C slaves. I believe the reaqson is that SPI is run as speeds so much higher, and provide no mechanism for clock stretching. Within the Arduino paradigm of taking a callback function called within the ISR, there is no way that one can have prayer of meeting relevant timing constraints except at exceedingly low clock speeds. Between the first sign that an

Personally, I think the greatest potential on these parts will come from using SPI slave mode with the part controlling it's own SS and SCK pins by connecting another output pin to them, taking advantage of the SPI shift-register and buffer, as well as one or more CCL blocks to efficiently output interrupt driven, non-SPI protocols. Plans are afoot to use several CCL blocks and event channels, the TCD, and at least one more timer to output neopixel data in the background, rather than monopolizing the CPU for it (particularly when these are overclocked, it starts to get a little absurd, with around 80% of the time it spends sending used in delays); if we could recover a portion of that to use for calculating what to display next, great. If we could calculate what to display during some of that time, huge frame rate improvements are possiblem.


## Note on terminology
At this point we are continuing to use the "master" and "slave" terminology for I2C and SPI devices; In some of it's reference material, Microchip has switcheed to "Host" and "Client", but this is reflected in only a small portion of their documents and is not reflected in register names, while countless thousands of documents, datasheets, guides, and code examples use "master" and "slave" (and while those words are distasteful, they are totally unambiguous, while host and client are not). Since clarity and avoidance of confusion is our goal we continue to use the old terminology, which is familiar to our users (particularly those who are not native english speakers and are likely not following politics in the western world). We continue to monitor the situation, and will switch when a replacement (hopefully something that is less ambiguous than "host" and "client")
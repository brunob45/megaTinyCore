# Improved Digital I/O Functionality
This core includes a number of features to provide more control or performance when doing digital I/O. This page describes how to use them. All of the options that can be configured for a pin are exposed. The only things that aren't exposed are the slew rate limiting feature, and the multi-pin configuration facilities. The slew rate limiting is can only be configured on a per port basis; turning it on and off is so simple (see below) that it needs no wrapper. The multi-pin configuration system does not have an obvious "right way" to expose it and should be handled directly by the sketch - it is very flexible and no wrapper around it would be able to preserve it's virtues while being much of a wrapper.

## Ballpark overhead figures
The digital I/O functions are astonishingly inefficient. This isn't my fault - it's the Arduino API's fault
These figures are the difference between a sketch containing a single call to the function, with volatile variables used as arguments to prevent compiler from making assumptions about their values, which may substantially reduce the size of the binary otherwise.

The comparison to the fast I/O functions is grossly unfair, because the fast I/O functions have constant arguments - but this gives an idea of the scale.
Remember also that executing 1 word of code (2 bytes) takes 1 clock cycle, though not all of the code paths are traversed for all pins (indeed, we know that large portions will be skipped for all pins, because each pin can have only one PWM timer, but on the other hand, there are a few small loops in there). Gives an idea of the scale.

| Function           | t3224 | t3216 | Notes
|--------------------|-------|-------|-------------------------
| pinMode()          |  +128 |  +146 |
| digitalWrite()     |  +198 |  +340 | Calls turnOffPWM(), included in figures.
| both of above      |  +296 |  +438 | Calls turnOffPWM()
| openDrain()        |  +128 |  +262 | Calls turnOffPWM()
| turnOffPWM()       |   +72 |  +228 | 3216 has the complicated type D timer, so this is huge.
| digitalRead()      |   +76 |   +78 | Official core calls turnOffPWM (we don't)
| pinConfigure() 1   |  +104 |  +116 | Only direction/output level
| pinConfigure() 2   |  +202 |  +220 | Only pullup, input sense, and direction/outlevel
| pinConfigure() 3   |  +232 |  +250 | Unconstrained
| digitalWriteFast() |    +2 |    +2 | With constant arguments.
| pinModeFast()      |   +12 |   +12 | With constant arguments.

The takeaways from this should be that:
* digitalWrite() can easily take 35-75 or more clock cycle with turnOffPWM and then over 50 itself. So digitalWrite can take up to 7 us per call (at 20 MHz; digitalWriteFast takes 0.05 us), and it is by far the worst offender. A small consolation is that turnOffPWM does not execute the entire block of code for all parts. It checks which timer is used, and uses that block, which very roughly gives 70 clocks on a TCA pin, around 150 on a TCD pin, and considerably fewer on a pin with no PWM, though it still has to determine that the pin doesn't have pwm.
* If you know you won't have PWM coming out of a pin, pinConfigure is faster than pinMode() and digitalWrite(), especially if only setting up the more common properties.
* pinConfigure is optimized differently when called with the second argument constrained to 0x00-0x0F (setting only direction and output value) (1), with it constrained to that plus configuring pullups (2), and without restrictions (3).
* 4 bytes, times the number of the highest number digital pin, are in the form of lookup tables.
* This is why the fast digital I/O functions exist, and why there are people who habitually do `VPORTA.OUT |= 0x80;` instead of `digitalWrite(PIN_PA7,HIGH);`
* This unexpectedly poor performance of the I/O functions is also why I am not comfortable automatically switching digital I/O to "fast" type when constant arguments are given. The normal functions are just SO SLOW that such a change would be certain to break code implicitly. I have seen libraries which, for example, would not meet (nor come close to meeting) setup and hold time specifications for an external device, but for the grindingly slow pin I/O. *This breakage would likely be wholly unexpected by both library author and the user, and would furthermore be very difficult for most users to diagnose, requiring an oscilloscope, awareness of the poor performance of these API functions, and sufficient familiarity with electronics to know that this was a problem that could occur*. Hence, it would be directly contrary to the goal of making electronics accessible (which Arduino is directed towards), and the goal of maintaining a high degree of compatibility with existing Arduino libraries, which is one of the primary goals of most Arduino cores.

megaTinyCore and DxCore, like all Arduino cores, provides implementations of the usual digital I/O functions. *These implementations are not guaranteed to have identical argument and return types, though we do guarantee that same will be implicitly convertible to the types used by the reference implementation*

Below are listed *additional* functions provided by these cores for the purpose of providing greater functionality, improving the usability of the core, and the aesthetic beauty of not "hiding" the enhanced capabilities of modern AVR microcontrollers. Many of these functions are entirely appropriate for any microcontroller (though some are not; the capabilities), and I encourage others to use these implementations in the interest of making code which does require these more advanced I/O facilities to be portable across the widest variety of devices possible; beside that, I have a direct interest in that, because I may find myself using those devices in the future.

## openDrain()
It has always been possible to get the benefit of an open drain configuration - you set a pin's output value to 0 and toggle it between input and output. This core provides a slightly smoother (also faster) wrapper around this than using pinmode (since pinMode must also concern itself with configuring the pullup, whether it needs to be changed or not, every time - it's actually significantly slower setting things input vs output. The openDrain() function takes a pin and a value - `LOW`, `FLOATING` (or `HIGH`) or `CHANGE`. `openDrain()` always makes sure the output buffer is not set to drive the pin high; often the other end of a pin you're using in open drain mode may be connected to something running from a lower supply voltage, where setting it OUTPUT with the pin set high could damage the other device.

Use pinMode() to set the pin `INPUT_PULLUP` before you start using `openDrain()`, or use `pinConfigure()` if you need the pullup enabled too; this doesn't change it. Note that there is nothing to stop you from doing `pinMode(pin,INPUT_PULLUP); openDrain(pin,LOW);` That is non-destructive (no ratings are exceeded), and would be a good place to start for a bitbanged open drain protocol, should you ever create one - but it doesn't exactly help with power consumption if you leave it like that! If running on batteries, be sure to turn off one of the two, either by using openDrain to release it to to pullups, or digitalWrite'ing it `HIGH`, or any other method of configuring pins that gets you a pin held in a defined state where it isn't working against it's own pullup.

```c++
openDrain(PIN_PA1, LOW); // sets pin OUTPUT, LOW.
openDrain(PIN_PA1, FLOATING); // sets pin INPUT. It will either float, or (if pullup was previously enabled) be pulled up to Vdd.
```

## Fast Digital I/O
This core includes the Fast Digital I/O functions, digitalReadFast(), digitalWriteFast() and openDrainFast(). These are take up less flash than the normal version, and can execute in as little as 1 clock cycle. The catch is that the pin MUST be a known constant at compile time. For the fastest, least flash-hungry results, you should use a compile-time known constant for the pin value as well. Remember than there are three options here, not just two. If you only want two of the options, you will get smaller binaries that run faster by using the ternary operator to explicitly tell the compiler that the value is only ever going to be those two values, and then it can optimize away the third case.

An openDrainFast() is also provided. openDrainFast() also writes the pin LOW prior to setting it output or toggling it with change, in case it was set `INPUT` and `HIGH`. `CHANGE` is slightly slower and takes an extra 2 words of flash because the `VPORT` registers only work their magic when the sbi and cbi (set and clear bit index) instructions will get you what you want. For the output value, writing a 1 to the bit in `VPORT.IN` will toggle it, and that's what we do, but there is no analogous way to change the pin direction, so we have to use a full STS instruction (2 words) to write to the `PORT.DIRTGL` register after spending another word on LDI to load the bit we want to toggle into a working register.

It also includes pinModeFast(). pinModeFast() requires a constant mode argument. INPUT and INPUT_PULLUP as constant arguments are still as large and take as long to execute as digitalWriteFast with either-high-or-low-state - they turn the pullups on and off (use openDrainFast(pin,FLOATING) to execute faster and leave the pullups as is).

```c++
digitalWriteFast(PIN_PD0, val); // This one takes 10 words of flash, and executes in around half 7-9 clocks.
digitalWriteFast(PIN_PD0, val ? HIGH : LOW); // this one takes 6 words of flash and executes in 6 clocks
digitalWriteFast(PIN_PD0, HIGH); // This is fastest of all, 1 word of flash, and 1 clock cycle to execute
VPORTD.OUT |= 1 << 0; // digitalWriteFast(PIN_PD0,HIGH) is turned into this.
                      // The previous line is syntactic sugar for this. Beyond being more readable
                      // it eliminates the temptation to try to write more than one bit which instead
                      // of an atomic SBI or CBI, is compiled to a read-modify-write cycle of three
                      // instructions which will cause unexpected results should an interrupt fire in the
                      // middle and write the same register.
```

### Flash use of fast digital output functions

| function            | Any value | HIGH/LOW | Constant        |
|---------------------|-----------|----------|-----------------|
| openDrainFast()     | 14 words  | 7 words  | 2 words if LOW<br/>1 if FLOATING |
| digitalWriteFast()  | 10 words  | 6 words  | 1 words         |
| pinModeFast()       |      N/A  |     N/A  | 1 word if OUTPUT<br/>6 otherwise |

Execution time is 1 or sometimes 2 clocks per word that is actually executed (not all of them are in the multiple possibility options. in the case of the "any option" digitalWrite, it's 5-7
Note that the HIGH/LOW numbers include the overhead of a `(val ? HIGH : LOW)` which is required in order to get that result. That is how the numbers were generated - you can use a variable of volatile uint8_t and that will prevent the compiler from assuming anything about it's value. It is worth noting that when both pin and value are constant, this is 2-3 times faster and uses less flash than *even the call to the normal digital IO function*, much less the time it takes for the function itself (which is many times that. The worst offender is the normal digitalWrite(), because it also has to check for PWM functionality and then turn it off if enabled (and the compiler isn't allowed to skip this if you never use PWM).

1 word is 2 bytes; when openDrainFast is not setting the pin FLOATING, it needs an extra word to set the output to low level as well (just in case). This was something I debated, but decided that since it could be connected to a lower voltage part and damage caused if a HIGH was output, ensuring that that wasn't what it would do seemed justified; if CHANGE is an option, it also uses an extra 2 words because instead of a single instruction against VPORT.IN, it needs to load the pin bit mask into a register (1 word) and use STS to write it (2 words) - and it also does the clearing of the output value - hence how we end up with the penalty of 4 for the unrestricted case vs digitalWriteFast.

The size and speed of digitalReadFast is harder to calculate since it also matters what you do with the result. It is far faster than digitalRead() no matter what. The largest impact is seen if you plan to use it as a test in an if/while/etc statement; there are single-clock instructions to skip the next instruction if a specific bit in a "low I/O" register like the ones involved is set or cleared. Thus, it is most efficient to do so directly:
```c
if (digitalReadFast(PIN_PA3)) {
  Serial.println("PA3 is high")
}
```
Getting a 1 or 0 into a variable is slower and uses more flash; I can think of at least one way to do it in 3 which makes no assumption about whether it's stored in an upper or lower register, but I'm not confident in how clever the compiler is, and it's tricky to set something up where the compiler doesn't have options such that you can generalize the result.

**The fast digital I/O functions do not turn off PWM** as that is inevitably slower (far slower) than writing to pins and they would no longer be "fast" digital I/O.

## pinConfigure()
pinConfigure is a somewhat ugly function to expose all options that can be configured on a per-pin basis. It is called as shown here; you can bitwise-or any number of the constants shown in the table below. All of those pin options will be configured as specified. Pin functions that don't have a corresponding option OR'ed into the second argument will not be changed. There are very few guard rails here: This function will happily enable pin interrupts that you don't have a handler for (but when they fire it'll crash the sketch - however, note that if you've 'attached()' an interrupt, you can then enable and disable it with this. This is likely faster - and also likely more robust on principle (and defining the interrupt without attach better still) on the grounds that the less you play with function pointers, the lower the chance that there's a latent bug that can corrupt the value of that pointer, leading to unpredictable results when it is dereferenced), or waste power with pins driven low while connected to the pullup and so on.

```c++
pinConfigure(PIN_PA4,(PIN_DIR_INPUT | PIN_PULLUP_ON | PIN_OUT_LOW));
// Set PIN_PA4 to input, with pullup enabled and output value of LOW without changing whether the pin is inverted or triggers an interrupt.

pinConfigure(PIN_PA2,(PIN_DIR_INPUT | PIN_OUT_LOW | PIN_PULLUP_OFF | PIN_INVERT_OFF | PIN_ISC_ENABLE));
// Set PD2 input, with output register set low, pullup, invert off, and digital input enabled. Ie, the reset condition.
```


| Functionality |   Enable  | Disable            | Toggle |
|---------------|-------|---------------------|--------------------|
| Direction, pinMode() | `PIN_DIR_OUTPUT`<br/>`PIN_DIR_OUT`<br/>`PIN_DIRSET` | `PIN_DIR_INPUT`<br/>`PIN_DIR_IN`<br/>`PIN_DIRCLR`       | `PIN_DIR_TOGGLE`<br/>`PIN_DIRTGL` |
| Pin output, `HIGH` or LOW | `PIN_OUT_HIGH`<br/>`PIN_OUTSET`         | `PIN_OUT_LOW`<br/>`PIN_OUTCLR`          | `PIN_OUT_TOGGLE`<br/>`PIN_OUTTGL`       |
| Internal Pullup  | `PIN_PULLUP_ON`<br/>`PIN_PULLUP`        | `PIN_PULLUP_OFF`<br/>`PIN_NOPULLUP`       | `PIN_PULLUP_TGL`       |
| Invert `HIGH` and LOW |`PIN_INVERT_ON`        | `PIN_INVERT_OFF`       | `PIN_INVERT_TGL`       |
| Digital input buffer | `PIN_INPUT_ENABLE`or<br/> `PIN_ISC_ENABLE`    | `PIN_ISC_DISABLE` or<br/>`PIN_INPUT_DISABLE`    | Not supported<br/>No plausible use case |
| Interrupt on change | `PIN_ISC_ENABLE` or<br/> `PIN_INPUT_ENABLE`       | `PIN_ISC_ENABLE` or<br/>oth     | Not applicable |
| Interrupt on Rise  | `PIN_ISC_RISE` or<br/> `PIN_INT_RISE`         | `PIN_ISC_ENABLE` or<br/>`PIN_ISC_DISABLE`     | Not applicable |
| Interrupt on Fall  | `PIN_ISC_FALL` or<br/> `PIN_INT_FALL` | `PIN_ISC_ENABLE` or<br/>`PIN_ISC_DISABLE`      | Not applicable |
| Interrupt on LOW  | `PIN_ISC_LEVEL`  or<br/> `PIN_INT_LEVEL` | `PIN_ISC_ENABLE` or<br/>`PIN_ISC_DISABLE`      | Not applicable |

For every constant with TGL or TOGGLE at the end, we provide the other spelling as well. For every binary option, in addition to the above, there is a `PIN_(option)_SET` and `PIN_(option)_CLR` for turning it on and off. The goal was to make it hard to not get the constants right.

While pinConfigure is not as fast as the fast digital I/O functions above, it's still faster than pinMode().

Again, note that unlike digitalWrite() this does not turn off PWM.

## Slew Rate Limiting (Dx-series and 2-series only)
All of the Dx-series parts have the option to limit the [slew rate](https://en.wikipedia.org/wiki/Slew_rate) for the OUTPUT pins on a on a per-port basis. This is typically done because fast-switching digital pins contain high-frequency components (in this sense, high frequency doesn't necessarily mean that it repeats, only that if it continued, it would be high frequency; this way of thinking is useful in electrical engineering), which can contribute to EMI, as well as ringing (if you're ever looked on a scope and noticed that after a transition, the voltage briefly oscillates around the final voltage - that's ringing) which may confuse downstream devices (not usually, at least in arduino land, though). Often, you will not know exactly *why* it's an issue, either, rather a datasheet may specify a maximum slew rate on it's inputs.
```c
// These work but are slow and inefficient.
// Enable slew rate limiting on PORTA
PORTA.PORTCTRL |= PORT_SRL_bm;
// Disable slew rate limiting on PORTA
PORTA.PORTCTRL &= ~PORT_SRL_bm;
```
However, because there is only one bit in this register, you don't actually need to use the |= and &= operators, which do a read-modify-write operation, taking 10 bytes and 6 clocks each. Instead, you can use simple assignment. This saves 4 bytes (3 clocks) when enabling, and 6 bytes (4 clocks) on disabling. (going from `LDS, ORI/ANDI, STS` to `LDI, STS` and just `STS`, since avr-gcc always keeps a "known zero" in r1). Of course, if a future part with more port options comes out, porting to that part would require a code change if you also wanted to use one of those new bits. I suggest a reminder that it's a shortcut if you take it, like the comment below.

```c
// Enable slew rate limiting on PORTA more efficiently
PORTA.PORTCTRL = PORT_SRL_bm; // simple assignment is okay because on DA, DB, and DD-series parts, no other bits of PORTCTRL are used, saves 2 words and 3 clocks
// Disable slew rate limiting on PORTA
PORTA.PORTCTRL = 0;           // simple assignment is okay because on DA, DB, and DD-series parts, no other bits of PORTCTRL are used, saves 3 words and 4 clocks
```

These parts (like many electronics) are better at driving pins low than high (all values are from the typ. column - neither min nor max are specified; one gets the impression that this is not tightly controlled):

| Direction   | Normal | Slew Rate Limited |
|-------------|--------|-------------------|
| Rising      |  22 ns |             45 ns |
| Falling     |  16 ns |             30 ns |

With a 24 MHz system clock, that means "normal" would be just over half a clock cycle while rising, and just under half a clock cycle falling; with slew rate limiting, it would be just over a clock cycle rising, and 3/4ths of a clock cycle on the falling side.

## There is no INLVL or PINCONFIG on tinyAVR devices
The configurable input level option (`INLVL`) is only available on parts with MVIO (AVR DB and DD-series).

Multi-pin configuration registers, which allow setting PINnCTRL en masse are only available on AVR Dx-series.


## turnOffPWM(uint8_t pin) is exposed
This used to be a function only used within wiring_digital. It is now exposed to user code - as the name suggests it turns off PWM (only if analogWrite() could have put it there) for the given pin. It's performance is similar to analogWrite (nothing to get excited over), but sometimes an explicit function to turn off PWM and not change anything else is preferable: Recall that digitalWrite calls this (indeed, as shown in the above table, it's one of the main reasons digitalWrite is so damned bloated on the DX-series parts!), and that large code takes a long time to run. DigitalWrite thus does a terrible job of bitbanging. It would not be unreasonable to call this to turn off PWM, but avoid digitalWrite entirely. The pin will return to whatever state it was in before being told to generate PWM, unless digitalWriteFast() or direct writes to the port register have been used since. If the pin is set input after PWM is enabled with analogWrite (or through other means - though in such cases, the behavior when turnOffPWM() is called on any pin which nominally outputs PWM controlled by the same timer *should be treated as undefined* - it is not possible to efficiently implement such a feature on AVRs nor most other microcontrollers while maintaining behavior consistent with the API - the core would (at least on an AVR) have to iterate through every possible source of PWM, testing each in turn to see if it might have PWM output enabled on this pin. In many cases, including this core, each timer would require checking a separate bit in one or more special function registers, as well as storing a database that associates each pin with one or more combinations of bits and the corresponding bit to clear to disable PWM through that pathway. Suffice to say, that is not practical within the resource constraints of embedded systems.

## Finding current PWM timer, if any: digitalPinToTimerNow() (DxCore only)
On many cores, there is a `digitalPinToTimer(pin)` macro, mostly for internal use, which returns the timer associated with the pin. The Arduino API implicitly assumes that this is constant, and both core libraries and code in the wild often assume that it is. The digitalPinToTimer() implementation is as a macro, which frequently is optimized to a constant (provided the argument is constant, of course). The assumption that there exists not-more-than-1-to-1 timer:pin mapping was never universally valid even among AVRs (there are examples of classic AVRs that have multiple timers usable on some pins, and some even permit that to be remapped easily at runtime (the classic ATtiny841 being a notable example). All modern AVRs have at least two pins available to each timer output compare channel; and while the tinyAVRs never have more than two timers potentially usable with a given pin, the Dx-series allows one of up to 7 pins to be associated with a given timer (in groups), with some pins having as many as 4 options.

For the modern tinyAVRs, during development of megaTinyCore the design decision was made to not support PWM output via analogWrite() for any timer *type* if that *type* would add not more than 1 additional PWM pin when the type A timer was used in split mode (which has up to 6 channels) in order to make efficient use of limited flash. This leaves out the type D timer on 8-pin and 14-pin 1-series parts, and the type B timer(s) on all parts. Either of these would nearly double the flash required for PWM (there are plans to provide more than one option for PWM layouts in a future version of megaTinyCore via a menu option, fixed at compile time, which would provide a means for their capabilities to be more readily used without imposing unwanted overhead when not needed). In any event, on megaTinyCore, `digitalPinToTimer()` is simply an alias of `digitalPinToTimer()` and this will remain the case.

On the Dx-series parts, timers, pins, multiplexing options, and clock cycles are all more abundant, as are other peripherals competing for any given pin meanwhile, counterintuitively, the multiplexing scheme is actually simpler, and that inflexibility was not appropriate. As of 1.3.x, `analogWrite()` works for either type A timer *provided* that the appropriate `PORTMUX` register is set first. As of 1.5.x this is extended to the type D timer where hardware allows (at time of writing, only the AVR DD-series; there is a silicon bug impacting all production DA/DB-series parts. It is expected that future silicon revisions will correct this on those parts, if those are ever made available; having waited more than 2 years hoping for such fixes, and 5 years for other corrections for the tinyAVRs, I am not optimistic). On DxCore there is both `digitalPinToTimer()` and `digitalPinToTimerNow()`. The former never returns TIMERAn, but will return a constant referring to a type D timer *channel* or a type B *timer*. The latter tests the relevant `PORTMUX` register if that pin can use a type A timer; If it cannot, it then checks the standard `digitalPinToTimer()`. If that indicates that a type D timer channel can be used with the pin, it then performs a bitwise AND with the type D timer mux register, which will be true if the TCD is pointed at that pin. If that gives us a type B timer, that timer is returned as the result (notice that pins that can use either the type D timer or a type B timer will never return the type B timer, even if the type D timer is pointed elsewhere - this was a necessary compromise for flash and code complexity considerations.

**In no case will either of these macros ever return a timer which the core does not permit use of analogWrite() with.**

Furthermore, the following situations should not be expected to produce the desired result:
* Calling turnOffPWM, digitalPinToTimer, digitalPinToTimerNow or analogWrite on a pin nominally driven by a timer which the core has been instructed not to reconfigure using the takeOverTCxn() function. The core will treat the pin as an ordinary digital pin - by calling that, you assume full responsibility for all management of that timer. That function must be called if any manual configuration of a PWM timer will be or has been performed, except as specifically noted in in the timer and PWM reference or type D timer reference.
* Calling turnOffPWM, digitalPinToTimer, digitalPinToTimerNow, analogWrite or digitalWrite on a pin nominally driven by a type A timer which has been manually reconfigured, particularly if it has been configured in*single mode* for 3x16-bit PWM channels. The Arduino API does not support PWM with > 8 bits of resolution. Users who require that level of control must call takeOverTCxn(). See the timer and PWM reference.
* Calling turnOffPWM, digitalPinToTimer, digitalPinToTimerNow, analogWrite or digitalWrite on a pin nominally driven by a type B timer, but which has been configured to use an output pin other than the one shown on the DxCore pin mapping for that part - this will reconfigure PWM on the wrong pin if at all.
* Manually configuring any timer without calling takeOverTCxn.

## Note on number of pins and future parts
The most any announced AVR has had is 86 digital pins, the ATmega2560; In the modern AVR era, the digital pin maximum is 55, and the maximum possible without rearchitecting how pins are accessed is 56 (ie, if UPDI could be turned into I/O on DA/DB we are already there). There is no way for an AVR that allows the same bit-level access that we enjoy to have it on on all pins to have more than 56 pins within the AVR instructionset. Each port takes up 4 of the 32 addresses in the low I/O space for the VPORT registers, on which the single cycle bit-level access relies. Only time will tell how this is handled.
* **No VPORTH or higher** - This is clean, simple, and limiting in some ways, but probably the best route. If you needed to use VPORT access, you'll just need to use the lower 7 ports for it. Surely you don't need 57+ pins all with VPORT atomic single cycle access! This is also, in my judgement, most likely based on the fact that on the ATmega2560, the only precedent we have for running out of low I/O registers for pins, they didn't even put ports H through L into the high I/O space.
* **VPORTs in High I/O** - The only way that this really solves any problems is by if they were to extend SBI/CBI/SBIC/SBIS to another 16 registers, enough to bring it to the same approximate I/O pin count as the 2560. This would only take 16 x 8 x 4 = 512 opcodes (unlike other instruction set wishlist items, which would be theoretically impossible to add to AVR), and one might note that they seem to have left themselves room for this in the instruction set around the current SBI/CBI isns. But the need for updated toolchains would be painful indeed. This would generate the best product.
* **VPORTs that can be remapped at runtime** - The stuff of library author's nightmares. You'd have some subset of the VPORTs (or maybe all of them) could be remapped. If you put the register for that in the high I/O space it could be written quickly. But this will cause unaware old code to break by writing to the wrong pins if it didn't know to check for it, and any code that used them (ie, code that is supposed to be fast) might have to check the mapping first. This is most like what was done on the XMegas, but also serves as a shining example of the over-complexity that I think doomed those parts to poor uptake and slow death. The best of these bad approaches would probably be to have VPORTG changed into say VPORTX. Could configure it with a register in the high I/O space so it was 2 cycle overhead to write (1 for LDI port number, 1 to OUT to register). That way all code for other ports wouldn't worry, and port G is present on the fewest parts so the least code would be impacted if ported, and would be straightforward to document

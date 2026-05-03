# Building a USB-C / Bluetooth DAC and Headphone Amp from Scratch

*[Month Year] — [your name/handle]*

---

I've been meaning to build a proper desktop DAC for a while because I wanted to actually understand what's going on inside one. I work in power electronics and embedded hardware, and designed an amp in school, so I wanted to extend that experience.  

So here's how that went.

---

## Goals

Before I get into what I built I want to list my design targets, since they added significant design challenges that drove several decisions.  

- USB-C audio input, with UAC 2.0 support - no OS drivers to write
- USB-C power from a standard 5V/3A supply - no wall wart, no PD controller IC, just CC resistors.  Plenty of power, but it needs plenty of sanitization.  
- A Class A output stage designed around the Beyerdynamic T90 as the primary load - simple and inefficient amp design, but these are pretty sensitive headphones.  
- A PCB I actually laid out myself, with deliberate decisions about grounding and power partitioning

The last point matters more than the others. I could have bought a CM6533 eval board and called it a day. The whole point was to make the design decisions myself and live with the consequences.

A few things I originally planned didn't make it into the first revision. Bluetooth audio input got cut when I couldn't source a suitable module with direct I2S output at a reasonable price. A display for EQ mode selection got deferred until the base firmware is stable, and a graphics driver can be selected. And I want a more efficient output stage eventually — Class A is simple and well-understood, but it burns quiescent power whether you're listening or not. I haven't evaluated the alternatives carefully enough to commit to a direction, so that's a future revision problem. The XMOS architecture was partly chosen because these additions can all use the XMOS as the hub, and be treated as peripheral add-ons, rather than a completely new design between revisions. 

The T90 constraint deserves its own note. It's a 250Ω headphone rated at 102 dB SPL/mW with a maximum power handling of 200 mW. Driving it well from a 5V USB supply isn't trivial: 5V doesn't give you the rail headroom you need for useful output swing into 250Ω. That forced the power architecture into a more interesting shape than a simpler headphone, which was a very pleasant technical challenge.

---

## System architecture

At the top level the design splits into three concerns: power, digital audio routing, and the analog output stage. They're somewhat interdependent, which is most of the interesting story.

[*Insert block diagram here*]

**USB audio path:** USB-C receptacle → XMOS → I2S → DAC

**Power:** USB-C (5V/3A via CC resistors) → boost converter (+rail) + inverting converter (−rail) → LDO per rail → analog supply

The XMOS sits in the middle of the digital audio section and does most of the work. It handles USB device enumeration, UAC 2.0 streaming, and passes I2S downstream to the DAC. I chose it over a fixed-function USB audio bridge IC primarily because the programmable architecture means future additions like Bluetooth input and EQ processing are firmware and peripheral changes rather than a conceptual redesign. When you are working on projects in your limited free time, avoiding signal chain redesigns from the ground up is really helpful.  

The power path is where the design got interesting. I wanted to keep the supply simple with no PD controller IC, no firmware negotiating voltages at startup, and I wanted to avoid wall warts or batteries, so the input is fixed at 5V from any standard USB-C port. The problem is 5V alone isn't enough rail headroom to swing the voltage you need into a 250Ω load at useful listening levels. The math isn't subtle: to hit 200 mW into 250Ω you need about 7 V RMS, which means you need rails well beyond ±5V to stay out of clipping with any headroom to spare.

So there are two DC/DC converters stepping the 5V bus up to a split rail — one boost for the positive side, one inverting converter for the negative, each followed by a dedicated LDO. The converters handle the heavy lifting efficiently; the LDOs clean up the switching noise before it reaches anything analog.

Power sequencing on split rails is also a real concern. If the positive and negative rails don't come up together (hint: they won't) the output stage briefly sees an asymmetric supply and can put DC voltage on the headphone output. The solution is a muting stage that holds the output disconnected until both rails are stable, which is worth designing in from the start rather than kluging in later.

---

## Component selection

### XMOS processor

This ended up being the most interesting component selection in the design, and it simplified the architecture considerably. I originally had a separate USB audio bridge IC and a microcontroller in the block diagram. The XMOS consolidates both. It's a multi-core processor built specifically for deterministic audio processing, and it handles UAC 2.0 USB enumeration and I2S output to the DAC entirely in firmware.  It is the industry standard for high end audio.  

The UAC 2.0 constraint was non-negotiable. UAC 1.0 limits you to 96 kHz at 24-bit, and plenty of cheap chips support it. UAC 2.0, which gets you 192 kHz and proper high-res audio class support on modern operating systems, is where the options narrow fast. [Summivox documented this problem well](https://summivox.wordpress.com/2017/02/23/making-myself-a-usb-dac-headphone-amp-usb-interface/) in a similar project, and the XMOS family was one of the few clean solutions then too.  This was also selected since it is basically the highest quality USB audio that doesn't require an OS driver, which is important because I don't want to have to write one.  

The programmable architecture is also why Bluetooth and EQ mode selection are viable future additions without a total redesign. Both are firmware work on the XMOS side plus a connector and module, not a fundamental change to the signal chain. The specific variant within the XMOS lineup [TBD — to be confirmed during firmware development] determines the number of available I2S channels and maximum sample rate.

### DAC IC

[Why you chose your DAC. What you evaluated. What you gave up.]

I spent longer on this than I expected. The headline specs on most modern DAC chips are good enough that the differentiators end up being: I2S interface compatibility with your bridge chip, available package (nothing too small to hand-solder for a prototype), and how many external components the reference design actually requires.

### Bluetooth module

I wanted to do my own RF design, since I have never done it, and there are some cool low cost VNAs on the market.  Unfortunately, finding a module that has professional documentation that is available to a non-business individual has been problematic, so it is being deferred.  

### Output stage

Class A topology — to be finalized after protoboard validation.

The binding requirements from the T90: enough voltage swing for 200 mW into 250Ω (roughly ±7V at the output), and output impedance under 1Ω. The sub-1Ω Zout target isn't just about the T90, but also making sure the output impedance doesn't interact with the impedance curve of whatever headphone is plugged in, in case I get different headphones in the future.  

Class A is the right choice for a first revision. It's well-understood, has no crossover distortion, and the design path from schematic to working board is predictable. The obvious downside is quiescent dissipation.  A Class A output stage burns power at idle regardless of signal level, which is heat that has to go somewhere on a board that already has two LDOs. A more efficient topology is on the future work list, but that evaluation needs to happen carefully. The common alternatives each have their own tradeoffs in a headphone amp context that aren't worth rushing.

The same output stage also needs to handle low-impedance loads like 32Ω IEMs without instability. That's a different constraint, current-limited rather than voltage-limited, and the dissipation math changes significantly.

### Power architecture

The input is 5V/3A from USB-C, advertised via CC resistors: no PD controller, no firmware, no negotiation.  This keeps the power input circuit simple and means the board works with any USB-C port that can do 15W, which (should be) most of them.

The tradeoff is that 5V is the ceiling, and as covered above, 5V isn't enough on its own for the output stage. So the 5V bus feeds two DC/DC converters: a boost for the positive rail, an inverting topology for the negative. [Converter choices and target voltages to be filled in.] Each converter output then feeds a dedicated LDO: one for the positive analog rail, one for the negative.

The LDO on each side is doing legit work here, not just acting as a voltage reference. Switching converters are noisy by nature; the LDO's job is to reject that noise before it reaches the DAC or the output stage. Choosing LDOs with excellent PSRR in the 100 kHz–1 MHz range (where the converter harmonics land) matters more than the headline dropout voltage spec.

---

## PCB design

[This section gets filled in once layout is underway / complete.]

Four layers: signal, ground, power, signal. The ground plane is continuous under the analog section — no splits except where the power supply return currents actually warrant it.

Things I got wrong on the first pass:
- [TBD]

Things I'm glad I did:
- [TBD]

[*Insert board render / layout screenshot here*]

---

## Bring-up

[To be written after first power-on.]

The short version of how I approach a new board: I always check continuity does not exist between important nets (thank you to my poor classmate who had that issue on a final project), then power comes up in stages. Check quiescent currents before anything is connected to I2S. Then bring up the digital section. Then introduce audio. Then measure everything before trusting anything.

Scope captures and audio analyzer output live in `testing/results/`.

---

## Results

[To be filled in after measurements.]

Planned measurements: output noise floor, THD+N vs. frequency, crosstalk (L/R), frequency response, PSRR of the analog rail.

---

## What I'd do differently

[To be written at the end of the project.]

There is always something.

---

## Resources / references

- [Summivox USB DAC writeup](https://summivox.wordpress.com/2017/02/23/making-myself-a-usb-dac-headphone-amp-usb-interface/) — good companion read on the USB audio bridge problem
- [Any app notes, datasheets, or posts that influenced your design]
- Full project files on GitHub: [link]

---

*Lucas is an electrical engineer based in Grand Rapids, MI. He works on power electronics and embedded hardware. More projects at [link].*

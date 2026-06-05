Pi Pico Rx Transceiver Experiments
----------------------------------

This page documents a series of experiments investigating the addition of transmit capability to the Pi Pico RX receiver platform.

Some time ago I developed a standalone transmitter based on similar techniques and hardware. While it appeared feasible to combine this transmitter with the Pi Pico RX to create a simple transceiver, the resulting spurious performance was not satisfactory. Although the concept worked, unwanted emissions and spectral purity became significant concerns that needed to be addressed before the approach could be considered practical.

The primary goal of these experiments is therefore to investigate methods of improving spurious performance and to assess the overall viability of building a simple, low-cost transceiver around the Pi Pico RX architecture.

A number of different approaches have been explored. Some experiments focus on incremental improvements to the original design, while others revisit the problem from first principles and investigate entirely different transmitter architectures. Along the way I have also examined a range of more advanced transmit features, including speech processing and Controlled Envelope Single Sideband (CESSB), to better understand what level of performance can realistically be achieved using the RP2040/RP2350 platform.

The results presented here are not intended as a finished design. Rather, they document an ongoing engineering investigation into the capabilities and limitations of software-defined transmit techniques on the Raspberry Pi Pico, highlighting both successful approaches and ideas that proved less effective than expected.


Caution
=======

This project is an experimental research effort and should be viewed in the spirit of scientific exploration rather than as a finished design.

The material presented here documents investigations, prototypes, measurements, successes, and failures as I explore the possibility of adding transmit capability to the Pi Pico RX platform. Many of the techniques and circuits described are works in progress and may change substantially as the project evolves.

No guarantees are made regarding performance, spectral purity, regulatory compliance, or suitability for on-air operation. In particular, some experimental transmitter configurations may produce unacceptable levels of spurious emissions or other unwanted signals. Anyone wishing to reproduce these experiments is responsible for verifying the performance of their own equipment and ensuring compliance with applicable regulations before transmitting.

This project should not be considered a construction guide, product, or recommendation for practical use. Designs, software, hardware configurations, and conclusions may change significantly as new ideas are tested and better approaches are discovered.

Finally, while I am happy to share my findings, I am unable to provide ongoing support for individual builds, hardware modifications, bug fixes, troubleshooting, or feature requests. The information is published primarily as a record of my own experiments and to encourage further investigation by others with similar interests.


Polar vs IQ Modulation
======================

Two primary transmit architectures are being explored in these experiments. Both are intended to assess different trade-offs in complexity, spectral performance, and flexibility when implemented on the Pi Pico RX platform.

Polar Modulated Transmitter (Original Approach)
'''''''''''''''''''''''''''''''''''''''''''''''

The first approach builds on a previously developed standalone transmitter design based on polar modulation. In this architecture, RF generation is handled using a PIO-based numerically controlled oscillator (NCO), with phase modulation applied directly in the digital domain. The amplitude component is then applied separately using a PWM-based output stage to modulate the RF envelope.

This separation of phase and amplitude allows for a highly flexible implementation, but also introduces challenges in maintaining spectral purity, particularly with regard to spurious emissions and unwanted mixing products.

For a complete understanding of the technique, refer to the `original design. <https://101-things.readthedocs.io/en/latest/ham_transmitter.html>`_

QSE-Based Modulator (Conventional Approach)
'''''''''''''''''''''''''''''''''''''''''''

In parallel, a more traditional quadrature sampling exciter (QSE) approach is being investigated. This method uses separate I and Q “audio” paths generated via PWM outputs, which are then fed into a QSE stage. The architecture is conceptually similar to the QSE implementation used in the Pi Pico RX receiver.

This approach is more conventional in SDR design and is expected to provide a more predictable and well-understood signal path. While it may be less exotic than the polar modulation method, it offers a useful fallback option if acceptable performance cannot be achieved with the polar architecture.

In addition, a QSE-based transmitter may ultimately be better suited to a more feature-rich or “bells and whistles” transceiver design, where flexibility and extensibility are more important than absolute minimalism.

Internal vs External NCO
========================

For both the polar-modulated transmitter and the QSE-based transmitter, two oscillator implementation options are being explored: an internal numerically controlled oscillator (NCO) implemented on the RP2040/RP2350, and an external oscillator based on an Si5351 clock generator.

This dual approach is intended to provide flexibility in evaluating trade-offs between integration, phase control, spectral performance, and system complexity.

Across these two modulation architectures and two oscillator options, there are therefore four distinct experimental configurations in total:

+ Polar modulation with internal NCO
+ Polar modulation with external Si5351
+ QSE-based modulation with internal LO
+ QSE-based modulation with external Si5351

These four variants form the core design space being explored in this project.

Polar Modulated Transmitter
'''''''''''''''''''''''''''

In the polar architecture, the internal NCO approach closely follows the original transmitter concept. The RF carrier is generated directly on-chip using a PIO-based oscillator, with phase control applied in software. This allows tight coupling between the modulation process and the carrier generation, enabling fine-grained phase manipulation and rapid updates.

An alternative implementation uses an external Si5351 clock generator. In this configuration, achieving phase modulation requires a different strategy: instead of directly controlling a digital NCO, the system must apply rapid frequency updates to the Si5351 over SPI in order to indirectly modulate phase. This approach is conceptually similar to the technique used in the uSDX transceiver architecture, a novel and disruptive approach where fast frequency stepping is used to approximate continuous phase modulation.

Reference to uSDX-style frequency update approach by Guido (PE1NNZ):
`<https://github.com/threeme3/usdx>`_

While this method introduces additional more complex hardware and additional constraints compared to an internal NCO, it may offer improved spectral performance.

QSE-Based Transmitter
'''''''''''''''''''''

The QSE-based architecture similarly supports both internal and external local oscillator options. In the Pi Pico RX receiver, a quadrature local oscillator can already be generated internally using PIO-based techniques or provided externally using an Si5351 clock generator.

In this transmitter variant, the same quadrature local oscillator is reused to drive the QSE stage directly, ensuring phase coherence between the I/Q paths and the RF modulation process. This reuse of the existing LO infrastructure simplifies the overall system design and maintains consistency with the receiver architecture.

As with the polar implementation, the choice between internal and external oscillator sources represents a key design trade-off between integration and flexibility, and both options are being evaluated as part of the experimental process.

Polar Test Bed
==============

The polar modulation experiments are currently being carried out using a simplified hardware test-bed. The primary focus at this stage is not on producing a complete or transmission-ready system, but rather on characterising the behaviour of the polar modulator itself and comparing its performance against alternative architectures.

As a result, the design intentionally omits elements such as low-pass filtering and RF power amplification. Instead, an analog switch is used as a behavioural stand-in for a polar-modulated RF output stage. This allows the core modulation dynamics and switching behaviour to be evaluated in isolation, without the additional variables introduced by a full RF chain.

A microphone input is provided using one of the spare ADC channels on the Pi Pico, enabling basic audio modulation experiments. In addition, two GPIO pins are used as digital inputs for a simple CW keyer, providing separate dit and dah inputs. These inputs also double as a PTT control mechanism, reducing overall pin count while maintaining functional flexibility.

A single RX/TX control GPIO is used to switch external hardware between receive and transmit modes. This provides a minimal but practical interface for coordinating external RF circuitry during experimentation.

The test-bed is designed to support both internal and external NCO configurations. In addition to the on-chip PIO-based oscillator, provision is included for an external Si5351 module, allowing direct comparison between internally generated RF carriers and externally synthesised clock sources under identical modulation conditions.

The overall hardware configuration is illustrated below:

.. image:: images/transmitter_experiments/polar.svg

This simplified arrangement allows rapid iteration on modulation algorithms and signal generation techniques, while deferring RF chain complexity until the behaviour of the core modulator is better understood.

Polar Concept Internal Oscillator
=================================

A significant portion of the development effort has focused on improving the spectral purity of the internal local oscillator implementation. The original design exhibited clearly visible spurious components, particularly when operating the polar modulator with a software-driven NCO. Several architectural and algorithmic changes have been introduced to address these issues.

1. Increasing Phase Update Rate and Interpolation

In the original implementation, the oscillator phase was updated abruptly at the audio sample rate (approximately 10 kHz). This resulted in discrete phase steps applied at regular intervals, which in turn produced strong spurious components spaced at 10 kHz offsets from the carrier. Because these spurs occur very close to the fundamental, they are difficult or impossible to remove using conventional analogue filtering.

The first and most significant improvement was to smooth these transitions by interpolating phase updates in software. Instead of applying a single abrupt phase correction per audio sample, the phase is now slewed gradually over the course of each sample. This effectively increases the internal phase update rate to approximately 500 kHz, allowing the local oscillator phase to evolve smoothly rather than discontinuously. The result is a substantial reduction in close-in spurious energy.

2. Migration from PIO to HSTX Output (Pi Pico 2 / RP2350)

The second major change is the migration from a PIO-based output system to the HSTX peripheral available on the RP2350 (Pi Pico 2). HSTX supports double data rate (DDR) operation, allowing a different value to be output on both the rising and falling edges of the clock. This effectively doubles the usable output rate from approximately 150 MHz to 300 MHz and reduces worst-case timing resolution from ~8 ns to ~4 ns.

Functionally, HSTX can be used in a very similar way to PIO for waveform generation. Precomputed waveform tables are stored in memory and streamed via DMA into the HSTX unit, effectively “playing back” the waveform at the desired carrier frequency. With HSTX, the overall architecture remains largely unchanged compared to the PIO implementation; however, the same waveform generation and DMA scheduling techniques are now applied to the higher-speed output engine.

One important consequence of this change is hardware dependency. Transmit functionality using this approach is only viable on the Pi Pico 2 (RP2350), and the set of usable GPIO pins is constrained to those that support HSTX functionality.

3. Adaptive Clock Frequency Selection

The third improvement leverages an adaptive system clock strategy, already used in the Pi Pico RX design to improve local oscillator frequency resolution. The same concept has been extended to the transmitter.

The key idea is to select the system clock frequency that most closely aligns with an integer multiple of the desired NCO frequency. By doing so, the number of fractional clock corrections required in the waveform generation process is reduced. This minimises the occurrence of “one-clock adjustments” in the output stream, thus reducing the total spurious spectral energy.

.. figure:: images/transmitter_experiments/adaptive_clocks.png

    The figure shows the closest exactly achievable clock for each tuning frequency.

4. Phase Dither to Suppress Periodic Spurs

The fourth improvement introduces phase dithering to break up deterministic error patterns in the NCO update process.

In the original system, small correction steps (“one-clock adjustments”) occur in a repeating pattern. This can be understood intuitively using a calendar analogy: if a small timing error is corrected in a perfectly regular cycle, it behaves like a periodic event—much like a leap year correction that occurs predictably every four years. While the average timing remains correct, the periodic nature of the correction introduces a new spectral component, which appears as discrete spurious tones.

To address this, a controlled amount of random dither is introduced into the phase update process. Instead of following a strictly repeating correction pattern, the timing adjustments are decorrelated and distributed pseudo-randomly over time. This transforms what would otherwise be discrete spectral spurs into broadband noise.

Because this noise is spread across a wide frequency range, its spectral density is significantly lower than the equivalent spur power. Furthermore, a substantial portion of this noise is removed by the transmitter’s output filtering stages, making it far less problematic in practice than deterministic spurious tones.

Overall, the combination of interpolation, higher-speed output hardware, adaptive clock selection, and phase dithering results in a marked improvement in spectral purity compared to the original implementation.

Polar Concept External Oscillator
=================================

An alternative transmitter implementation has been developed using an external Si5351 clock generator. In this configuration, the third output of the Si5351 is dedicated to the transmit signal path, while outputs 1 and 2 remain reserved for the quadrature local oscillator used by the Pi Pico RX receiver. This allows the receiver and transmitter subsystems to coexist while sharing a common frequency reference source.

To support this approach, the existing Si5351 library has been extended with additional functionality to enable rapid, small-step frequency updates. These updates are intended to act as the primary modulation mechanism for transmission. (The code used for these modifications is documented separately and can be referenced alongside this section.)

Modulation Principle
''''''''''''''''''''

Unlike the internal NCO-based approach, which directly manipulates oscillator phase, the Si5351 method operates by dynamically adjusting the output frequency in small increments. These updates are applied continuously at the audio sample rate, effectively encoding the modulation information as short-term frequency deviations.

Because phase is the integral of frequency, this approach implicitly generates the desired phase trajectory over time. However, it is important to note that phase is not directly controlled, and instead emerges from the accumulated effect of frequency changes.

Initial Design Concerns
'''''''''''''''''''''''

Several potential limitations were identified early in the design process:

1. Phase drift and lack of absolute phase control
Since frequency updates are not synchronised to a global phase reference, the absolute phase of the transmitted signal can drift over time. Unlike a true phase accumulator (as used in an NCO), there is no direct mechanism to guarantee phase continuity across updates.

2. I2C bandwidth limitations
The Si5351 is controlled via I2C, which imposes a hard limit on the maximum achievable update rate. This constrains how finely the modulation can be sampled and potentially limits the fidelity of rapid waveform changes.

3. Non-atomic register updates
Frequency configuration in the Si5351 is distributed across multiple registers. These are not updated atomically, meaning that during an update sequence the device may briefly pass through intermediate and potentially incorrect frequency states. These transient states could, in principle, introduce short spurious artefacts.

Implementation Mitigations
''''''''''''''''''''''''''

To address these concerns, several practical optimisations were introduced. The I2C bus was operated at a high speed of 800 kHz, and the modulation sample rate was reduced to approximately 8 kHz to ensure reliable update timing and reduce bus contention.

Initially, results from this configuration were somewhat disappointing, with reduced clarity in the transmitted signal. However, it was found that improving the quality of the audio input path had a significant impact on overall performance. In particular, increasing ADC resolution through oversampling resulted in a much clearer signal.

Results and Conclusion
''''''''''''''''''''''

My initial concerns regarding phase coherence, update latency, and non-atomic register writes proved to be unfounded. The Si5351-based approach ultimately produced surprisingly good results. With appropriate tuning of update rate, I2C speed, and audio signal conditioning, the system was able to generate audio-quality transmissions comparable to the other experimental architectures under consideration.

While the Si5351 method is fundamentally different from the internal NCO approach, it has proven to be a viable alternative for practical experimentation and may offer a useful balance between simplicity, flexibility, and performance in certain configurations.

QSE Test Bed
============

The quadrature sampling exciter (QSE) variant of the transmitter is implemented using a hardware configuration that closely leverages components already present in the Pi Pico RX design. One of the key advantages of this approach is that the existing 74CBLVT3253 analogue multiplexer already includes a spare 4:1 channel, which can be repurposed to implement a QSE stage with minimal additional external hardware.

This allows the QSE transmitter to be integrated into the existing receiver architecture with only modest extensions, making it a particularly attractive “low overhead” experimental path.

I/Q Signal Generation
'''''''''''''''''''''

The in-phase (I) and quadrature (Q) baseband signals are generated using PWM outputs from the Pi Pico. These PWM signals are used as audio-frequency representations of the modulation waveform.

In many conventional QSE implementations, the required phase inversion is typically achieved using op-amp based inverting and non-inverting stages to produce accurate I, Q, −I, and −Q signals. In this design, that function is instead implemented digitally using a 74LV04 inverter.

By using the same logic physical device to generate both the normal and inverted signals, the design ensures that each pair of complementary signals is produced under closely matched conditions. This reduces the opportunity for I/Q imbalance introduced by analogue component mismatch or op-amp non-linearities.

PWM Generation, Interpolation, and Noise Shaping
''''''''''''''''''''''''''''''''''''''''''''''''

Each of the four PWM-derived audio signals (I, Q, and their inverted counterparts) is passed through a simple RC low-pass filter. The PWM frequency is chosen to be sufficiently high that the resulting filter can be relatively narrow while still achieving strong attenuation of switching noise.

To further improve baseband signal quality, the PWM generation process also incorporates interpolation and noise shaping. Interpolation is used to smooth transitions between successive audio samples, reducing abrupt quantisation steps in the reconstructed waveform. Noise shaping is then applied to spectrally push quantisation error out of the most critical in-band region, concentrating residual error at higher frequencies where it is more effectively attenuated by the analogue RC filtering.

This combination of high PWM carrier frequency, interpolation, noise shaping, and tight analogue filtering is intended to maximise rejection of PWM ripple. This is particularly important because any residual PWM energy, when mixed up to RF frequencies in the QSE stage, would manifest as unwanted spurious components in the transmitted spectrum.

System Architecture
'''''''''''''''''''

Aside from the baseband generation and QSE-specific hardware, the overall architecture closely mirrors the polar modulation test-bed. The same provisions are included for microphone input via ADC, and for CW keying via dual dit/dah GPIO inputs, which also double as PTT control signals. A single RX/TX control GPIO is again used to switch external RF hardware between transmit and receive modes.

As with the polar design, no RF low-pass filtering or power amplification stages are included at this stage. The focus remains on evaluating the behaviour of the modulation and mixing stages in isolation.

The overall hardware configuration is illustrated below:

.. image:: images/transmitter_experiments/polar.svg

This configuration provides a compact and hardware-efficient route to evaluating QSE performance on the Pi Pico platform, while maintaining strong compatibility with the existing receiver architecture.

Comparison of the Four Transmitter Configurations
=================================================

The four transmitter variants can be summarised more simply by grouping their key trade-offs in terms of audio quality, spectral performance, amplifier requirements, and hardware complexity.

Overall, the designs fall into two broad categories:

Polar modulation: compatible with efficient switching (Class-E) power amplification, but more sensitive to non-linearities in phase and amplitude generation.
QSE modulation: produces cleaner audio and fewer modulation artefacts, but requires a linear RF power amplifier.

A further division exists between internal NCO and external Si5351 implementations, which primarily affects spurious performance and hardware complexity.

Summary Table
'''''''''''''

+----------------------+-------------------------------------+-------------------------------------+----------------------------+------------+-----------------------------------------------------------------------+
| Configuration        | Audio Quality                       | Spurious Performance                | PA Requirement             | Complexity | Key Characteristics                                                   |
+======================+=====================================+=====================================+============================+============+=======================================================================+
| Polar + Internal NCO | Good                                | Moderate (meets limits, unverified) | Efficient Class-E possible | Low        | Best balance of simplicity and performance                            |
+----------------------+-------------------------------------+-------------------------------------+----------------------------+------------+-----------------------------------------------------------------------+
| Polar + Si5351       | Good (slightly reduced vs internal) | Better than internal NCO            | Efficient Class-E possible | Medium     | Improved spurious behaviour but limited by I2C update constraints     |
+----------------------+-------------------------------------+-------------------------------------+----------------------------+------------+-----------------------------------------------------------------------+
| QSE + Internal LO    | Good                                | Poor (does not meet limits)         | Linear amplifier required  | Low        | Clean audio but spurious performance limited by internal LO behaviour |
+----------------------+-------------------------------------+-------------------------------------+----------------------------+------------+-----------------------------------------------------------------------+
| QSE + Si5351         | Good                                | Best overall                        | Linear amplifier required  | Medium     | Cleanest spectrum and best audio quality; most “SDR-like” behaviour   |
+----------------------+-------------------------------------+-------------------------------------+----------------------------+------------+-----------------------------------------------------------------------+

Key Observations
''''''''''''''''

The results highlight several consistent trends:

The QSE architectures consistently produce better audio quality than the polar approaches, with fewer perceptible artefacts and reduced modulation distortion.
The polar implementations exhibit a small amount of splatter, likely arising from non-linearities in phase and amplitude interaction. This is not observed in the QSE versions.
In both architectures, the Si5351 improves spurious performance compared to the internal oscillator, despite its bandwidth and update-rate limitations.
Between the two polar variants, the internal oscillator produces slightly better audio quality than the Si5351-based approach, although both remain acceptable for experimental use.
The QSE + internal oscillator variant performs least well overall in terms of spectral purity, and is unlikely to be suitable for further development.
The QSE + Si5351 variant provides the best overall signal quality, but at the cost of requiring a linear RF power amplifier.
The polar + internal NCO variant offers the best overall balance, combining low complexity, efficient Class-E compatibility, and acceptable (if not perfect) spectral performance.

SDR Reception Comparison Videos
'''''''''''''''''''''''''''''''

To complement the static analysis above, example over-the-air reception recordings have been captured using an SDR receiver. These illustrate the real-world spectral behaviour and perceived audio quality of each transmitter configuration under comparable conditions.

**Polar + Internal NCO (SDR reception video)**

.. raw:: html

    <iframe width="560" height="315" src="https://www.youtube.com/embed/8QQ-m0NK7xA?si=xPQ2uln3I8yRApIc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

**Polar + Si5351 (SDR reception video)**

.. raw:: html

    <iframe width="560" height="315" src="https://www.youtube.com/embed/wNny0XqQmVQ?si=b2XnpOx8p8PoY9Bk" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

**QSE + Internal LO (SDR reception video)**

.. raw:: html

    <iframe width="560" height="315" src="https://www.youtube.com/embed/Y8t6osKoMVM?si=ly03Gs4XYwbA_-9M" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

**QSE + Si5351 (SDR reception video)**

.. raw:: html

    <iframe width="560" height="315" src="https://www.youtube.com/embed/ZQKlAK6OwqE?si=bv7ib8PiQNZwTil2" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

These recordings provide a practical comparison of how each architecture performs when observed through a typical SDR receiver chain, including the combined effects of modulation quality, spectral purity, and real-world RF impairments.

Conclusion
''''''''''

Of the four configurations, three remain strong candidates for further development:

+ Polar + Internal NCO: best balance of simplicity, efficiency, and acceptable performance
+ Polar + Si5351: improved spectral purity with moderate complexity
+ QSE + Si5351: highest signal quality but requires linear amplification

The QSE + internal NCO configuration is unlikely to be pursued further due to its inferior spurious performance.

In summary, the polar architectures remain attractive for efficient RF power stages and minimal hardware, while the QSE architectures provide superior signal fidelity at the cost of amplifier efficiency. The choice between them ultimately depends on whether efficiency or signal purity is the primary design goal.


Modes
==========

CW
==

CW Keyer
Pulse Shaping

Speech Processor
================

+ Mic Gain
+ Noise Gate
+ Equilisation - include link to Bob Heil
+ Compression - impreve peak to average ratio

CESSB
=====

+ Filtering causes overshoot
+ Clipping causes frequency splatter
+ "More than cluipping"
+ Pico Rx Approach

Speech processor results
========================


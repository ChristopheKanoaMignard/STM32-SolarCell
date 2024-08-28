## Overview
Characterizing a solar cell.
Boilerplate statments about board number and file structures and contents.
Change references in code to photodiode to solar cell.
## Hardware
I integrated a Sunyima monocrystaline photocell into an op-amp current sink circuit, as displayed in the image below. This assembly is thermally bonded to a block of aluminum and an NTC thermistor embedded within. This thermistor is part of a voltage divider circuit as shown below. This circuit is driven by a PWM voltage, and the resulting voltage is measured by an ADC.

[Circuit]

### Thermistor & Voltage Divider

The thermistor has a temperature-dependent resistance. Soldering it into a voltage divider allows us to determine this resistance, $R_{therm}$, based on the signal voltage: which is converted by the MCU ADC. This, in turn, allows us to determine the temperature. 

 Using the voltage divider formula and rearranging to solve for the thermistor resistance produces:
$$R_{therm} = \frac{R_1}{^{V_{cc}}/_{V_{sig}} - 1}.$$

The STM32F726G evaluation board as a 12-bit ADC with a high pin voltage of $3.3 V$, so the ratio between signal voltage per ADC value and their maximums is $\frac{V_{sig}}{ADC_{sig}} = \frac{V_{cc}}{ADC_{max}} =\frac{3.3V}{4095bits}.$ Rearranging for the ratio of voltages and substituting yeilds the formula:
$$R_{therm} = \frac{R_1}{\frac{{ADC_{max}}}{{ADC_{sig}}} - 1 }.$$

To convert this resistance to temperature we can use the Steinhart-Hart equation $R_{therm}(T) = R_{therm}(T_0)e^{β(\frac{1}{T}-\frac{1}{T_0})},$ where $R_{therm}(298K)=10kΩ$ and $β = 3650K.$ By rearranging for $T$ and substituting the expression for &R_{therm}$ above, we derive a final equation for the temperature of the thermistor. 

$$\begin{aligned} 
R_{therm} &= R_{therm}(T_0)e^{β(\frac{1}{T}-\frac{1}{T_0})}\\
ln(R_{therm}) 	&= ln\left( R_{therm}(T_0)\right)* β \left(\frac{1}{T}-\frac{1}{T_0} \right)\\
\frac{1}{T}   -   \frac{1}{T_0}    &=   \frac{1}{β}   ln   \left(   \frac{R_{therm}}{R_{therm}(T_0)}   \right)\\
 T &= \left(   \frac{1}{β} ln\left(   \frac{R_{therm}}{R_{therm}(T_0)}   \right)  +\frac{1}{T_0} \right)^{-1}\\
 &= \left(   \frac{1}{β} ln\left(   \frac{ \frac{10kΩ}{   \frac{{ADC_{max}}}{{ADC_{sig}}}   - 1 } }   {10kΩ}   \right)  +\frac{1}{T_0} \right)^{-1}\\
 &=\left(   -\frac{1}{β} ln\left(   \frac{{ADC_{max}}}{{ADC_{sig}}}    - 1 \right)  +\frac{1}{T_0} \right)^{-1}\\
 &= \left(   -\frac{1}{4095K} ln\left(   \frac{4095.001}{{ADC_{sig}}}    - 1 \right)  +\frac{1}{298K} \right)^{-1}.
\end{aligned}$$

Note that at the chosen reference temperature $R_1=R_{therm}(298K)$. Additionally, this formula provides temperature in Kelvin; when implemented in code, simply add $273$ to convert to degrees Celsius. Finally, the code accounts for the singularity when $ADC_{sig}=4095$ by adding a tiny value to $ADC_{max}$. 

### PWM

The average voltage produced by our PWM is $V_{pwm} = duty * 3.3V.$ However, because of the voltage divider, the voltage at the op-amp's positive terminal is 
$$V_{op+} = V_{pwm} * \frac{R_2}{R_3+R_2}=0.033V_{pwm}. $$
The op-amp will force $V_{op+} = V_{op-},$ so the current flowing through $R_1,$ and therefore out of the solar cell, is
$$
I = \frac{V_{op+}} {R_1} = duty*0.11A.
$$ 
With this formula, we are able to control the current by adjusting the duty cycle. While this is deceptively simple, several complications arose that provided good learning opportunities. First 

## Data and Analysis
First we want to confirm that the solar cell is working as expected. The solar cell should have a maximum $2.3V$ open-circuit voltage which should also be temperature-dependent. I exposed the solar cell to an incandescent light and ramped the current between $0A$ and $1.2A$ using a PWM peripheral. As seen below, we confirmed that the I-V curve looks exactly was we expected. 

[IV curve gif]

An interesting issue arose when I looked at the voltage with an oscilloscope. The voltage was spiking when the the PWM signal changed states, clearly indicating some sort of coupling. The coupling was produced because I had set the timer frequency too slow: within an order of magnitude of the cutoff frequency. 

[Schematic diagram]

The equivalent resistance of this circuit is $3.2kΩ,$ so the cutoff frequency is $50Hz.$ Therefore the PWM frequency ought to be at least a couple orders of magnitude larger. 

## Complications and Learning Opportunities




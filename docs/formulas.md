# Formulas and Calculations

This document lists the formulas used in the EnviroTrack firmware and forecasting model, with links to the code that implements them.

| # | Topic | Used in |
|---|---|---|
| 1 | [SDS011 data frame](#1-sds011-data-frame) | `sds011.py` |
| 2 | [SDS011 checksum](#2-sds011-checksum) | `sds011.py` |
| 3 | [Calibration](#3-calibration) | `loop_sds011.py` |
| 4 | [Voltage measurement](#4-voltage-measurement) | `loop_sds011.py` |
| 5 | [LED matrix addressing](#5-led-matrix-addressing) | `ledpm25.py` |
| 6 | [Display timing](#6-display-timing) | `ledpm25.py` |
| 7 | [LED matrix current](#7-led-matrix-current) | hardware design |
| 8 | [Rolling average](#8-rolling-average) | notebook |
| 9 | [Normalization and sequences](#9-normalization-and-sequences) | notebook |
| 10 | [CNN layer shapes and parameters](#10-cnn-layer-shapes-and-parameters) | notebook |
| 11 | [Loss and error](#11-loss-and-error) | notebook |

## 1. SDS011 data frame

The SDS011 answers a query with a 10-byte frame:

```
AA C0 D1 D2 D3 D4 D5 D6 CS AB
```

D1-D2 hold PM2.5 (low byte, high byte), D3-D4 hold PM10, D5-D6 the sensor ID and CS the checksum. The concentrations are

```math
\mathrm{PM}_{2.5} = \frac{256\,D_2 + D_1}{10}, \qquad \mathrm{PM}_{10} = \frac{256\,D_4 + D_3}{10} \qquad [\mu\mathrm{g/m^3}]
```

In [`sds011.py` L94-L96](../firmware/sds011.py#L94-L96) the frame is unpacked with `struct.unpack('<HHBBBs', packet)`, which reads each pair as a little-endian 16-bit word, and the result is divided by 10.

Example: D1 = 0x2F (47) and D2 = 0x01 give PM2.5 = (256 + 47) / 10 = 30.3 µg/m³.

## 2. SDS011 checksum

```math
CS = \Big(\sum_{k=1}^{6} D_k\Big) \bmod 256
```

After unpacking, the driver has the 16-bit words w25 = 256 D2 + D1 and w10 = 256 D4 + D3. Since 256 D2 and 256 D4 vanish modulo 256, the high bytes are added back with a shift ([`sds011.py` L97](../firmware/sds011.py#L97)):

```math
\Big(w_{25} + w_{10} + D_5 + D_6 + \big\lfloor w_{25}/256 \big\rfloor + \big\lfloor w_{10}/256 \big\rfloor\Big) \bmod 256 \;=\; \Big(\sum_{k=1}^{6} D_k\Big) \bmod 256
```

Commands sent to the sensor are 19 bytes long: `AA B4`, 15 data bytes, checksum and `AB`. The data bytes are the command, mode and parameter, ten zero bytes and the target sensor ID `FF FF` (any sensor), so the checksum is ([`sds011.py` L64](../firmware/sds011.py#L64)):

```math
CS_{\text{cmd}} = (\text{cmd} + \text{mode} + \text{param} + 255 + 255) \bmod 256
```

For the wake command (cmd 6, mode 1, param 1): (6 + 1 + 1 + 510) mod 256 = 6.

## 3. Calibration

Each reading is corrected with an offset b that can be set remotely with the `Setb=` command ([`loop_sds011.py` L662-L668](../firmware/loop_sds011.py#L662-L668)):

```math
y = x + b
```

PM values are rounded to 2 decimal places, temperature and humidity to 1.

## 4. Voltage measurement

The supply (GPIO36) and battery (GPIO35) voltages pass through on-board voltage dividers that halve them before the ADC ([`loop_sds011.py` L612-L613](../firmware/loop_sds011.py#L612-L613)):

```math
V = 2 \times \frac{u}{10^{6}}
```

where u is the pin voltage in microvolts returned by `ADC.read_uv()`. Example: u = 1,950,000 µV gives V = 3.90 V.

## 5. LED matrix addressing

The 1024 LEDs form one chain that runs through the matrix column by column in a zigzag. For column x (0-63) and row y (0-15, top to bottom), the index in the chain is ([`ledpm25.py` L27-L36](../firmware/ledpm25.py#L27-L36)):

```math
i(x, y) = \begin{cases} 16x + y, & x \text{ even} \\ 16x + 15 - y, & x \text{ odd} \end{cases}
```

Examples: (0, 15) is LED 15 and (1, 15) is LED 16, which are neighbours in the chain; (63, 0) is the last LED, 1023.

## 6. Display timing

Characters are placed 8 pixels apart. The text starts at x = 64 and moves one pixel left per frame while x > -8n ([`ledpm25.py` L118-L128](../firmware/ledpm25.py#L118-L128)), so an n-character message needs

```math
F = 64 + 8n
```

frames. Each `np.write()` transmits 24 bits per LED at 800 kHz (1.25 µs per bit):

```math
t_{\text{write}} = 1024 \times 24 \times 1.25\ \mu\text{s} = 30.72\ \text{ms}
```

With the 20 ms delay per frame, the scroll time is at least

```math
T_{\text{scroll}} \ge F \cdot (t_{\text{write}} + 20\ \text{ms})
```

For the message `PM25:30.4 PM10:31.2 TEMP:29.1 HUMI:58.2` (n = 39): F = 376 and T ≥ 376 x 50.72 ms ≈ 19.1 s.

The SDS011 is awake for about 5 s of each cycle of at least 5 + 19.1 + 0.2 s, a duty cycle of

```math
D \le \frac{5}{5 + 19.1 + 0.2} \approx 0.21
```

## 7. LED matrix current

A WS2812B LED draws about 20 mA per colour channel at full brightness. The text colour `(255, 255, 0)` uses two channels, so

```math
I \approx N_{\text{lit}} \times 40\ \text{mA}
```

Counting the lit pixels of the message above with `font.py` over all 376 frames gives 80.5 on average and 111 at most:

| | Lit LEDs | Current | Power at 5 V |
|---|---|---|---|
| Average | 80.5 | 3.2 A | 16 W |
| Peak | 111 | 4.4 A | 22 W |

In addition, each LED draws roughly 0.5-1 mA when off (0.5-1 A for the whole matrix). With all 1024 LEDs at full white the current would be 1024 x 60 mA ≈ 61 A. Current scales roughly with the colour value, so a value of 64 instead of 255 reduces the text current by about four times.

## 8. Rolling average

Each variable is smoothed with a rolling mean over 15 samples, about 30 minutes at one reading every 2 minutes (`rolling(window=15, min_periods=1).mean()` in the notebook):

```math
\bar{x}_t = \frac{1}{m_t} \sum_{k=0}^{m_t - 1} x_{t-k}, \qquad m_t = \min(15,\ t + 1)
```

## 9. Normalization and sequences

Min-max scaling per variable (`MinMaxScaler`) and its inverse:

```math
x' = \frac{x - x_{\min}}{x_{\max} - x_{\min}}, \qquad x = x'\,(x_{\max} - x_{\min}) + x_{\min}
```

With s_t the normalized vector [PM2.5, PM10, temperature, humidity] at step t, the samples are

```math
X_i = \begin{bmatrix} s_i \\ s_{i+1} \\ s_{i+2} \end{bmatrix} \in \mathbb{R}^{3 \times 4}, \qquad y_i = s_{i+3} \in \mathbb{R}^{4}, \qquad i = 0, \dots, T-4
```

giving T - 3 samples from T time steps.

## 10. CNN layer shapes and parameters

Conv1D with kernel size k = 2 and no padding on a sequence of length 3:

```math
L_{\text{out}} = L_{\text{in}} - k + 1 = 3 - 2 + 1 = 2
```

Output of filter f (16 filters) at position t:

```math
h_{t,f} = \mathrm{ReLU}\Big(b_f + \sum_{j=0}^{1} \sum_{c=1}^{4} w_{j,c,f}\, x_{t+j,\,c}\Big), \qquad t \in \{0, 1\}
```

MaxPooling1D with pool size 2 reduces the length to floor(2 / 2) = 1, giving 1 x 16 values, flattened to 16.

Number of parameters (weights and biases):

```math
\begin{aligned}
\text{Conv1D:}\quad & (k\,C_{\text{in}} + 1)\,F = (2 \cdot 4 + 1) \cdot 16 = 144 \\
\text{Dense 10:}\quad & (16 + 1) \cdot 10 = 170 \\
\text{Dense 4:}\quad & (10 + 1) \cdot 4 = 44 \\
\text{Total:}\quad & 358
\end{aligned}
```

## 11. Loss and error

Mean squared error over N samples and 4 outputs:

```math
\mathrm{MSE} = \frac{1}{4N} \sum_{i=1}^{N} \sum_{c=1}^{4} \left(\hat{y}_{i,c} - y_{i,c}\right)^2
```

At epoch 100 the MSE on normalized data is 1.75 x 10⁻⁴ for training and 1.2 x 10⁻³ for validation:

```math
\mathrm{RMSE}_{\text{val}} = \sqrt{1.2 \times 10^{-3}} \approx 0.035
```

This corresponds to about 3.5% of each variable's range. Multiplying by (x_max - x_min) of a variable converts the error to physical units.

The training log shows 51 batches of 32 per epoch, i.e. 1,601-1,632 training samples. These are 80% of all samples, so the dataset contains about 2,000 samples, or roughly 67 hours of data at 2 minutes per reading.

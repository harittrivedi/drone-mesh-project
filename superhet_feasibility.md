# Using a Superheterodyne Transceiver to Shift Wi‑Fi Signals

## Introduction

The mesh networking system described in this project is designed around **Wi‑Fi 6 (5.8 GHz) radios** like the Qualcomm QCN9074.  It uses a standard Wi‑Fi mesh plus a low‑duty HaLow side channel for control.  One idea that occasionally comes up in amateur radio circles is to **convert a Wi‑Fi signal to operate on a completely different band** using a frequency‑conversion front end (also called a **transverter**).  By mixing the 5.8 GHz carrier down to a lower band, the network could theoretically access quieter spectrum or penetrate obstacles better.  This document explains what a superheterodyne transceiver does, examines prior work on Wi‑Fi transverters, and evaluates the feasibility of using such a design to build an IP‑based mesh network.  It also lists the major components required to implement a superheterodyne‑based frequency converter.

## What is a Superheterodyne Transceiver?

A **superheterodyne** receiver (or transceiver) converts an incoming radio signal at frequency \(f_{\text{RF}}\) to a **fixed intermediate frequency** \(f_{\text{IF}}\).  It does this by mixing the RF signal with a **local oscillator (LO)**.  The mixer produces sum and difference frequencies; the desired heterodyne occurs at \(f_{\text{RF}} \pm f_{\text{LO}}\).  The unwanted mixer products are removed by narrow band‑pass filters at the intermediate frequency.  In a typical superheterodyne architecture the antenna, RF filter and optional low‑noise amplifier select the desired band, the mixer and LO convert the signal to the fixed IF, and an IF band‑pass filter and amplifier supply most of the gain and selectivity【273655769275336†L458-L495】.  The LO is tuned such that the RF signal mixes to the IF; using low‑side injection the IF is \(f_{\text{IF}} = f_{\text{RF}} - f_{\text{LO}}\) and using high‑side injection it is \(f_{\text{IF}} = f_{\text{LO}} - f_{\text{RF}}\)【273655769275336†L520-L535】.  This architecture allows most filtering and amplification to take place at a constant frequency, making high gain and narrow filtering practical【273655769275336†L523-L546】.

### Image frequency and filtering

Because the mixer produces both sum and difference components, an unwanted **image frequency** at \(f_{\text{LO}} \pm f_{\text{IF}}\) can also mix down to the IF.  To suppress the image, the RF front end must include band‑pass filtering or an **image‑reject mixer**【273655769275336†L520-L548】.  A well‑designed superheterodyne transceiver therefore includes:

* **RF preselection filter and low‑noise amplifier** – limits the input band and provides initial gain.
* **Local oscillator** – a tunable or synthesised LO that sets the conversion frequency.
* **Mixer** – a non‑linear device that combines the RF and LO signals to produce the heterodyne.
* **Intermediate frequency (IF) filter and amplifier** – a narrow band‑pass filter at a fixed frequency and an amplifier that provides most of the gain【273655769275336†L458-L495】.
* **Demodulator/Baseband** – extracts the digital or analog information from the IF.

In a **transceiver**, additional stages such as power amplifiers and matching networks are added for transmission.  A superheterodyne transmitter uses the same principle in reverse: the baseband is modulated onto an IF, which is then mixed with an LO to create the RF carrier.

## Frequency Conversion and Transverters

A **transverter** (frequency converter) is essentially a superheterodyne transmitter/receiver pair that connects in front of an existing radio.  It mixes the outgoing and incoming signals to a different band.  For example, an amateur radio project on the Green Bay Professional Packet Radio (GBPPR) site proposed mixing **802.11 b channel 9 at 2.451 GHz** with a **1.536 GHz local oscillator**; the resulting heterodyne at **915 MHz** would be used as the new transmit/receive frequency【635212932002369†L6-L24】.  The design notes warn that several practical issues must be solved, including isolation pads between stages, mixer diplexers, phase‑locked loop (PLL) loop filter values and switching voltages【635212932002369†L27-L31】.  The author also notes that the design had not yet been completed and that there might be leakage or high‑side injection issues【635212932002369†L76-L88】.

The **High‑Speed Multimedia (HSMM) radio** community recognises that transverters can be used to move **802.11b/g/n** signals from the 2.4 GHz band to bands like 3.4 GHz, 900 MHz or 420 MHz.  More drastic frequency changes require a frequency converter; the transverter converts the 802.11 signal from its original band to a new band while preserving the modulation【609529959923763†L568-L603】.  However, regulatory restrictions and modern firmware (e.g., Linux regulatory databases) prevent casual users from operating Wi‑Fi equipment outside of authorised bands【609529959923763†L576-L596】.  In addition, the transverter must preserve the complex modulation of Wi‑Fi (OFDM), maintain low error‑vector magnitude, and avoid adding excessive phase noise.
More drastic frequency changes require a frequency converter; the transverter converts the 802.11 signal from its original band to a new band while preserving the modulation.  However, regulatory restrictions and modern firmware (e.g., Linux regulatory databases) prevent casual users from operating Wi-Fi equipment outside of authorised bands.  In addition, the transverter must preserve the complex modulation of Wi-Fi (OFDM), maintain low error-vector magnitude, and avoid adding excessive phase noise.

## Using Superheterodyne Conversion with an IP-Based Mesh

For the drone mesh system, the concept would be to place a **superheterodyne up/down converter** between the Qualcomm QCN9074 (or a similar Wi-Fi chipset) and the antenna.  On transmission the converter would mix the 5.8 GHz signal down to a target band (e.g., 865–868 MHz) by using an LO at \(f_{\text{LO}} = f_{\text{RF}} - f_{\text{IF}}\).  On reception it would mix the incoming signal up to 5.8 GHz before it enters the radio.  The mesh network would still operate using the Wi-Fi PHY/MAC; the converter simply shifts the entire RF spectrum.  In theory this allows an **IP-based mesh** to operate in a different band where propagation or regulatory conditions are more favourable.

### Challenges

1. **Complex modulation** – Wi-Fi uses orthogonal frequency-division multiplexing (OFDM) and MIMO.  The frequency converter must maintain linearity and low phase noise to preserve constellation quality.  Any LO phase noise appears as common-phase error across OFDM subcarriers.
2. **High bandwidth** – Wi-Fi channels are up to 80 MHz wide.  Mixers and filters must accommodate this bandwidth without distortion.
3. **Image rejection and filtering** – Shifting a 5.8 GHz signal down to 865–868 MHz requires careful filtering to suppress images at \(f_{\text{LO}} \pm f_{\text{IF}}\).  For example, if an IF of 300 MHz is chosen, the LO would be approx 5.5 GHz and the image would appear at 5.2 GHz; filters would need to attenuate the unwanted component.
4. **Integration with Qualcomm chipsets** – Modern chipsets like the QCN9074 integrate the RF front end.  They are designed to operate on regulated bands with built-in RF amplifiers and filters.  Breaking out the RF signal before the front end for conversion is non-trivial and may void regulatory approval.  An external frequency converter must provide proper impedance matching and isolation so as not to disturb the chipset’s calibration.
5. **Regulatory compliance** – Operating Wi-Fi equipment in non-Part-15 bands may require an amateur radio licence or may be prohibited.  Countries like India only allow specific duty cycles and power levels in the 865–868 MHz ISM band.  Regulatory constraints may make this approach impractical compared with using 802.11 ah modules which are already designed for these bands.
6. **Cost and complexity** – Building a high-performance wideband converter requires specialised RF components and test equipment.  Off-the-shelf HaLow (802.11 ah) radio modules may provide a more economical solution.

## Feasibility Analysis

**Technical feasibility:**

* The superheterodyne architecture can certainly translate radio frequencies, and radio amateurs have built transverters for 2.4 GHz to 900 MHz.  However, these experiments often target simple spread-spectrum modems at low data rates (1 Mbps).  The wide bandwidth and strict linearity requirements of Wi-Fi 6 mean that mixers, oscillators and amplifiers must exhibit much better phase noise and gain flatness.  Achieving this across an 80 MHz channel is challenging and expensive.
* Modern Wi-Fi chipsets like Qualcomm’s QCN9074 integrate the RF front end and calibrate digital pre-distortion, making it difficult to access the raw RF without re-engineering the chip package.  An external transverter could theoretically be added after the RF output of the chipset, but this would bypass internal power amplifiers and front-end modules, degrading performance and violating certification.
* The HSMM community notes that transverters are used to move 802.11 signals to other amateur bands, but these implementations usually rely on older Atheros chipsets with open drivers and custom firmware.  Regulatory databases and secure firmware on modern chipsets restrict such modifications.

**Economic and practical feasibility:**

* Building a custom transverter adds cost: besides the Wi-Fi radio, you must source and assemble an LO synthesiser, mixer, filters, amplifiers and shielding.  Many of these components — especially high-frequency mixers and PLLs — are expensive and require careful PCB layout.  Mass-market 802.11 ah modules that already operate in the 865–868 MHz band may be cheaper and simpler.
* Regulatory compliance costs (testing, certification) can exceed the cost of the converter itself.  If the goal is to operate in the 865–868 MHz band, using COTS HaLow modules such as Doodle Labs 860 MHz transceiver may be more straightforward.

Giv
### SDR and FPGA‑Based Tunable RF Front Ends

An alternative to a discrete superheterodyne converter is to use **software‑defined radio (SDR)** chips that integrate the RF front end, frequency synthesizers and data converters into one package. These chips expose a digital I/Q interface that can be connected to an FPGA for flexible baseband processing. Examples include:

* **Analog Devices AD9361** – This highly integrated transceiver contains dual independent transmit and receive channels with 12‑bit ADC/DACs. The receiver covers **70 MHz–6 GHz** and the transmitter covers **47 MHz–6 GHz**; the tunable channel bandwidth is up to **56 MHz**. Integrated fractional‑N synthesizers generate LOs for both paths, and the device is packaged in a **10 mm × 10 mm** chip‑scale BGA【494275668734146†L14-L50】【494275668734146†L74-L79】. The AD9361 effectively combines a tunable RF front end with a flexible mixed‑signal baseband that can be interfaced directly to an FPGA【494275668734146†L39-L50】.

* **Lime Microsystems LMS7002M** – This full‑duplex transceiver provides continuous coverage from **100 kHz to 3.8 GHz** with programmable modulation bandwidth up to **120 MHz (analog) or 56 MHz (digital)**. It integrates **12‑bit ADC/DAC converters, fractional‑N PLLs**, calibration circuits and a microcontroller in an **11.5 mm × 11.5 mm** aQFN package【710745085554493†L7-L57】【710745085554493†L57-L63】. The LMS7002M includes LNAs, mixers, filters, synthesizers and requires few external components【710745085554493†L192-L205】. Its digital interface can be connected to an FPGA for custom baseband processing.

* **Qorvo RFFC2072** – This re‑configurable frequency conversion device contains a fractional‑N PLL synthesizer, VCO and high‑linearity mixer. It can generate local oscillators between **85 MHz and 2700 MHz** and up/down convert RF signals from **30 MHz to 2700 MHz**. The device is controlled via a simple 3‑wire serial interface and is available in a **5 mm × 5 mm** QFN package【284687923675379†L474-L521】. While not a complete SDR, it can serve as a tunable RF front end when paired with an FPGA.

Using an integrated SDR front end removes the need for an external LO, mixer and filters and simplifies the design. The **FPGA handles modulation/demodulation**, packet processing and mesh networking, while the SDR chip performs frequency synthesis and analog RF processing. Such devices are commercially available, have undergone regulatory certification and are already used in compact radios and UAV payloads. Therefore, they provide a feasible path to a **frequency‑agile mesh network** without the complexity of a custom superheterodyne transverter.
en these considerations, **using a superheterodyne frequency converter to repurpose a 5.8 GHz Qualcomm Wi-Fi chipset for 865–868 MHz operation is technically possible but highly complex and generally not practical**.  It is better to use purpose-built radios (e.g., 802.11 ah or LoRa) for low-frequency links, while retaining the 5.8 GHz mesh for high-throughput backhaul.

## Build Steps (Hypothetical)

If one still wants to experiment with a superheterodyne converter for educational purposes, the following high-level steps outline the process:

1. **Define the target frequency and intermediate frequency.**  For example, to translate 5.8 GHz to 865 MHz using an IF of 300 MHz, the LO would be tuned to 5.8 GHz - 0.3 GHz = 5.5 GHz.  Choose an IF that allows available filters and amplifiers.
2. **Select or design the local oscillator.**  A fractional-N PLL synthesiser (e.g., Analog Devices ADF4351/ADF5355) can generate a stable LO at 5.5 GHz.  The LO must have low phase noise over the Wi-Fi channel bandwidth.
3. **Choose an RF mixer.**  Use a high-linearity double-balanced mixer (e.g., Mini-Circuits ZX05-83+) that can handle the 5.8 GHz input and 80 MHz bandwidth.  The mixer must operate bidirectionally (up/down conversion).
4. **Design RF and IF filters.**  A band-pass filter ahead of the mixer must select the 5.8 GHz channel and attenuate the image frequency (5.2 GHz in this example).  The IF filter (centered at 300 MHz) must pass the entire Wi-Fi channel.  Use microstrip or cavity filters with steep skirts.
5. **Add low-noise and power amplifiers.**  A low-noise amplifier (LNA) at the input improves sensitivity.  After mixing and filtering, an IF amplifier boosts the signal.  On the transmit side, a power amplifier at 865 MHz raises the converted signal to the required power level.
6. **Provide control and switching.**  A microcontroller or FPGA controls the LO synthesiser and handles transmit/receive switching.  You may also need a diplexer or circulator to separate the transmit and receive paths.
7. **Integrate with the Wi-Fi chipset.**  Connect the RF output of the QCN9074 or IPQ95xx board to the input of the transverter using coax and a matching network.  Ensure the transverter presents the correct impedance and power level to the radio.
8. **Validate performance.**  Use a spectrum analyser and vector signal analyser to verify that the converted signal has acceptable error-vector magnitude (EVM) and that the receiver can demodulate the Wi-Fi waveform.  Adjust LO power, mixer drive and filter alignment accordingly.
9. **Ensure regulatory compliance.**  Before operating the system, verify that transmissions at the new frequency comply with local regulations and that the equipment is properly certified.

## Major Bill of Materials

The table below lists essential components needed for a superheterodyne frequency converter.  Specific part numbers are suggestions; equivalent components may be chosen based on availability and performance requirements.

| Component category | Example part | Role |
|---|---|---|
| **Wi-Fi baseband transceiver** | Qualcomm **QCN9074** PCIe module or IPQ95xx system | Provides the Wi-Fi 6 PHY/MAC for mesh networking; outputs 5.8 GHz RF |
| **Local oscillator / synthesiser** | Analog Devices **ADF4351/ADF5355** fractional-N PLL | Generates the tunable LO at the appropriate frequency for up/down conversion |
| **RF mixer** | Mini-Circuits **ZX05-83+** or Hittite **HMC154** | Performs the frequency mixing between the RF and LO, producing the intermediate frequency |
| **RF preselection filter** | 5.8 GHz band-pass filter (e.g., custom microstrip or cavity filter) | Removes unwanted signals and suppresses image frequencies before the mixer |
| **IF filter** | 300 MHz band-pass SAW or LC filter | Passes the desired 80 MHz Wi-Fi channel at the chosen intermediate frequency |
| **Low-noise amplifier (LNA)** | Avago **ATF-54143** or Qorvo **QPL9064** | Amplifies the weak incoming RF signal before mixing |
| **IF amplifier** | Mini-Circuits **Gali-55+** or similar | Provides gain at the intermediate frequency to overcome mixer losses |
| **Power amplifier (PA)** | Qorvo **RFPA5522** (865 MHz) or custom 868 MHz PA | Boosts the converted 865 MHz transmit signal to the required power |
| **Directional coupler / Diplexer** | 3 dB hybrid or RF switch | Separates transmit and receive paths and routes signals appropriately |
| **Control MCU/FPGA** | STM32 microcontroller or small FPGA | Controls the LO synthesiser, switching and monitors system status |
| **Support components** | RF shielding cans, SMA connectors, power regulators | Provide mechanical and electrical integration |

## Conclusion

While a superheterodyne frequency converter can theoretically shift a 5.8 GHz Wi-Fi signal to a lower band, **the practical challenges are significant**.  The design must preserve the wideband OFDM modulation, provide image rejection and low phase noise, integrate with highly integrated chipsets and comply with regulations.  Historical experiments show that converting 802.11b to other bands is possible but prone to numerous issues and rarely completed.  The high-speed multimedia radio community acknowledges that transverters exist for 802.11 but notes that regulatory and firmware constraints complicate their use.  For most real-world applications — especially in a safety-critical UAV mesh network — it is **more practical to use commercial radios designed for the desired band** (e.g., 802.11 ah modules for 865–868 MHz) and leverage the existing 5.8 GHz Wi-Fi mesh for high throughput.

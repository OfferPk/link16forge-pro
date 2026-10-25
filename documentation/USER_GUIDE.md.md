# Link16Forge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your browser.  
2. In the "Tactical Data Link Transceiver (MIDS)" panel, configure your Track Number (e.g. `01234`) and Callsign (e.g. `VIPER_01`).  
3. Select your J-Series Message Type:  
   - `J2.2`: Air PPLI (Precise Participant Location & Identification)  
   - `J3.2`: Air Track Surveillance Report  
   - `J3.5`: Land Track Surface Report  
   - `J7.0`: Track Management & Correlation  
4. Configure WGS-84 Latitude (`33.7490°`), Longitude (`-84.3880°`), Altitude (`35,000 ft`), Speed (`450 kts`), and Course (`90°`).  
5. Click **Transmit J2.2 PPLI Slot** to encode the 75-bit Initial Word (\$I\$) and Extension Word (\$E\$), calculate WGS-84 23-bit BAM values, schedule the 7.8125 ms TDMA time slot, and inspect the bitfield hex dissector.  
6. Click **Run Normative Audit** to evaluate all 10 MIL-STD-6016F / STANAG 5516 protocol compliance checks.

## 2\. Standalone Headless Python CLI

Run automated regression testing and compliance verification:

python3 link16-cli.py \--audit \--track 01234 \--callsign VIPER\_01 \--lat 33.7490 \--lon \-84.3880
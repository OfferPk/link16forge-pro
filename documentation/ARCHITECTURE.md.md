# MIL-STD-6016F & NATO STANAG 5516 Tactical Data Link (Link 16\) Technical Architecture

## 1\. Executive Summary & Protocol Overview

Link 16, standardized under MIL-STD-6016F and NATO STANAG 5516, is a military tactical data network operating in the UHF band (969–1206 MHz) that provides jam-resistant, secure, high-throughput situational awareness across allied Joint Air, Ground, and Naval combat assets.

### Key Architectural Concepts:

1. **Time Division Multiple Access (TDMA) Architecture**:  
   - Synchronized to Network Time Reference (NTR) derived from GPS or atomic rubidium standards.  
   - **Time Slot**: Fundamental atomic unit of transmission, exactly 7.8125 ms duration (128 slots/second).  
   - **Frame**: 12.0 seconds consisting of 1,536 time slots.  
   - **Epoch**: 12.8 minutes consisting of 64 frames (98,304 time slots total).  
2. **75-Bit J-Series Word Packing**:  
   - Fixed Format messages are composed of 75-bit words:  
     - **Initial Word (\$I\$)**: Identifies message label (5 bits), sublabel (3 bits), Track Number (19 bits), and primary kinematic coordinates.  
     - **Extension Word (\$E\$)**: Transmits supplemental flight data (speed, course, altitude, fuel, equipment status).  
     - **Continuation Word (\$C\$)**: Carries secondary track reports, amplifier data, or free-text lines.  
3. **WGS-84 Binary Angle Measurement (BAM)**:  
   - Latitude and Longitude mapped to 23-bit two's complement binary angle measurements, achieving 0.000043° angular resolution (\~4.7 meters precision).  
4. **Physical Layer Jam Resistance**:  
   - Fast frequency hopping across 51 discrete UHF channels.  
   - Forward Error Correction (FEC) employing Reed-Solomon \$(31, 15)\$ 5-bit symbol coding (corrects up to 8 symbol errors per 31-symbol codeword).  
   - MSK (Minimum Shift Keying) continuous phase frequency modulation.
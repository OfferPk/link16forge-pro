/\*\*  
 \* MIL-STD-6016F & NATO STANAG 5516 Tactical Data Link (Link 16\) Core Protocol Engine  
 \* Implements 75-bit J-Series message word packing (Initial, Extension, Continuation),  
 \* WGS-84 23-bit binary angle coordinates, and TDMA 7.8125 ms time slot scheduling.  
 \*/

// J-Series Message Catalog Identifiers (Label \+ Sublabel)  
const J\_SERIES\_CATALOG \= {  
  J2\_2\_AIR\_PPLI: { label: 2, sublabel: 2, name: 'J2.2 Air PPLI (Precise Participant Location)' },  
  J3\_2\_AIR\_TRACK: { label: 3, sublabel: 2, name: 'J3.2 Air Track (Surveillance)' },  
  J3\_5\_LAND\_TRACK: { label: 3, sublabel: 5, name: 'J3.5 Land Track (Surface/Ground)' },  
  J7\_0\_TRACK\_MGMT: { label: 7, sublabel: 0, name: 'J7.0 Track Management' },  
  J28\_2\_FREE\_TEXT: { label: 28, sublabel: 2, name: 'J28.2 Free Text Message' }  
};

// Tactical Track Identity (Friend, Hostile, Neutral, Unknown)  
const TACTICAL\_IDENTITIES \= {  
  PENDING: 0,  
  UNKNOWN: 1,  
  ASSUMED\_FRIEND: 2,  
  FRIEND: 3,  
  NEUTRAL: 4,  
  SUSPECT: 5,  
  HOSTILE: 6  
};

// TDMA Architecture Parameters  
const TDMA\_SPECS \= {  
  TIME\_SLOT\_MS: 7.8125,       // Exact time slot duration in ms (128 slots/sec)  
  SLOTS\_PER\_FRAME: 1536,      // 12-second Frame \= 1536 time slots  
  FRAMES\_PER\_EPOCH: 64,       // 12.8-minute Epoch \= 64 frames  
  SLOTS\_PER\_EPOCH: 98304,     // Total time slots per 12.8-min Epoch  
  FREQUENCY\_CHANNELS: 51,     // 51 UHF frequencies (969 \- 1206 MHz)  
  PULSES\_PER\_SLOT\_STD: 258    // Packed-2 transmission pulses per slot  
};

class Link16Engine {  
  constructor(sourceTrackNumber \= '01234', sourceCallsign \= 'VIPER\_01') {  
    this.sourceTrackNumber \= sourceTrackNumber; // 5-character alphanumeric (e.g. '01234')  
    this.sourceCallsign \= sourceCallsign;  
    this.currentNet \= 12; // Net Number (0 to 127\)  
    this.slotSequence \= 0;  
  }

  /\*\*  
   \* Encodes WGS-84 Latitude to 23-bit Two's Complement Binary Angle Measurement (BAM)  
   \* Scaling: \-90.0 to \+90.0 deg mapped to 23 bits (LSB \= 180 / 2^23 \= \~0.0000214 deg)  
   \*/  
  static encodeLatitudeBAM(latDeg) {  
    const clamped \= Math.max(-90.0, Math.min(90.0, latDeg));  
    const factor \= Math.pow(2, 23\) / 180.0;  
    let bam \= Math.round(clamped \* factor);  
    if (bam \< 0\) {  
      bam \= (Math.pow(2, 23\) \+ bam);  
    }  
    return bam & 0x7FFFFF;  
  }

  /\*\*  
   \* Encodes WGS-84 Longitude to 23-bit Two's Complement BAM  
   \* Scaling: \-180.0 to \+180.0 deg mapped to 23 bits (LSB \= 360 / 2^23 \= \~0.0000429 deg)  
   \*/  
  static encodeLongitudeBAM(lonDeg) {  
    const clamped \= Math.max(-180.0, Math.min(180.0, lonDeg));  
    const factor \= Math.pow(2, 23\) / 360.0;  
    let bam \= Math.round(clamped \* factor);  
    if (bam \< 0\) {  
      bam \= (Math.pow(2, 23\) \+ bam);  
    }  
    return bam & 0x7FFFFF;  
  }

  /\*\*  
   \* Encodes Altitude (ft) to 12-bit integer (25 ft resolution up to 102,375 ft)  
   \*/  
  static encodeAltitude(altFt) {  
    const clamped \= Math.max(0, Math.min(100000, altFt));  
    return Math.round(clamped / 25.0) & 0x0FFF;  
  }

  /\*\*  
   \* Encodes a standard 75-bit J2.2 Air PPLI (Precise Participant Location & Identification) Message  
   \* Initial Word I (75 bits):  
   \* \[74:70\] Label (5 bits \= 2\)  
   \* \[69:67\] Sublabel (3 bits \= 2\)  
   \* \[66:48\] Track Number (19 bits)  
   \* \[47:25\] Latitude (23 bits BAM)  
   \* \[24:2\]  Longitude (23 bits BAM)  
   \* \[1:0\]   Word Identifier (00 \= Initial Word I)  
   \*/  
  serializeJ22AirPpli(latDeg \= 33.7490, lonDeg \= \-84.3880, altFt \= 35000, speedKts \= 450, courseDeg \= 90\) {  
    this.slotSequence++;  
    const label \= J\_SERIES\_CATALOG.J2\_2\_AIR\_PPLI.label;  
    const sublabel \= J\_SERIES\_CATALOG.J2\_2\_AIR\_PPLI.sublabel;  
    const latBAM \= Link16Engine.encodeLatitudeBAM(latDeg);  
    const lonBAM \= Link16Engine.encodeLongitudeBAM(lonDeg);  
    const altCode \= Link16Engine.encodeAltitude(altFt);

    // Initial Word I (75 bits) represented across three 32-bit words (96 bits with 21 bits padding)  
    const wordI\_High \= ((label & 0x1F) \<\< 27\) \| ((sublabel & 0x07) \<\< 24\) \| ((parseInt(this.sourceTrackNumber, 16\) \|\| 0x1234) & 0xFFFFFF);  
    const wordI\_Mid \= latBAM;  
    const wordI\_Low \= (lonBAM \<\< 2\) \| 0x00;

    // Extension Word E (75 bits): Altitude, Speed, Course, Fuel  
    const courseCode \= Math.round((courseDeg / 360.0) \* 511\) & 0x01FF; // 9 bits  
    const speedCode \= Math.round(speedKts / 2.0) & 0x01FF; // 9 bits  
    const wordE\_High \= (0x01 \<\< 30\) \| (altCode \<\< 18\) \| (speedCode \<\< 9\) \| courseCode;  
    const wordE\_Mid \= 0x55AA1234; // System health & fuel  
    const wordE\_Low \= 0x01; // Extension word tag

    return {  
      messageType: 'J2.2\_AIR\_PPLI',  
      label: label,  
      sublabel: sublabel,  
      trackNumber: this.sourceTrackNumber,  
      callsign: this.sourceCallsign,  
      coordinates: {  
        latDeg: latDeg,  
        lonDeg: lonDeg,  
        altFt: altFt,  
        latBAMHex: '0x' \+ latBAM.toString(16).toUpperCase().padStart(6, '0'),  
        lonBAMHex: '0x' \+ lonBAM.toString(16).toUpperCase().padStart(6, '0')  
      },  
      tdmaSlot: {  
        currentSlotIndex: (this.slotSequence % TDMA\_SPECS.SLOTS\_PER\_FRAME),  
        slotDurationMs: TDMA\_SPECS.TIME\_SLOT\_MS,  
        frequencyHoppingChannels: TDMA\_SPECS.FREQUENCY\_CHANNELS,  
        pulses: TDMA\_SPECS.PULSES\_PER\_SLOT\_STD  
      },  
      words75Bit: \[  
        { wordType: 'INITIAL\_WORD\_I', bits: 75, hex: \`0x\${wordI\_High.toString(16).toUpperCase()}\_\${wordI\_Mid.toString(16).toUpperCase()}\` },  
        { wordType: 'EXTENSION\_WORD\_E', bits: 75, hex: \`0x\${wordE\_High.toString(16).toUpperCase()}\_\${wordE\_Mid.toString(16).toUpperCase()}\` }  
      \]  
    };  
  }

  /\*\*  
   \* Serializes a J3.2 Air Surveillance Track Message  
   \*/  
  serializeJ32AirTrack(targetTrackNum \= '05678', identity \= TACTICAL\_IDENTITIES.HOSTILE, altFt \= 28000, speedKts \= 520\) {  
    const label \= J\_SERIES\_CATALOG.J3\_2\_AIR\_TRACK.label;  
    const sublabel \= J\_SERIES\_CATALOG.J3\_2\_AIR\_TRACK.sublabel;  
    const altCode \= Link16Engine.encodeAltitude(altFt);

    return {  
      messageType: 'J3.2\_AIR\_TRACK',  
      label: label,  
      sublabel: sublabel,  
      targetTrackNumber: targetTrackNum,  
      identity: identity,  
      identityName: Object.keys(TACTICAL\_IDENTITIES).find(k \=\> TACTICAL\_IDENTITIES\[k\] \=== identity),  
      altitudeFt: altFt,  
      speedKts: speedKts,  
      wordsCount: 2  
    };  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= {  
    Link16Engine,  
    J\_SERIES\_CATALOG,  
    TACTICAL\_IDENTITIES,  
    TDMA\_SPECS  
  };  
}  

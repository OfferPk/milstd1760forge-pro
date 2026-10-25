/\*\*  
 \* MIL-STD-1760E Aircraft/Store Electrical Interconnection System (AEIS) Core Protocol Engine  
 \* Implements MIL-STD-1553B AEIS subaddress mapping (SA 11, SA 19, SA 27, SA 1),  
 \* Two's complement 16-bit checksum calculation, and discrete Release Consent safety interlocks.  
 \*/

// AEIS Standard Subaddresses (MIL-STD-1760E Table A-I)  
const AEIS\_SUBADDRESSES \= {  
  HEALTH\_CHECKSUM: 1,      // Subaddress 1: General Status & Checksum  
  TRANSFER\_HEADER: 11,     // Subaddress 11: Mission Store Control & Transfer Header  
  CRITICAL\_CONTROL: 19,    // Subaddress 19: Critical Control & Fuzing Arming  
  BASE\_INVENTORY: 27       // Subaddress 27: Store Description & Inventory  
};

// Store Arming & Release States  
const STORE\_STATES \= {  
  OFF: 'OFF',  
  INITIALIZED\_SAFE: 'INITIALIZED\_SAFE',  
  STANDBY: 'STANDBY',  
  ARMED\_CONSENT\_PENDING: 'ARMED\_CONSENT\_PENDING',  
  READY\_FOR\_RELEASE: 'READY\_FOR\_RELEASE',  
  RELEASED: 'RELEASED'  
};

// Standard Store Catalog (Country Code \+ Store Class)  
const STORE\_CATALOG \= {  
  JDAM\_GBU38: { countryCode: 840, storeId: 0x0110, name: 'GBU-38 JDAM (500 lb)' },  
  SDB2\_GBU53: { countryCode: 840, storeId: 0x0225, name: 'GBU-53/B SDB II StormBreaker' },  
  AMRAAM\_AIM120D: { countryCode: 840, storeId: 0x0340, name: 'AIM-120D AMRAAM Air-to-Air' },  
  TARGETING\_POD: { countryCode: 840, storeId: 0x0501, name: 'Advanced Targeting Pod (ATP)' }  
};

class MIL1760Engine {  
  constructor(stationId \= 3, remoteTerminalAddr \= 5\) {  
    this.stationId \= stationId;                 // Aircraft Station Interface (ASI) Pylon \#  
    this.remoteTerminalAddr \= remoteTerminalAddr; // 1553 RT Address (0-30)  
    this.state \= STORE\_STATES.INITIALIZED\_SAFE;  
    this.discretes \= {  
      releaseConsent: false,  // Hardware discrete interlock line  
      interlockReturn: true,  // Hardware loopback continuity detected  
      power28vPrimary: true,  
      power28vSecondary: false,  
      power115vAc: false  
    };  
    this.activeStore \= STORE\_CATALOG.JDAM\_GBU38;  
    this.messageSequence \= 0;  
  }

  /\*\*  
   \* MIL-STD-1760E Checksum Algorithm:  
   \* The sum of all 16-bit data words in the message block plus the checksum word  
   \* shall equal zero (modulo 2^16 \= 0x10000).  
   \* Checksum Word \= (- sum) & 0xFFFF  
   \*/  
  static calculateTwoComplementChecksum(dataWords16) {  
    let sum \= 0;  
    for (let w of dataWords16) {  
      sum \= (sum \+ (w & 0xFFFF)) & 0xFFFF;  
    }  
    const checksum \= (-sum) & 0xFFFF;  
    return checksum;  
  }

  /\*\*  
   \* Verifies that the sum of data words including the checksum word equals 0  
   \*/  
  static verifyChecksum(dataWordsWithChecksum) {  
    let sum \= 0;  
    for (let w of dataWordsWithChecksum) {  
      sum \= (sum \+ (w & 0xFFFF)) & 0xFFFF;  
    }  
    return sum \=== 0;  
  }

  /\*\*  
   \* Encodes Subaddress 27 Base Inventory (Store Description)  
   \*/  
  encodeBaseInventory() {  
    this.messageSequence++;  
    const words \= \[  
      0x001B, // Word 1: Subaddress 27 Tag  
      this.activeStore.countryCode, // Word 2: Country Code (840 \= US)  
      this.activeStore.storeId,     // Word 3: Specific Store ID  
      0x0001, // Word 4: Operational Mode (Armed Capable)  
      0x0200, // Word 5: Mass / Weight Class (500 lb)  
      0x0000, 0x0000, 0x0000 // Reserved  
    \];

    const checksum \= MIL1760Engine.calculateTwoComplementChecksum(words);  
    words.push(checksum);

    return {  
      subaddress: AEIS\_SUBADDRESSES.BASE\_INVENTORY,  
      storeName: this.activeStore.name,  
      wordsCount: words.length,  
      words: words,  
      checksumHex: '0x' \+ checksum.toString(16).toUpperCase().padStart(4, '0'),  
      isChecksumValid: MIL1760Engine.verifyChecksum(words)  
    };  
  }

  /\*\*  
   \* Encodes Subaddress 11 Mission Store Transfer Header & Target Handoff  
   \*/  
  encodeTransferHeader(targetLat \= 33.7490, targetLon \= \-84.3880, targetAltFt \= 1050\) {  
    this.messageSequence++;  
    // Convert target coordinates to 16-bit military grid words  
    const latWord \= Math.round(((targetLat \+ 90.0) / 180.0) \* 65535\) & 0xFFFF;  
    const lonWord \= Math.round(((targetLon \+ 180.0) / 360.0) \* 65535\) & 0xFFFF;  
    const altWord \= Math.min(65535, Math.max(0, targetAltFt)) & 0xFFFF;

    const words \= \[  
      0x000B, // Word 1: SA 11 Identification  
      this.messageSequence & 0xFFFF, // Word 2: Transfer Sequence Count  
      0x0041, // Word 3: Guidance Mode (GPS/INS Aided)  
      latWord, // Word 4: Target Latitude  
      lonWord, // Word 5: Target Longitude  
      altWord, // Word 6: Target Altitude  
      0x00FF, // Word 7: Impact Angle & Velocity  
      0x0001  // Word 8: Terminal Trajectory Flag  
    \];

    const checksum \= MIL1760Engine.calculateTwoComplementChecksum(words);  
    words.push(checksum);

    return {  
      subaddress: AEIS\_SUBADDRESSES.TRANSFER\_HEADER,  
      targetCoords: { lat: targetLat, lon: targetLon, altFt: targetAltFt },  
      wordsCount: words.length,  
      words: words,  
      checksumHex: '0x' \+ checksum.toString(16).toUpperCase().padStart(4, '0'),  
      isChecksumValid: MIL1760Engine.verifyChecksum(words)  
    };  
  }

  /\*\*  
   \* Encodes Subaddress 19 Critical Control & Safety Interlocks  
   \*/  
  encodeCriticalControl(armCommand \= false) {  
    this.messageSequence++;  
      
    // Safety verification: Can only arm if hardware Release Consent discrete is asserted  
    const isConsentAsserted \= this.discretes.releaseConsent && this.discretes.interlockReturn;  
    const fuzingState \= (armCommand && isConsentAsserted) ? 0xAA55 : 0x0000;

    if (fuzingState \=== 0xAA55) {  
      this.state \= STORE\_STATES.READY\_FOR\_RELEASE;  
    } else if (armCommand && \!isConsentAsserted) {  
      this.state \= STORE\_STATES.ARMED\_CONSENT\_PENDING;  
    } else {  
      this.state \= STORE\_STATES.INITIALIZED\_SAFE;  
    }

    const words \= \[  
      0x0013, // Word 1: SA 19 Header  
      fuzingState, // Word 2: Fuzing Arming Pattern (0xAA55 \= ARMED)  
      isConsentAsserted ? 0x0001 : 0x0000, // Word 3: Release Consent Flag  
      0x0002, // Word 4: Safe Separation Timer (1.2s delay)  
      0x0000  // Word 5: Reserved  
    \];

    const checksum \= MIL1760Engine.calculateTwoComplementChecksum(words);  
    words.push(checksum);

    return {  
      subaddress: AEIS\_SUBADDRESSES.CRITICAL\_CONTROL,  
      fuzingArmed: fuzingState \=== 0xAA55,  
      consentAsserted: isConsentAsserted,  
      storeState: this.state,  
      wordsCount: words.length,  
      words: words,  
      checksumHex: '0x' \+ checksum.toString(16).toUpperCase().padStart(4, '0'),  
      isChecksumValid: MIL1760Engine.verifyChecksum(words)  
    };  
  }

  /\*\*  
   \* Toggles Hardware Discrete Release Consent  
   \*/  
  setReleaseConsent(asserted) {  
    this.discretes.releaseConsent \= asserted;  
    return this.discretes;  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= {  
    MIL1760Engine,  
    AEIS\_SUBADDRESSES,  
    STORE\_STATES,  
    STORE\_CATALOG  
  };  
}  

# MILSTD1760Forge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your web browser.  
2. In the "Weapons Station Management (SMS)" panel, set the Aircraft Station Pylon Number (e.g. `3`) and the Remote Terminal (RT) Address (e.g. `5`).  
3. Select the mission store weapon (e.g. `GBU-38 JDAM (GPS/INS 500 lb)` or `GBU-53/B SDB II StormBreaker`).  
4. Select the AEIS Subaddress operation:  
   - `SA 27`: Request Base Inventory to verify country code and store identity.  
   - `SA 11`: Transmit Transfer Header with GPS target coordinate handoffs.  
   - `SA 19`: Command Critical Control / Fuzing Arm.  
5. In the Hardware Safety Discretes panel, verify the Interlock Return Loop and toggle Release Consent to satisfy the dual-condition interlock.  
6. Click **Transmit Subaddress** to calculate the Two's Complement Checksum and inspect the 16-bit word hex dump.  
7. Click **Arm Weapon (SA 19\)** to test the state transition from `INITIALIZED_SAFE` to `READY_FOR_RELEASE`.  
8. Click **Run Normative Audit** to evaluate all 10 MIL-STD-1760E protocol compliance checks.

## 2\. Standalone Headless Python CLI

Run automated regression testing and compliance verification:

python3 mil1760-cli.py \--audit \--station 3 \--store JDAM\_GBU38
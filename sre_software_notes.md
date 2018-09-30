# Spartan Racing Electric Software Notes  
  
*Note: The code written for the VCU does not include RTOS and is non-blocking code, thus time will not affect outputs*  
  
## BMS   
### isoSPI
* signals for CSB and SCK  
&nbsp;&nbsp;&nbsp;• bidirectional communication  
&nbsp;&nbsp;&nbsp;• Device 0 has logic states  
&nbsp;&nbsp;&nbsp;• iso barrier  
&nbsp;&nbsp;&nbsp;• Device 1 has logic  
&nbsp;&nbsp;&nbsp;• In between is decoding and encoding  
* drives differential signals with source & sink currents  
&nbsp;&nbsp;&nbsp;• no need for transformer tap and reducing EMI  
* window comparators in receiver detect differential signal  
* resistor divider sets  
&nbsp;&nbsp;&nbsp;• drive currents  
&nbsp;&nbsp;&nbsp;• comparator thresholds  
* LT3990 handles drive pin (LT6804 SchDoc)  
* LTC6811-1 check these pins  
&nbsp;&nbsp;&nbsp;• SCK -> Monitored by Wake Up  
&nbsp;&nbsp;&nbsp;• CSB -> Monitored by Wake Up  
&nbsp;&nbsp;&nbsp;• ISOMD -> SPI or isoSPI?  
&nbsp;&nbsp;&nbsp;• WDT -> enabled by Vreg  
&nbsp;&nbsp;&nbsp;• DRIVE -> LT3990:ENNVLO, it outputs high to supply the regulator  
  
### 6811 State  
* SLEEP - ADC down, WDTravel, isoSPI in IDLE, supply min, drive = 0V  
&nbsp;&nbsp;&nbsp;• wakeup signal  
* STANDBY - ADC off, WDT running, drive pin 5V  
&nbsp;&nbsp;&nbsp;• ADC command -> MEASURE  
&nbsp;&nbsp;&nbsp;• REFON set to 1, IC power up -> REFUP  
&nbsp;&nbsp;&nbsp;• REFON set to 0, IC power down -> SLEEP  
* MEASURE - Ref and ADC power up  
&nbsp;&nbsp;&nbsp;• Done -> REFON set to 0 or 1  
* REFUP - WRCFGA command  
&nbsp;&nbsp;&nbsp;• ADC off  
&nbsp;&nbsp;&nbsp;• Ref power up  
&nbsp;&nbsp;&nbsp;• ADC faster than Standby  
&nbsp;&nbsp;&nbsp;• ADC command -> Measure  
&nbsp;&nbsp;&nbsp;• WRCFGA = 0 -> Standby  
&nbsp;&nbsp;&nbsp;• WDT expires -> Standby  
  
### Write Configuration  
Note: Check page 56 of the 6811 datasheets  
* RDCFGA   
&nbsp;&nbsp;&nbsp;00000000010  
* CC[10:0] = WRCFGA  
&nbsp;&nbsp;&nbsp;00000000001  
* Read Cell Voltage A   
&nbsp;&nbsp;&nbsp;00000000100  
  
### Waking up the Serial Interface  
* pins 41 & 42  
* ISOMD = V-, Port A -> SPI (on page 6 in 6811 datasheets)  
* ISOMD = Vreg, Port A -> isoSPI  
* Daisy chain  
&nbsp;&nbsp;&nbsp;• N&times;t(wake) or N&times;t(ready) -> propagate wake up signal  
&nbsp;&nbsp;&nbsp;• t(ready)/t(wake) < isoSPI pulses < t(idle) i.e. CSB of 6820 or ISOMD=0  
  
### Power  
* V+ >= Top cell V - 0.3V (on page 21 of 6811 datasheets)  
&nbsp;&nbsp;&nbsp;• HV elements  
* Vreg = 5V  
&nbsp;&nbsp;&nbsp;• remaining core and isoSPI circuit  
&nbsp;&nbsp;&nbsp;• powered by extern transistor  
&nbsp;&nbsp;&nbsp;• driven by regulated DRIVE pin  
&nbsp;&nbsp;&nbsp;• external supply  
  
### ADC Modes  
* Measures inputs then calibrates channel  
&nbsp;&nbsp;&nbsp;• based on -3dB bandwidth of ADC measure  
* CFGRO[0] - conversion command  
&nbsp;&nbsp;&nbsp;• ADCOPT bit (on page 58 of 6811 datasheets)  
&nbsp;&nbsp;&nbsp;• Mode select bits, MD[1:0]   
&nbsp;&nbsp;&nbsp;• 8 modes of ADC correspond to oversampling ratios (OSR) (on page 59 of the 6811 datasheets)  
  
### Timing  
_Note: On page 8 of the 6811 datasheets_  
* t(ready)  = 10 us  
* t(idle)   = 4.3, 5.5, 6.7 ms  
* t(clk)    = 1 us  
* t(dwell)  = 240 ns, time at Vwake before detection  
* Vwake     = 200 mV  
  
### isoSPI State  
* IDLE  
&nbsp;&nbsp;&nbsp;• Nothing happens, waits for wake up  
* READY  
&nbsp;&nbsp;&nbsp;• if 6811 on STANDBY, then DRIVE & Vreg biased up  
&nbsp;&nbsp;&nbsp;• if 6811 on SLEEP, slower transition  
&nbsp;&nbsp;&nbsp;• Port B for 6811-1 enabled  
&nbsp;&nbsp;&nbsp;• No activity on Port A > t(idle) = 5.5 ms ~ 182 Hz  
&nbsp;&nbsp;&nbsp;• if data transmits or receives -> ACTIVE  
* ACTIVE  
&nbsp;&nbsp;&nbsp;• uses one of isoSPI ports  
&nbsp;&nbsp;&nbsp;• consumes max power  
  
## ADC  
* RTD LED   
&nbsp;&nbsp;&nbsp;• pin 107  
&nbsp;&nbsp;&nbsp;• pin 20 (circlular dash)  
&nbsp;&nbsp;&nbsp;• IO_ADC_CUR_03  
* Eco LED  
&nbsp;&nbsp;&nbsp;• pin 108  
&nbsp;&nbsp;&nbsp;• pin 22 (dash)  
&nbsp;&nbsp;&nbsp;• IO_ADC_CUR_01  
  
## HVIL  
* HVIL declared as __extern Sensor Sensor_HVILTerm__ in *sensors.c*    
* Term Sense -> pin 253, bool SensorValue    
  
## CAN  
`canManager.c`    
* 0x509    
&nbsp;&nbsp;&nbsp;• `byte[0,1] -> HVIL.sensorValue   // MCM RTD`  
* 0x506   
&nbsp;&nbsp;&nbsp;• `byte[6,7] -> Safety_getNotices  // TermSenseLost`  
&nbsp;&nbsp;&nbsp;• Returns 0x0566 in PCAN View  
* channel <- CAN0_HIPRI -> MCM, BMS, PCAN  
&nbsp;&nbsp;&nbsp;• write handle  
&nbsp;&nbsp;&nbsp;• ioErr write  
&nbsp;&nbsp;&nbsp;• read limit  
&nbsp;&nbsp;&nbsp;• ioErr read  
&nbsp;&nbsp;&nbsp;• read handle  
  
## Motor Controller  
_Pseudocode for Relay Control_  
```c++  
void relayControl  
    if Term Sense is false & Override is false  
        if previous state is true  
            go to time that HVIL was lost  
        if relay true AKA MCM is on and HVIL is lost  
            turn MCM off when 0 torque commanded or wait 2 seconds  
            pin 144 is the MC_relay  
        reset inverter control, MCM off  
        reset inverter status  
        reset slockout status   
        previous state set false  
    if Term Sense is high or Override is high  
        if previous state is LOW  
            send serial message  
            set the MCM or ONLY if OFF  
        Set previous state -> HIGH  
        Turn on RMC  
```  
  
_Pseudocode for Inverter Control_  
```  
Check if RTD button is pressed   
State 0: MCM is off -> stay until lockout enabled  
State 1: MCM is on  -> lockout=en, inverter=disabled until lockout disabled  
State 2: MCM is on  -> lockout=dis, inverter=disabled until RTD pressed  
State 3: inverter=dis, rtd=pressed, wait for inverter=en  
State 4: inverter=dis, rtd=started  
State 5: inverter=en, rtd=!started  
Set RTD LED based on logic level of RTD button  
```  
  
## Safety Checker  
_Pseudocode_  
```c++  
ubyte2 notices  
    HVIL Term Lost  -> 1  
    Over 75kW_BMS   -> 0x10  
    Over 75kW_MCM   -> 0x20  
```  
_Warnings_  
- LVS 10% SoC -> 9200 = empty   
- LVS -> 13100 recharge percentage   
- LVS is __ V good standing  
- Safety bypass -> debugging  
- HVIL Override for MCM  
_Overrides_  
0x5FF  
* HVIL Override  
0xC4  
* Safety bypass  
0x01  
* Torque  
  
  
## Start-up Sequence  
### Calculations  
1. Read inputs from buttons, sensors, 12V battery  
2. Read CAN -> CAN obj, channel, MCM obj, BMS obj, Safety   
&nbsp;&nbsp;&nbsp;• LV is 0x2E32 -> 11826  
3. Run calibration if commanded  
```c++  
if Eco button pressed  
    begin timer after Eco button released  
    if 3 seconds pass  
        tps calib  
        bps calib  
        set Eco light  
    else  
        if 10 ms < Eco timestamp < 1 s  
            send message "Eco request"  
        reset Eco timestamp  
```  
4. Update speed of 4 wheels  
&nbsp;&nbsp;&nbsp;• RR sensor missing so comment out code in `sensors.c`  
&nbsp;&nbsp;&nbsp;• Does this hold up the MCM or HVIL?  
5. Calculate cooling system & enact cooling  
&nbsp;&nbsp;&nbsp;• Does this hold up the MCM or HVIL?  
6. Update Safety faults, notices, warnings -> VCU Error Faults  
* tps, bps not calibrated  
* tps, bps power not set or init  
* pedals failed to init or get signal  
* pedals out of range  
&nbsp;&nbsp;&nbsp;• (spec Min or spec Max)  
* throttle percentage out of sync  
&nbsp;&nbsp;&nbsp;• percent = voltage(tps) - CalibMin(tps) / CalibMax(tps) - CalibMin(tps)  
&nbsp;&nbsp;&nbsp;• total percent = percent(tps0) + percent(tps1) / 2  
* brakes activated and throttle/tps > 25% implausability  
  
### Adjustments  
7. Torque adjustments
_Pseudocode_  
```  
if faults  
    No Torque DNm  
else   
    OK Torque DNm  
```  
  
### Outputs  
8. Dash error if faults from SafetyChecker  
- Pedal Dance  
9. MCM Relay Control -> HVIL Term Sense  
10. MCM Inverter Control -> TPS, BPS, RTDS  
11. CAN Debug Messages    
```  
0x500 -> TPS0  
0x501 -> TPS1  
0x502 -> BPS0  
0x503 -> WSS FL & FR, RL & RR + 0.5  
0x504 -> WSS FL & FR sensor value  
0x505 -> WSS RL & RR sensor value  
0x506 -> Safety Check for 4 bytes Faults, 2B Warnings, 2B Notices  
0x507 -> LV Battery Sensor  
0x508 -> Regen  
0x509 -> HVIL Term Sense, HVIL override status  
0x50A -> LV Testing (All 0's)  
0xC0  -> Get torque, direction inverter, torque limit  
0x520 -> Torque Encoder, TCS Knob, Eco, RTD  
```  
  
## Design Judging  
These are my personal notes from listening to input from Design Judges at the 2018 Formula SAE Lincoln competition  
  
### Suspension  
* Moment of intertia good because of transience  
&nbsp;&nbsp;&nbsp;• Prove it e.g. twist test  
* Soft rear lowers CG, load transfer  
* They want numbers  
* Motor/Inverter use RMS for load demand  
&nbsp;&nbsp;&nbsp;• efficiency map or heat map  
&nbsp;&nbsp;&nbsp;• cooling  
* Knowledge Transfer  
&nbsp;&nbsp;&nbsp;• Recommend having thick binders  
&nbsp;&nbsp;&nbsp;• Recommend showing thought process  
* Simulations  
&nbsp;&nbsp;&nbsp;• Can be written in R, Python, Matlab  
&nbsp;&nbsp;&nbsp;• Show suspension verification where applicable  
&nbsp;&nbsp;&nbsp;• Gather tire data  
&nbsp;&nbsp;&nbsp;• Display gear ratio  
  
### Powertrain  
* Manipulate pedal maps  
&nbsp;&nbsp;&nbsp;• Consistent torquer and varying voltages  
&nbsp;&nbsp;&nbsp;• Derate max torque (judge emphasized on this)  
* Validate data (sweet spot)  
&nbsp;&nbsp;&nbsp;• Display temperature, axis, and torque estimates  
&nbsp;&nbsp;&nbsp;• Reduce battery packs (how???)  
  
### Systems  
* Present exploded view of accumulators  
&nbsp;&nbsp;&nbsp;• No one can see inside the modules  
&nbsp;&nbsp;&nbsp;• What makes them swappable?  
&nbsp;&nbsp;&nbsp;• Make poster board for things viewers cannot see externally  
* Show accumulator split view or 3D print   
* Cost and weight?  
* Accumulator internals  
&nbsp;&nbsp;&nbsp;• Issues with loose wiring   
&nbsp;&nbsp;&nbsp;• Issues with simular coloring  
&nbsp;&nbsp;&nbsp;• Issues with simular connectors  
* How will we go about customers troubleshooting?  
&nbsp;&nbsp;&nbsp;• Legacy  
&nbsp;&nbsp;&nbsp;• Wiring diagram  
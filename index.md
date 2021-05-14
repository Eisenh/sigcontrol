# sigcontrol (sigcontrol)

Control SPD3303X power supply
note: for windows, winUSB driver must first be installed (e.g.,using Zadig)
https://github.com/pbatard/libwdi/wiki/Zadig



## Install the dependencies
```bash
yarn
```

### Start the app in development mode (hot-code reloading, error reporting, etc.)
```bash
quasar dev
```

### Lint the files
```bash
yarn run lint
```

### Build the app for production
```bash
quasar build
```

### Customize the configuration
See [Configuring quasar.conf.js](https://quasar.dev/quasar-cli/quasar-conf-js).

  ## SCPI commands for SPD3303X  
INST CH1  
INST?  
MEAS:CURR?  
MEAS:CURR? CH1  
*READALL?  
CURR?  
CH2:CURR?  
CH2:CURR 0.5  
CH2:VOLT 0.5  
TIME:SET? CH1,2    - response 20.000,0.100,10  
TIME:SET? CH1,1      - 2.000,0.100,20  - V,C TIME  
TIME CH1,ON  
OUTP: CH1,OFF  
OUTP:WAVE CH1,ON  
OUTP:WAVE CH1,OFF  
OUTP:TRACK 1     - PUT IN SERIAL MODE  
OUTP:TRACK 0  
*SAV 1  
*RCL 1  
SYST:STAT?  - 0x4  - bits rep CV or CC, etc.  
SYST:VERS?  
  


1.  “*LOCK” – Locks out the function of the front panel buttons.

2. “*UNLOCK” – Unlocks the function of the front panel buttons.

3. “*LOCK?” – This is a query that asks the instrument the current state of the lock feature.

 --Short Cmd-----------Long Cmd------------Set-Get-----Extended v1---------Extended v2--------  
**READID                                  X   
*CALCLS                                   X   
*CALRCL                                   X   
*CALST                                    X   
*CPU                                          X  
*DEL                                      X   
*IDN                                          X  
*LOCK                                     X   X  
*RCL                                      X   
*READALL                                      X  
*SAV                                      X   
*UNLOCK                                   X   
*UPGRADE                                  X   
CAL:CURR            CAL:CURRENT           X         CALIBRATION:CURR    CALIBRATION:CURRENT  
CAL:VOLT            CAL:VOLTAGE           X         CALIBRATION:VOLT    CALIBRATION:VOLTAGE  
CH1:CURR            CH1:CURRENT           X   X  
CH1:VOLT            CH1:VOLTAGE           X   X  
CH2:CURR            CH2:CURRENT           X   X  
CH2:VOLT            CH2:VOLTAGE           X   X  
CLREEPROM                                 X   
CURR                CURRENT                   X  
DHCP                                      X   X  
FACTORY                                   X   X  
GATE                GATEADDR              X   X  
IDN-SGLT-PRI                                  X  
INST                INSTRUMENT            X   X  
IP                  IPADDR                X   X  
IPST                                      X   X  
MAC                                       X   X  
MASK                MASKADDR              X   X  
MEAS:CURR           MEAS:CURRENT              X     MEASURE:CURR        MEASURE:CURRENT  
MEAS:POWE           MEAS:POWER                X     MEASURE:POWE        MEASURE:POWER   
MEAS:VOLT           MEAS:VOLTAGE              X     MEASURE:VOLT        MEASURE:VOLTAGE  
OUTP                OUTPUT                X   
OUTP:TRACK          OUTPUT:TRACK          X   
OUTP:WAVE           OUTPUT:WAVE           X   X  
SCDP                                      X   
SRLN                                      X   X      
SYST:ERR            SYST:ERROR                X     SYSTEM:ERR          SYSTEM:ERROR  
SYST:STAT           SYST:STATUS               X     SYSTEM:STAT         SYSTEM:STATUS  
SYST:VERS           SYST:VERSION              X     SYSTEM:VERS         SYSTEM:VERSION  
TIME                TIMER                 X   
TIME:SET            TIMER:SET             X   X  
VIRKEY                                    X   
VOLT                VOLTAGE                   X  
                                             

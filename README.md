After installing the firmware, you need to reset the system. Here is the reset method.

config>System Config>Advanced>Factory Reset

<img width="355" height="241" alt="ScreenShot_2026-06-26_103803_777" src="https://github.com/user-attachments/assets/cd859393-abe0-4028-96b0-43dfcc9bd47b" />


# Cardputer-ADV-CC1101-NRF24L01-LoRa

![9082439408](https://github.com/user-attachments/assets/f7679754-3829-4d2e-b69f-2b8fac79e70f)

Firmware Download Procedure

1、Press and hold the G0 button on the Cardputer ADV, then press the RST button to enter Download Mode.

2、Open flash_download_tool_3.9.2.exe, and select the type/mode for downloading according to the picture.

<img width="217" height="206" alt="image" src="https://github.com/user-attachments/assets/99abad9a-d58b-4c9b-b709-3f9e70ae47af" />

3、Select the COM port, configure the settings as shown in the picture, then click START to begin downloading. Wait until the download is complete, then press the RST button to run normally.

<img width="757" height="675" alt="image" src="https://github.com/user-attachments/assets/cd624401-6ef6-4268-b44b-055dae3fa166" />


Bruce Firmware Adaptation Guide for M5Stack Cardputer with External CC1101 / NRF24L01 / LoRa Modules

This document summarizes the complete modification process for adding support for the following modules to Bruce firmware on M5Stack Cardputer:

.CC1101

.NRF24L01

.LoRa

.Keyboard I2C pin multiplexing

.SPI device conflict avoidance

The core idea of this solution is:


1.Modify boards/m5stack-cardputer/m5stack-cardputer.ini to add external module pin definitions

2.Modify boards/m5stack-cardputer/interface.cpp to implement keyboard I2C and external pin multiplexing

3.Modify src/modules/NRF24/nrf_common.cpp to release the CC1101 chip select before initializing NRF24, preventing SPI bus conflicts


1. Files to Modify
   
The following files need to be edited:

    boards/m5stack-cardputer/m5stack-cardputer.ini
    
    boards/m5stack-cardputer/interface.cpp
    
    src/modules/NRF24/nrf_common.cpp
    
3. Notes Before Modification
   
4. In this solution, some Cardputer pins are reused for multiple functions, so pay special attention to the following points.
   
------------------------------------------------------------------------

2.1 SPI Bus Sharing

The following modules share the same SPI bus:

.CC1101

.NRF24L01

.LoRa

When sharing the SPI bus, you must ensure that:

.Each module has its own dedicated CS/SS pin

.The CS pin of every inactive module stays HIGH


Otherwise, you may run into:
.Initialization failures
.Communication conflicts
.Mutual interference between modules

------------------------------------------------------------------------
2.2 Keyboard I2C Pin Multiplexing

The Cardputer keyboard controller TCA8418 uses:

.SDA = GPIO8

.SCL = GPIO9


In this setup, the NRF24 also uses these pins:

.CE = GPIO8

.SS = GPIO9

Because of this, the firmware must implement the following logic:

.When the keyboard needs to be accessed, switch GPIO8 / GPIO9 back to I2C mode

.After reading the keyboard event, switch GPIO8 / GPIO9 back to normal GPIO output mode

Otherwise, the keyboard and NRF24 will directly conflict with each other.


------------------------------------------------------------------------
🛠️ 3. Modify m5stack-cardputer.ini

File path:boards/m5stack-cardputer/m5stack-cardputer.ini

Under build_flags =, add or confirm the following configuration.


------------------------------------------------------------------------

3.1 Add CC1101 Configuration

-DUSE_CC1101_VIA_SPI

-DCC1101_GDO0_PIN=13

-DCC1101_SS_PIN=15

-DCC1101_MOSI_PIN=SPI_MOSI_PIN

-DCC1101_SCK_PIN=SPI_SCK_PIN

-DCC1101_MISO_PIN=SPI_MISO_PIN

;-DCC1101_GDO2_PIN=-1

Description:


GDO0 uses GPIO13

SS uses GPIO15

SPI data lines reuse the default SPI bus

3.2 Add NRF24L01 Configuration


-DUSE_NRF24_VIA_SPI

-DNRF24_CE_PIN=8

-DNRF24_SS_PIN=9

-DNRF24_MOSI_PIN=SPI_MOSI_PIN

-DNRF24_SCK_PIN=SPI_SCK_PIN

-DNRF24_MISO_PIN=SPI_MISO_PIN

Description:

CE = GPIO8

SS = GPIO9

These pins are shared with the keyboard I2C bus, so interface.cpp must also be modified

3.3 Add LoRa Configuration


-DLORA_SCK=SPI_SCK_PIN

-DLORA_MISO=SPI_MISO_PIN

-DLORA_MOSI=SPI_MOSI_PIN

-DLORA_CS=5

-DLORA_RST=3

-DLORA_DIO0=4

Description:


LoRa also reuses the default SPI bus

CS = GPIO5

RST = GPIO3

DIO0 = GPIO4

3.4 Example of the Final Relevant Configuration

Below is the full relevant configuration block after modification:


------------------------------------------------------------------------
[env:m5stack-cardputer]
board = m5stack-cardputer
board_build.partitions = custom_8Mb.csv
build_src_filter =${env.build_src_filter} +<../boards/m5stack-cardputer>
build_flags =
    ${env.build_flags}
    -Iboards/m5stack-cardputer
    -DCORE_DEBUG_LEVEL=0

    -D ANALOG_BAT_PIN=10

    -DTCA8418_INT_PIN=11
    -DTCA8418_I2C_ADDR=0x34
    -DTCA8418_SDA_PIN=8
    -DTCA8418_SCL_PIN=9

    ;Features Enabled
    ;-DLITE_VERSION=1
    -D ES8311_CODEC=1
    -D ES8311_ADDR=0x18

    ;Microphone
    -DMIC_SPM1423=1
    -DPIN_CLK=43
    -DI2S_SCLK_PIN=43
    -DI2S_DATA_PIN=46
    -DPIN_DATA=46

    ;FM Radio
    -DFM_SI4713=1
    -DFM_RSTPIN=40

    ;RGB LED
    -DHAS_RGB_LED=1
    -DRGB_LED=21

    ;Speaker
    -DHAS_NS4168_SPKR=1
    -DBCLK=41
    -DWCLK=43
    -DDOUT=42
    -DMCLK=43

    ;USB HID
    -DUSB_as_HID=1

    ;Buttons configuration
    -DHAS_BTN=0
    -DBTN_ALIAS='"Ok"'
    -DBTN_PIN=0
    -DBTN_ACT=LOW

    ;Font sizes
    -DFP=1
    -DFM=2
    -DFG=3

    ;Infrared
    -DIR_TX_PINS='{ {"Default", TXLED}, {"M5 IR Mod", GROVE_SDA}, {"Grove W", GROVE_SCL}, {"Grove Y", GROVE_SDA}, {"ADV 3", 3},{"ADV 4", 4},{"ADV 5", 5},{"ADV 6", 6},{"ADV 13", 13},{"ADV 15", 15} }'
    -DIR_RX_PINS='{ {"M5 IR Mod", GROVE_SCL}, {"Grove W", GROVE_SCL}, {"Grove Y", GROVE_SDA},{"ADV 3", 3},{"ADV 4", 4},{"ADV 5", 5},{"ADV 6", 6},{"ADV 13", 13},{"ADV 15", 15}}'
    -DTXLED=44
    -DLED_ON=HIGH
    -DLED_OFF=LOW

    ;RF one-pin modules
    -DRF_TX_PINS='{ {"M5 RF433T", GROVE_SDA}, {"Grove W", GROVE_SCL}, {"Grove Y", GROVE_SDA},{"ADV 3", 3},{"ADV 4", 4},{"ADV 5", 5},{"ADV 6", 6},{"ADV 13", 13},{"ADV 15", 15}}'
    -DRF_RX_PINS='{ {"M5 RF433R", GROVE_SCL}, {"Grove W", GROVE_SCL}, {"Grove Y", GROVE_SDA},{"ADV 3", 3},{"ADV 4", 4},{"ADV 5", 5},{"ADV 6", 6},{"ADV 13", 13},{"ADV 15", 15}}'

    ; CC1101
    -DUSE_CC1101_VIA_SPI
    -DCC1101_GDO0_PIN=13
    -DCC1101_SS_PIN=15
    -DCC1101_MOSI_PIN=SPI_MOSI_PIN
    -DCC1101_SCK_PIN=SPI_SCK_PIN
    -DCC1101_MISO_PIN=SPI_MISO_PIN
    ;-DCC1101_GDO2_PIN=-1

    ; NRF24L01
    -DUSE_NRF24_VIA_SPI
    -DNRF24_CE_PIN=8
    -DNRF24_SS_PIN=9
    -DNRF24_MOSI_PIN=SPI_MOSI_PIN
    -DNRF24_SCK_PIN=SPI_SCK_PIN
    -DNRF24_MISO_PIN=SPI_MISO_PIN

    ; W5500
    -DUSE_W5500_VIA_SPI
    -DW5500_SS_PIN=SPI_SS_PIN
    -DW5500_MOSI_PIN=SPI_MOSI_PIN
    -DW5500_SCK_PIN=SPI_SCK_PIN
    -DW5500_MISO_PIN=SPI_MISO_PIN
    -DW5500_INT_PIN=GROVE_SDA

    ; LoRa
    -DLORA_SCK=SPI_SCK_PIN
    -DLORA_MISO=SPI_MISO_PIN
    -DLORA_MOSI=SPI_MOSI_PIN
    -DLORA_CS=5
    -DLORA_RST=3
    -DLORA_DIO0=4

    ;Screen
    -DHAS_SCREEN=1
    -DROTATION=1
    -DBACKLIGHT=38
    -DMINBRIGHT=160

    ;TFT_eSPI
    -DUSER_SETUP_LOADED=1
    -DUSE_HSPI_PORT=1
    -DST7789_2_DRIVER=1
    -DTFT_RGB_ORDER=1
    -DTFT_WIDTH=135
    -DTFT_HEIGHT=240
    -DTFT_BACKLIGHT_ON=1
    -DTFT_BL=38
    -DTFT_RST=33
    -DTFT_DC=34
    -DTFT_MOSI=35
    -DTFT_SCLK=36
    -DTFT_CS=37
    -DTOUCH_CS=-1
    -DSMOOTH_FONT=1
    -DSPI_FREQUENCY=20000000
    -DSPI_READ_FREQUENCY=20000000
    -DSPI_TOUCH_FREQUENCY=2500000

    ;SD Card
    -DSDCARD_CS=12
    -DSDCARD_SCK=40
    -DSDCARD_MISO=39
    -DSDCARD_MOSI=14

    ;Default I2C
    -DGROVE_SDA=2
    -DGROVE_SCL=1

    ;Default SPI
    -DSPI_SCK_PIN=40
    -DSPI_MOSI_PIN=14
    -DSPI_MISO_PIN=39
    -DSPI_SS_PIN=GROVE_SCL

    -DDEVICE_NAME='"M5Stack Cardputer"'

------------------------------------------------------------------------
⌨️ 4. Modify interface.cpp to Implement I2C / GPIO Multiplexing
File path:


boards/m5stack-cardputer/interface.cpp

The goal here is to let the keyboard I2C pins switch back to I2C mode when needed, and be released as normal GPIO pins when not needed so NRF24 can use them.


------------------------------------------------------------------------
4.1 Add I2C Pin Switching Functions

Add the following two functions to the file:


void setI2cPinsTooutput() {

    Wire1.end(); // OFF I2C
    
    pinMode(TCA8418_SDA_PIN, OUTPUT);
    
    pinMode(TCA8418_SCL_PIN, OUTPUT);
    
}

void setI2cPinsToI2c() {

    Wire1.begin(TCA8418_SDA_PIN, TCA8418_SCL_PIN);
    
    delay(5);
    
}


Description:

setI2cPinsTooutput()

Disables I2C and switches SDA / SCL to normal output mode

setI2cPinsToI2c()

Reinitializes I2C so the keyboard controller can be accessed again


------------------------------------------------------------------------

4.2 Add I2C Switching in the Keyboard Interrupt Handling Logic

Inside InputHandler(), in the UseTCA8418 branch, add:


if (digitalRead(11) == LOW) { setI2cPinsToI2c(); }

Then call the following after reading the keyboard event:

setI2cPinsTooutput();


------------------------------------------------------------------------
The key code block looks like this:

if (digitalRead(11) == LOW) { setI2cPinsToI2c(); }


// try to clear the IRQ flag

// if there are pending events it is not cleared

tca.writeRegister(TCA8418_REG_INT_STAT, 1);

int intstat = tca.readRegister(TCA8418_REG_INT_STAT);

if ((intstat & 0x01) == 0) { kb_interrupt = false; }


// if (tca.available() <= 0) return;

int keyEvent = tca.getEvent();

bool pressed = (keyEvent & 0x80); // Bit 7: 1 Pressed, 0 Released

uint8_t value = keyEvent & 0x7F;  // Bits 0-6: key value


setI2cPinsTooutput();

4.3 Recommended Placement

For cleaner structure, it is recommended to place these functions after getKeyChar() and before handleSpecialKeys():


char getKeyChar(uint8_t row, uint8_t col) {

    char keyVal;
    
    if (shift_key_pressed ^ caps_lock) {
    
        keyVal = _key_value_map[row][col].value_second;
        
    } else {
    
        keyVal = _key_value_map[row][col].value_first;
        
    }
    
    return keyVal;
    
}

void setI2cPinsTooutput() {

    Wire1.end(); // OFF I2C
    
    pinMode(TCA8418_SDA_PIN, OUTPUT);
    
    pinMode(TCA8418_SCL_PIN, OUTPUT);
    
}

void setI2cPinsToI2c() {

    Wire1.begin(TCA8418_SDA_PIN, TCA8418_SCL_PIN);
    
    delay(5);
    
}


------------------------------------------------------------------------
4.4 What This Change Does

After this change, the workflow becomes:

By default, GPIO8 / GPIO9 are released for external modules

When a keyboard interrupt is detected:

Temporarily restore I2C mode

Read the key event from TCA8418

After reading:

Disable I2C

Release the pins again

This helps reduce the conflict between the keyboard controller and NRF24.


------------------------------------------------------------------------
📡 5. Modify nrf_common.cpp to Avoid SPI Conflicts

File path:


src/modules/NRF24/nrf_common.cpp

Since CC1101 and NRF24 share the SPI bus, it is recommended to force the CC1101 chip select pin HIGH before initializing NRF24. This ensures that CC1101 will not incorrectly respond to SPI traffic.

------------------------------------------------------------------------

5.1 Add the Following Code to nrf_start()

At the beginning of the function, add:



pinMode(15, OUTPUT);

digitalWrite(15, HIGH);

The modified code block looks like this:



bool nrf_start(NRF24_MODE mode) {


    bool result = false;
    
    pinMode(15, OUTPUT);
    
    digitalWrite(15, HIGH);
    

    if (mode == NRF_MODE_DISABLED) return false;

------------------------------------------------------------------------
5.2 What This Does

Here, GPIO15 is the SS pin assigned to CC1101:

-DCC1101_SS_PIN=15

So before initializing NRF24, the following is executed:


pinMode(15, OUTPUT);

digitalWrite(15, HIGH);

This ensures that:

CC1101 remains deselected

CC1101 does not interfere with the SPI bus

NRF24 initialization becomes more reliable


------------------------------------------------------------------------
✅ 6. Summary

This modification allows Bruce firmware on M5Stack Cardputer to support:


CC1101

NRF24L01

LoRa

It does so by:

Adding board-level pin definitions

Implementing keyboard I2C / GPIO multiplexing

Releasing CC1101 chip select before NRF24 initialization

The two most important points are:

Handling the GPIO8 / GPIO9 multiplexing between the keyboard and NRF24

Keeping all inactive SPI device chip select pins HIGH


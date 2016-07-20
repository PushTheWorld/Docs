# OpenBCI V3 Data Format

This discussion of the OpenBCI data format only applies to OpenBCI V3 (2014). For OpenBCI V3, the OpenBCI board contains either a Atmel or ChipKIT microcontroller that can be programmed through the Arduino IDE and Microchip's MPIDE, respectively. The OpenBCI board has an on-board RFDuino radio module acting as a "Device". The OpenBCI system includes a USB dongle for the PC, which acts as the RFDuino "Host". The format of the OpenBCI data as seen on the PC is defined by a combination of the Arduino code on the OpenBCI board and of the RFDuino code running on the Host. So, if you don't like the data format defined here, feel free to change it!

### Proprietary ("RFDuino") vs Standard Bluetooth

OpenBCI uses RFDuino modules for its Bluetooth wireless connection. To achieve the highest data rates, OpenBCI supplies a RFDuino USB dongle that connects to the computer. When using this USB dongle, higher data rates can be achieved versus using a standard bluetooth 4.n BLE connection.

If you prefer to use a standard bluetooth connection (to a mobile phone, for instance), that software and data format has not yet been defined.

### Serial Setup

The RFDuino USB dongle (the RFDuino "Host") is connected to an FTDI USB<>Serial converter configured to appear to the computer as if it is a standard serial port running at a rate of 115200 baud using the typical 8-N-1. It is possible to run at faster baud (FT231XQ-R on dongle tested up to 1Mbaud), but 115200 is required if you want to upload code to the target uC.

### Startup
**8bit Board**

When the serial port for the dongle is opened by your PC, it will reset an 8bit Board. That's because the DTR signal from FTDI is getting sent over air, and the ATmega is configured as an Arduino UNO. You will see the familiar blink of the pin 13 LED. After a second or so, you will see the OpenBCI board generate a few lines of ASCII text displaying the device IDs of the ADS1299 and Accelerometer ending in $$$. If you are writing client software for the PC, your software must expect an ASCII string on startup, and look for the $$$ to know it is ready to receive commands.

**32bit Board**

The chipKIT on our 32bit Board does not go through a reset cycle when its serial port is opened. Because of this, it's possible to connect to the 32bit board and not know it's state. In this case, the terminal or application should write a **v** to the serial port, which causes the system to reset its state to default values.

### Initiating Binary Transfer

Once the OpenBCI has initialized itself and sent the $$$, it waits for commands. In other words, it sends no data until it is told to start sending data. To begin data transfer, transmit a single ASCII **b**. Once the **b** is received, continuous transfer of OpenBCI data in binary format will ensue. To turn off the fire hose, send an **s**. Both the Host and Device radios take notice of the **b** and **s**, and go into or out of streamingMode accordingly. That's right, the radio modules on both the OpenBCI board and the Dongle have two states:

* **!**streamingData
	* The radios appear to be a transparent UART betweeen the PC and target uC
	* [Command characters](https://github.com/OpenBCI/Docs/blob/master/software/01-OpenBCI_SDK.md) need some delay before and after sending to pass from PC to target uC
* streamingData
	* Device radio expects to get 31 bytes in each data packet from the uC
	* After 1 second of no transmission, or not getting 31 bytes in time, Device and Host will revert to **!**streamingData mode
	* Command characters can be sent from PC following timing protocol above

### Binary Format

Each packet contains a header followed by a sample counter, followed by 8 ADS channel data, followed by the three axes values of the accelerometer, followed by a footer. The accelerometer data are optional, and don't need to be sent with every packet when used. if unused, the bytes will read 0. This allows for user defined auxiliary data to be sent in the last six bytes before the footer. Also, there may be room for compressing more samples. Here are details on the format.

**Header**

* Byte 1: 0xA0  
* Byte 2: Sample Number

**EEG Data**  
Note: values are 24-bit signed, MSB first

* Bytes 3-5: Data value for EEG channel 1  
* Bytes 6-8: Data value for EEG channel 2  
* Bytes 9-11: Data value for EEG channel 3  
* Bytes 12-14: Data value for EEG channel 4  
* Bytes 15-17: Data value for EEG channel 5  
* Bytes 18-20: Data value for EEG channel 6  
* Bytes 21-23: Data value for EEG channel 6  
* Bytes 24-26: Data value for EEG channel 8  

**Aux Data**
* Bytes 27-32: 6 bytes of data defined and parsed based on the **Footer** below

**Footer**  

* Byte 33: 0xCX where X is 0-F in hex

#### Firmware Version 1 (2014 to Fall 2016)

Stop Byte | Byte 27 | Byte 28 | Byte 29 | Byte 30 | Byte 31 | Byte 32
--------- |:-------:|:-------:|:-------:|:-------:|:-------:|:------:
0xC0 | AX1 | AX0 | AY1 | AY0 | AZ1 | AZ0

AX1-AX0: Data value for accelerometer channel X
AY1-AY0: Data value for accelerometer channel Y
AZ1-AZ0: Data value for accelerometer channel Z

#### Firmware Version 2 (Fall 2016 to Now)

Stop Byte | Byte 27 | Byte 28 | Byte 29 | Byte 30 | Byte 31 | Byte 32
--------- |:-------:|:-------:|:-------:|:-------:|:-------:|:------:
0xC0 | AX1 | AX0 | AY1 | AY0 | AZ1 | AZ0
0xC1 | UDF | UDF | UDF | UDF | UDF | UDF
0xC2 | UDF | UDF | UDF | UDF | UDF | UDF
0xC3 | AC | AV | T4 | T3 | T1 | T0
0xC4 | AC | AV | T4 | T3 | T1 | T0
0xC5 | UDF | UDF | T4 | T3 | T1 | T0
0xC6 | UDF | UDF | T4 | T3 | T1 | T0

AX1-AX0: Data value for accelerometer channel X
AY1-AY0: Data value for accelerometer channel Y
AZ1-AZ0: Data value for accelerometer channel Z
AC: Code to interpret AV
AC | AV
--- | ---
'X' | AX1
'x' | AX0
'Y' | AY1
'y' | AY0
'Z' | AZ1
'z' | AZ0

**Note** UDF stands for User Defined.

### 24-Bit Signed Data Values

For the EEG data values, you will note that we are transferring the data as a 24-bit signed integer, which is a bit unusual. We are using this number format because it is the native format used by the ADS1299 chip that is at the core of the OpenBCI board. To convert this unusual number format into a more standard 32-bit signed integer, you can steal some ideas from the example Processing (aka, Java) code:

    int interpret24bitAsInt32(byte[] byteArray) {     
        int newInt = (  
         ((0xFF & byteArray[0]) << 16) |  
         ((0xFF & byteArray[1]) << 8) |   
         (0xFF & byteArray[2])  
        );  
        if ((newInt & 0x00800000) > 0) {  
          newInt |= 0xFF000000;  
        } else {  
          newInt &= 0x00FFFFFF;  
        }  
        return newInt;  
    }  

### 16-Bit Signed Data Values

The accelerometer data, if used, is sent as a 16bit signed value. We're using a similar scheme to convert these values into 32bit integers in Processing.

	int interpret16bitAsInt32(byte[] byteArray) {
    	int newInt = (
      	((0xFF & byteArray[0]) << 8) |
       	(0xFF & byteArray[1])
      	);
    	if ((newInt & 0x00008000) > 0) {
      		newInt |= 0xFFFF0000;
    	} else {
      		newInt &= 0x0000FFFF;
    	}
    	return newInt;
  	}

### 32-Bit Unsigned Time Stamp

To time stamp data, if used, is sent as a 32 bit unsigned integer representing time since the board was started in ms. We are using a different scheme to convert these values into 32 bit integers in Processing.

### Interpreting the EEG Data

Once you receive and parse the data packets, it is important to know how to interpret the data so that the EEG values are useful in a quantitative way. The two critical pieces of information are (1) the sample rate and (2) the scale factor.

For the sample rate, we set the default rate to 250 Hz. Faster rates are supported by the ADS1299, but the RFDuino wireless link and the serial limits might not be able to keep up with faster sample rates. If you give it a try, let us know how it worked!

For the scale factor, this is the multiplier that you use to convert the EEG values from “counts” (the int32 number that you parse from the binary stream) into scientific units like “volts”. By default, our Arduino sketch running on the OpenBCI board sets the ADS1299 chip to its maximum gain (24x), which results in a scale factor of 0.02235 microVolts per count. Because the gain is user-configurable (24x, 12x, 8x, 6x, 4x, 2x, 1x), the scale factor will be different. If the gain is changed, the equation that you should use for determining the scale factor is:


	Scale Factor (Volts/count) = 4.5 Volts / gain / (2^23 - 1);

Note that 2^23 might be an unexpected term in this equation considering that the ADS1299 is a 24-bit device. This equation is from the ADS1299 data sheet, specifically it is from the text surrounding Table 7. This scale factor has also been confirmed experimentally using known calibration signals.

Accelerometer data must also be scaled before it can be correctly interpreted. The equation used to scale Accelerometer data is as follows (We assume 4Gs, so 2mG per digit):


	Accelerometer Scale Factor = 0.002 / 2^4

### 16 Channel Data
Our 16 channel system allows for control of individual settings for all 16 channels, and data is retrieved from both ADS1299 IC at a rate of 250SPS. The current bandwith limitations on our serial radio links limit the number of packets we can send. To solve for this, we are sending data packet at the same rate of 250SPS, and alternating sample packets between the on Board ADS1299 and the on Daisy ADS1299. The method takes an average of the current and most recent channel values before sending to the radio. On **odd** sample numbers, the Board ADS1299 values are sent, and on **even** sample numbers, the Daisy ADS1299 samples are sent. When running the system with 16 channels, it is highly recommended that you use an SD card to store the raw (un-averaged) data for post processing.


###Room For Improvement

**Chage baud rate on the fly.**  This would help increase data rate. However, we have not been able to increase the Board UART baud beyond 115200. The Dongle baud has been tested up to 1Mbaud.

**Change protocol to meet other standards.** The over-air data is sent in packets (or frames, depending upon your preferred word). The maximum bytes allowed per packet is 32. We are reserving the first byte to use as a packet check-sum in our protocol. So the available bytes-per-packet, as far as the uC is concerned, is 31. The over-air protocol that the Dongle/RFduino Host gets is:

* Byte 1: Packet Counter
* Byte 2: Sample Number
* Bytes 3-5: Data value for EEG channel 1  
* Bytes 6-8: Data value for EEG channel 2  
* Bytes 9-11: Data value for EEG channel 3  
* Bytes 12-14: Data value for EEG channel 4  
* Bytes 15-17: Data value for EEG channel 5  
* Bytes 18-20: Data value for EEG channel 6  
* Bytes 21-23: Data value for EEG channel 6  
* Bytes 24-26: Data value for EEG channel 8  
* Bytes 27-28: Data value for accelerometer channel X  
* Bytes 29-30: Data value for accelerometer channel Y  
* Bytes 31-32: Data value for accelerometer channel Z

Our Host code removes the Packet Counter and adds the header and footer. It could be modified to work natively with other protocol specs for other signal processing software....

**Data Compression.** In situations where an increase in the sample resolution, or higher channel counts are desired, a data compression scheme can be implemented. As noted above, when sending **only** the ADS1299 values for 8 channels there are six unused bytes in the radio packet. It may be possible to, for example, increase the sample rate, and compress two samples worth of ADS data into a single radio packet. Or fit all 16 channels of data into a single packet and avoid the averaging that is currently used.

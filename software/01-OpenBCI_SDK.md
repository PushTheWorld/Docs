# OpenBCI Firmware SDK
The OpenBCI boards communicate using a byte string (mostly ASCII) command protocol. This Doc covers command use for the OpenBCI 8bit and 32bit boards. Some of the commands are board specific, where noted. Further this Doc covers the commands needed in order to alter the radio system. There have been several iterations of the firmware, the 8bit board runs `v0`, while the 32bit runs `v1` and as of summer 2016 runs `v2`.

## OpenBCI Command Protocol Overview

OpenBCI boards have two powerful microcontrollers on board, and come pre-programmed with the firmware. The RFduino radio link uses the Nordic Gazelle stack and library. The Board mounted RFduino is configured as a DEVICE. The microcontroller (ATmega328P or PIC32MX250F128B) has been programmed with firmware that interfaces between the ADS1299 (Analog Front End), LIS3DH (Accelerometer), micro SD (if installed), and RFduino (Radio module). The user, or application, controls the board by sending commands over wireless serial connection. You should have received a Dongle with the OpenBCI board. The Dongle has an RFduino running the Gazelle library configured as a HOST, and interfaces your computer through a Virtual Com Port (FTDI). (See the Radios portion for more info on the RFduino link).

On startup, the OpenBCI 8bit board (`v0`) sends the following text over the radio:

	OpenBCI V3 8bit Board
	Setting ADS1299 Channel Values
	ADS1299 Device ID: 0x3E
	LIS3DH Device ID: 0x33
	$$$

the OpenBCI 32bit board with firmware `v1` sends the following text over the radio:

	OpenBCI V3 16 channel
	ADS1299 Device ID: 0x3E
	LIS3DH Device ID: 0x33
	$$$

the OpenBCI 32bit board with firmware `v2` sends the following text over the radio:

	OpenBCI V3 8-16 channel
	ADS1299 Device ID: 0x3E
	LIS3DH Device ID: 0x33
	Firmware: v2.0.0
	$$$

Device ID info is useful for general board health confirmation. The $$$ is clear indication to the controlling application that the message is complete and the OpenBCI board is ready to receive commands. As of `v2`, there is an additional printout to indicate the exact firmware version.

Pay attention to timing when sending commands when using `v0` and `v1` firmware. There must be some delay before and after sending a command character from the PC (controlling program or user running a terminal). As of `v2`, this is no longer the case, send as fast as you can!

## Command Set
### Turn Channels OFF
**1 2 3 4 5 6 7 8**  
These ASCII characters turn the respective channels [1-8] off. The channel will read 0.00 when off during streamData mode. These commands work in and out of streamData mode.

###Turn Channels ON  
**! @ # $ % ^ &  * **  
These ASCII characters turn the respective channels [1-8] on. The channel will read ADC output values during streamData mode. These commands work in and out of streamData mode.

###Test Signal Control Commands  
**0 - = p [ ]**  
Turn **all** available channels on, and connect them to internal test signal. These are useful for self test and calibration. For example, you can measure the internal noise by sending **0** and connecting all inputs to internal GND.

* **0**  Connect to internal GND (VDD - VSS)  
* **-**  Connect to test signal 1xAmplitude, slow pulse  
* **=**  Connect to test signal 1xAmplitude, fast pulse  
* **p**  Connect to DC signal  
* **[**  Connect to test signal 2xAmplitude, slow pulse  
* **]**  Connect to test signal 2xAmplitude, fast pulse  

	**Note: Not all of the internal test connections are implemented here **

###Channel Setting Commands   
** x (CHANNEL, POWER_DOWN, GAIN_SET, INPUT_TYPE_SET, BIAS_SET, SRB2_SET, SRB1_SET) X **  
Channel Settings commands have six parameters for each ADS channel. To access Channel Settings, first send **x**. The OpenBCI board will then expect the next 7 bytes to be channel settings specific commands. The first byte is the channel number. (If you have the Daisy Module, you can select up to 16 channels to set). The following six ASCII characters are accepted as parameters to set. Lastly, sending **X** will latch the settings to the ADS channel.

**CHANNEL**

* **1 2 3 4 5 6 7 8**  for single board channel select
* **Q W E R T Y U I**  for selecting channels on the Daisy Module

**POWER_DOWN**

* 0 = ON (default)
* 1 = OFF    

**GAIN_SET**

* 0 = Gain 1
* 1 = Gain 2
* 2 = Gain 4
* 3 = Gain 6
* 4 = Gain 8
* 5 = Gain 12
* 6 = Gain 24	(default)


**INPUT_TYPE_SET**  
Select the ADC channel input source  


* 0        ADSINPUT_NORMAL     	(default)  
* 1        ADSINPUT_SHORTED          
* 2        ADSINPUT_BIAS_MEAS  
* 3        ADSINPUT_MVDD  
* 4        ADSINPUT_TEMP  
* 5        ADSINPUT_TESTSIG  
* 6        ADSINPUT_BIAS_DRP  
* 7        ADSINPUT_BIAS_DRN  

**BIAS_SET**  
Select to include the channel input in BIAS generation.

* 0 = Remove form BIAS
* 1 = Include in BIAS  (default)  

**SRB2_SET**  
Select to connect this channel's P input to the SRB2 pin. This closes a switch between P input and SRB2 for the given channel, and allows the P input also remain connected to the ADC.  

* 0 = Disconnect this input from SRB2
* 1 = Connect this input to SRB2  (default)  

**SRB1_SET**  
Select to connect all channels' N inputs to SRB1. This effects all pins, and disconnects all N inputs from the ADC.  

* 0 = Disconnect all N inputs from SRB1 (default)
* 1 = Connect all N inputs to SRB1  

**EXAMPLE**

User sends **x  3  0  2  0  0  0  0  X**

'x' enters Channel Settings mode. Channel 3 is set up to be powered up, with gain of 2, normal input, removed from BIAS generation, removed from SRB2, removed from SRB1. The final 'X' latches the settings to the ADS1299 channel settings register.

For firmware `v0` and `v1` it is required that you allow a time delay (>10mS) when setting the channel and parameters. As of `v2`, you may stack multiple channel settings together such as:

**EXAMPLE**

User sends **x1020000Xx2020000Xx3020000X**

This will set the first three channels.

###Default Channel Settings
**d** To set all channels to default  
**D** To get a report of the default settings send.

When you query the default settings, expect to get 6 ASCII characters followed by **$$$**

*Note: Users can change the default channel settings in the initialization function inside the OpenBCI library. Requires re-programming the board*

###LeadOff Impedance Commands  
**z (CHANNEL, PCHAN, NCHAN) Z**  
This works similar to the Channel Settings commands. For firmware `v0` and `v1` care must be taken to delay between sending characters, as of `v2`, you may send as fast as possible in a byte stream. Impedance settings have two parameters for each ADS channel. Impedance is measurable by applying a small 31.5Hz AC signal to the given channel.

* 0 = Test Signal Not Applied (default)
* 1 = Test Signal Applied  

**EXAMPLE**

User sends **z  4  1  0  Z**

'z' enters Impedance Settings mode. Channel 4 is set up to measure impedance on the P input. The final 'Z' latches the settings to the ADS registers.

###SD card Commands  
**A S F G H J K L**  
Send to initiate SD card data logging for specified time  

* A    =      5MIN  
* S    =      15MIN  
* F    =      30MIN  
* G    =      1HR  
* H    =      2HR  
* J    =      4HR  
* K    =      12HR  
* L    =      24HR  
* a	   =      about 14 seconds for testing

**j**  
Stop logging data and close SD file  

###Stream Data Commands  
**b**  
Start streaming data

**s**  
Stop Streaming data  


###Miscellaneous  
**?**  
Query register settings  
Read and report all register settings for the ADS1299 and the LIS3DH. Expect to get a verbose serial output from the OpenBCI Board, followed by **$$$**  

**v**

Soft reset for the Board peripherals. The 8bit board gets a reset signal from the Dongle any time an application opens the serial port, just like a arduino. the 32bit board doesn't have this feature. So, if you want to soft-reset the 32bit board (`v1` or `v2`), send it a **v**.


##16 Channel Commands
Currently, the Daisy Module is implemented only on the 32bit board. The Daisy Module adds 8 more input channels for a total of 16. These are the commands specific to controlling the ADS1299 on the Daisy Module.

###Turn Channels OFF
**q w e r t y u i**  
These ASCII characters turn the respective channels [9-16] off. The channel will read 0.00 during streamData mode. These commands work in and out of streamData mode.

###Turn Channels ON  
**Q W E R T Y U I**  
These ASCII characters turn the respective channels [9-16] on. The channel will contain ADC values during streamData mode. These commands work in and out of streamData mode.

###Select maximum channel number

**c**

Use 8 channels only. If the Daisy Module is attached, it will be unattached, and access to only channels 1-8 are available.

**C**

Use 16 channels.

*Note: On reset, the OpenBCI 32bit board will 'sniff' for the Daisy Module, and if it is present, it will default to 16 channel capability.*

#Firmware v2.0.0 New Commands

##Time Stamping

**<**  
Start time stamping and resynchronize command. The Host radio will send a **,** on send of the **<** back to the driver. Since the Host cannot send packets to the Device ad hoc, it may be helpful to know when the Host was actually able to send the command. If the Board is not streaming, then expect a response of `Time stamp ON$$$`; however if the board is streaming, then you will get a response in a different **stop byte** as described in the next software document titled _OpenBCI Streaming Data Format_.

**>**
Stops time stamping. If the Board is not streaming, then expect a response of `Time stamp OFF$$$`; however if the board is streaming, then you will get a response in a different **stop byte** as described in the next software document titled _OpenBCI Streaming Data Format_.


## Radio Configuration Commands
As of firmware version `v2`, a set of commands has been implemented to change the radio system and improve over-the-air programming of the main OpenBCI board.
In order to use the commands you must keep to the form of key-code-(payload) where key is`0xF0`, code is defined below and payload is optional and dependent on the code. For example, to get system status send `0xF0` then send `0x07`.

####Get Channel Number `0x00`
Returns either success or failure. If you get a failure, it will give you the host channel number and a failure message. If success it gives you both host and device channel numbers and a success message. Channel numbers can only be 1-25. The byte before EOT ($$$) will contain the channel number value in HEX.

####Set Channel Number `0x01`
Returns either success or failure. On failure it will ask you to verify the channel number and print a failure message. On success, it gives you both host and device channel numbers and a success message. Channel numbers can only be 1-25. The byte before EOT ($$$) will contain the channel number value in HEX.

**EXAMPLE**
User sends **0xF0** **0x01** **0x07** to set the system to channel 7

####Channel Set Override `0x02`
Returns either success or failure. On failure it will ask you to verify the channel number and print a failure message. If success it sends "Host Override" followed by the channel number. Channel numbers can only be 1-25. The byte before EOT ($$$) will contain the channel number value in HEX.

####Get Poll Time `0x03`
Returns success followed by the poll time in HEX.

####Set Poll Time `0x04`
Returns either success or failure. On failure it will send a "Communications Timeout" message. On success it sends success followed by the poll time in HEX. Values sent must be from 0-255 and must be sent in HEX. Defaults to 80ms.

**EXAMPLE**
User sends **0xF0** **0x04** **0xA0** which sets the poll time to 160ms.

####Set BAUD to Default (115200) `0x05`
Returns success and sends the BAUD rate in ASCII (115200).

####Set BAUD to High-Speed mode (230400) `0x06`
Returns success and sends the BAUD rate in ASCII (230400).

####Check if System is Up `0x07`
Returns success or failure. On failure it will send a "system is down" message. On success, it will send a "system is up" message.


##Unused ASCII Characters
These are currently unused characters across the OpenBCI platforms:


**~ ` 9 ( ) _ { } o O f g h k l ; : ' " V n N M , . / (space)**

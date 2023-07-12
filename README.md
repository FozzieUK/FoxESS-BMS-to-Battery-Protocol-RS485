# FoxESS-BMS-to-Battery-Protocol-RS485
FoxESS BMS to Battery RS485 Protocol between FoxESS V1 BMS and V1 HV2600 Battery

RS485 @ 115,200 baud, N,8,1 - **Big Endian**

This is an early work (in progress) in decoding the messages between the FoxESS BMS and HV Batteries

## Basic Protocol

The RS485 should be configured with 115,200 baud, N, 8, 1 

This mostly documents the running state, the startup is documented towards the end - at startup the BMS checks the packs for operation response, and then requests battery serial numbers and pack status before it moves into normal state.

## Messages

Unlike other BMS<>Battery protocols where the messages have an [SOI]...[EOI] this protocol differs somewhat..
  
As a general rule the BMS sends a 10 byte message which contains the [pack_id],[function],[length],[command bytes 1,2,3], followed by a 2 byte checksum and <0d>,<0a>.

### BMS Sends - 
  
  <01>,<03>,<00>,<00>,<00>,<08>,<44>,<0C>,<0D>,<0A>
    
  which is [Pack_id=1],[Function=3],[Length=0][CMD1=0],[CMD2=0],[CMD3=8],[CSUM Bytes 1=44,2=0C],[EOI=0D,0A]
    
When (Length)= 0 this is a request for data.


### Received message - The <pack_id> requested will respond with its data as -
    
  <01>,<03>,<10>,<00>,<00>,<67>,<E2>,<F7>,<B5>,<00>,<4C>,<0D>,<00>,<0C>,<FD>,<0B>,<0E>,<0A>,<D2>,<27>,<41>,<0D>,<0A>
    
which is [Pack_id=1],[Function=3],[Length=x10][Data Bytes 1...Length],[CSUM Bytes 1=27,2=41],[EOI=0D,0A]
  

Occasionally the BMS will send a broadcast message with pack_id = 0
    
    > 00,8C,FD,D4,81,0D,0A
    
    The packs do not respond to this message

This is the BMS instrutcing the packs what the minimum cell voltage is across the pack.
The command data for this example is 8C, FD     the first byte '8C' the '8' is number of packs, the 'C' is the msb and the 'FD' is the lsb
i.e. as it is big endian the value is 3,313mV

The BMS breaks the basic rule (terminated by 0d,0a) by sending a message that does not have the 0d,0a terminator, this appears to be an instruction to all the packs as well as a keep alive timer.
This is a sample of 3 of the messages -
    
    > 01,03,02,AB,02,81,15,15,92,F5 
    
    This is [1],[3],[02,AB,02,81],[15,15],[CSUM,CSUM] - command is currently unknown
    
    > 01,03,00,5E,00,92,11,11,EC,7C - Pack status
    
    The data is [00,5E,00,92] where 00,5E is SINT16 Current Sense (-0.94A),  00,92 is UINT16 Highest Cycles Count (94 cycles)
    
    > 01,03,0D,28,0D,1F,13,13,36,8C - Pack cell volts High and Low
    
    The data is [0D,28,0D,1F] where 0D,28 is UINT16 Pack Highest mV (3355mV) 0D,1F is UINT16 Pack lowest mV (3346mV)
    
It is **never** replied to, it does have a valid checksum with a small difference in calculation (see checksum section below)

### BMS Commands 

The BMS send command is comprised of three command codes, 

<Command 1>,<Command 2>, <Command 3>  

  they are tabulated here
    
| Command 1  | Command 2 | Command 3 |            Notes                            |
| :---------: | :-------: | :-------: | :----------------------------------------: |
|  	 22      |    00     |    05     |   Send pack statistics                      |
|    08      |    00     |    09     |   Send pack cell mV 1-9                     |
|    11      |    00     |    09     |   Send pack cell mV 10-18                   |
|    1A      |    00     |    08     |   Send pack temps                           |
|    00      |    00     |    08     |   Send pack status                          |
|    23      |    00     |    01     |   Request pack state (startup only)         |
|    E0      |    00     |    09     |   Request pack serial number (startup only) |

each message sequence is sent approx every 100mS, gaps between messages are approx 30mS - so each pack sequence is approx 0.5S apart
  
## Command by Command review
  
  ### BMS command <22,00,05> Pack Stats
  
  the send message to pack_1 looks like this -
  > 01,03,00,22,00,05,25,C3,0D,0A
  
  and the pack_1 response is this -
  > 01,03,0A,00,00,01,00,58,33,00,0B,20,82,04,05,0D,0A
  
  The Length is 0A (decimal 10), the data begins at byte 4 for 10 bytes, followed by checksum of 04,05 and terminated by 0d,0a
  
  The data packet is <00,00,01,00,58,33,00,0B,20,82>
  
  So far all 16 bit integers have been the first byte is most significant.
  
  the data packet contains the following information
  
| Byte1   | Byte2  | Byte3   | Byte4  | Byte5  | Byte6  |  Byte7,8  | Byte9   | Byte10  |    Notes                     |
|:-------:|:------:|:-------:|:------:|:------:|:------:|:---------:|:-------:|:-------:|:-----------------------------|
|	 <0x00> | <0x00> | Pack_id | <0x00> |  SoC   | Flags? |  Cycles   | f/w_ver |batt_type|                              |
|   0x00  |  0x00  |  0x01   |  0x00  |	 0x58  |  0x33  | 0x00,0x0B |  0x20   |  0x82   | Received packet              |
|   00    |   00   |   01    |   00   |	 88%   |00110011|  Cycles=11|   2.0   |   82    | Decoded info, SoC=88%, Flags=00110011, ver 2.0, battery_type=(0x82=V1HV2600, 0x84=V2), Cycles = 11 |
||
  Notes:
      Cycles Byte7=msb, Byte8=lsb - diviser 1 e.g (0x00*256)+0x90= 144 cycles
  
      f/w ver Byte9 top nibble is major, bottom nibble is minor version so 0x1F is 1.15 and 0x20 is 2.0
  
      batt_type  Byte10=0x82 is HV2600 V1, 0x83 is ECS4100 v1, 0x84 is HV2600 v2
  
      Flags, Byte6 assumed to be charge, discharge, balance etc.. - more testing required.

    

  
  
  ### BMS command <08,00,09> Cell mV 1-9
  
  the send message to pack_8 looks like this -
  
  > 01,03,00,08,00,09,04,0E,0D,0A
  
  and the pack_8 response is this -
  
  > 01,03,12,0C,FD,0C,FD,0C,FE,0C,FD,0C,FD,0C,FD,0C,FD,0C,FD,0C,FD,20,F1,0D,0A
  
  The Length is 12 (decimal 18), the data begins at byte 4 for 18 bytes, followed by checksum of 20,F1 and terminated by 0d,0a
  
  The data packet is <0C,FD,0C,FD,0C,FE,0C,FD,0C,FD,0C,FD,0C,FD,0C,FD,0C,FD>
  
  So far all 16 bit integers have seen the first byte as most significant.
  
  the data packet contains the following information
  
|  Byte1, 2  |  Byte3,4  |  Byte5,6  |  Byte7,8  |  Byte9,10 | Byte11,12 | Byte13,14 | Byte15,16 | Byte17,18 |  Notes     |
|:----------:|:---------:|:---------:|:---------:|:---------:|:---------:|:---------:|:---------:|:---------:|:-----------|
| (Cell_1mV) |(Cell_2mV) |(Cell_3mV) |(Cell_4mV) |(Cell_5mV) |(Cell_6mV) |(Cell_7mV) |(Cell_8mV) |(Cell_9mV) |            |
|  0x0C,0xFD | 0x0C,0xFD | 0xOC,0xFE | 0x0C,0xFD | 0x0C,0xFD | 0x0C,0xFD | 0x0C,0xFD | 0x0C,0xFD | 0x0C,0xFD |            |
|   3325mV   |  3325mV   | 	3326mV   |  3325mV   |  3325mV   |  3325mV   |  3325mV   |  3325mV   | 3325mV    |            |
||
  
  Notes:  
      Cell mV, 1st byte=msb, 2nd byte4=lsb, diviser = 1 e.g 0x0C,0xFE = 3326mV
 
 
   
  ### BMS command <11,00,09> Cell mV 10-18
  
  the send message to pack_1 looks like this -
  
  > 01,03,00,11,00,09,D5,C9,0D,0A
  
  and the pack_1 response is this -
  
  > 01,03,12,0C,FE,0C,FD,0C,FE,0C,FC,0C,FC,0C,FC,0C,FB,00,00,00,00,17,8C,0D,0A
  
  The Length is 12 (decimal 18), the data begins at byte 4 for 18 bytes, followed by checksum of 17,8C and terminated by 0d,0a
  
  The data packet is <0C,FE,0C,FD,0C,FE,0C,FC,0C,FC,0C,FC,0C,FB,00,00,00,00>
  
  So far all 16 bit integers have been the first byte is most significant.
  
  the data packet contains the following information
  
|  Byte1, 2  |  Byte3,4  |  Byte5,6  |  Byte7,8  |  Byte9,10 | Byte11,12 | Byte13,14 | Byte15,16 | Byte17,18 |  Notes     |
|:----------:|:---------:|:---------:|:---------:|:---------:|:---------:|:---------:|:---------:|:---------:|:-----------|
|   (C10mV)  |  (C11mV)  |  (C12mV)  |  (C13mV)  |  (C14mV)  |  (C15mV)  |  (C16mV)  |  (C17mV)  |  (C18mV)  | Cells 17,18| 
| 0x0C,0xFE  | 0x0C,0xFD | 0xOC,0xFE | 0x0C,0xFC | 0x0C,0xFC | 0x0C,0xFC | 0x0C,0xFB | 0x00,0x00 | 0x00,0x00 | not fitted | 
| 3326mV     |  3325mV   |   3326mV  |  3324mV   |  3324mV   |  3324mV   |  3323mV   |   n/a     |    n/a    | on HV2600  |
||


  ### BMS command <00,00,08> Pack Status
  
  the send message to pack_1 looks like this -
  > 01,03,00,00,00,08,44,0C,0D,0A
  
  and the pack_1 response is this -
  > 01,03,10,00,00,67,E7,F7,DC,00,ED,0C,FE,0C,FB,0A,6E,0A,32,58,27,0D,0A
  
  The Length is 10 (decimal 16), the data begins at byte 4 for 16 bytes, followed by checksum of 58,27 and terminated by 0d,0a
  
  The data packet is <00,00,67,E7,F7,DC,00,ED,0C,FE,0C,FB,0A,6E,0A,32>
  
  So far all 16 bit integers have been the first byte is most significant.
  
  the data packet contains the following information
  
| Byte1   | Byte2  | Byte3,4  | Byte5  | Byte6  |  Byte7,8  | Byte9,10  | Byte11,12 | Byte13,14  | Byte15,16  |    Notes      |
|:-------:|:------:|:--------:|:------:|:------:|:---------:|:---------:|:---------:|:----------:|:----------:|:--------------|
|	 <0x00> | <0x00> | Volts    |   HW   |  kWR   |    Amps   |   HimV    |   LomV    | HiCellTemp | LoCellTemp |               |
|   0x00  |  0x00  | 0x67,0xE7|  0xF7  |	0xDC  | 0x00,0xED | 0x0C,0xFE | 0x0C,0xFB |  0x0A,0x6E |  0x0A,0x32 |               |
|   00    |   00   |  53.198V |   00   | 2.2kwH |  -2.37A   |     3326  |   3323    |   26.70C   |   26.10C   |               |
 
  Notes:
      Volts Byte3=msb, Byte4=lsb - then double value, diviser 0.001 e.g (0x67*256)+0xE7= (26,599 * 2)/1000 = 53.198V
  
      kWR   Byte6 diviser, convert to decimal, diviser = 0.01 e.g (0xDC) = 220/100 = 2.2kwh
  
      Amps  Byte7=msb,Byte8=lsb signed, diviser = 0.01 0x00,0xED = 237/100 = -2.37A (discharge), >32768=charge
  
      Hi/LomV, 1st byte=msb, 2nd byte4=lsb, diviser = 1 e.g 0x0C,0xFE = 3326mV
  
      Hi/LoCellTemp 1st byte=msb, 2nd byte4=lsb, diviser = 0.01 e.g 0x0A,0x6E = 26.70C

      HW    Byte5 looks to be either hardware version or encoded manufacture date.
  
  
  
### Checksum

  The checksum is calculate using all bytes of the message (except the checksum and terminator 0a 0d) using CRC16 (modbus) Big Endian
 
  > **03,03,00,08,00,09,05**,EC,0D,0A  e.g.030300080009 = 0x05EC
  
  > **04,03,00,08,00,09**,04,5B,0D,0A    = 0x045B
  
  > **07,03,00,08,00,09**,04,68,0D,0A    = 0x0468
  
  > **08,03,00,08,00,09**,04,97,0D,0A    = 0x0497
  
  > **08,03,12,0C,FC,0C,FD,0C,FD,0C,FC,0C,FC,0C,FE,0C,FE,0C,FE,0C,FE**,23,56,0D,0A     = 0x2356


Note - The weird BMS message has a checksum but it calculates from the second byte of the message
> (1) 1,3,0,0,0,95,11,11,F4,70 - checksum calculation is 03000000951111 = 0xF470
> (2) 1,3,0c,df,0c,d3,13,13,3,4b - checksum calculation is 030cdf0cd31313 = 0x034b
> (3) 1,3,2,76,2,61,15,15,7f,10 

This appears to be an instruction from the BMS to set charge/balance flags - it is unclear at this moment as the message seems static apart from 2 bytes of message 3, work continues....

### STARTUP PROCESS

When the BMS is starting up, the following procedure is observed

BMS sends the following commands (with no response)

> -> 00,01,01,[CSUM],[0D0A]

> -> 00,01,01,[CSUM],[0D0A]

> -> 00,01,01,[CSUM],[0D0A]

> -> 00,00,03,[CSUM],[0D0A]

> -> 00,00,03,[CSUM],[0D0A]

> -> 00,00,03,[CSUM],[0D0A]

This may be part of the pack learning/allocating the pack_id's as battery addressing on the HV2600's is always fixed with batteries nearest the BMS running 1,2,3,4,5... as it gets further away (assume the master/slave ports are involved in this in some way)


Then the BMS sends

> -> FF, 10, 00,23, 00, 01, 01, 00, 8B, 59, 0D, 0A  ('FF' as pack is 'all' broadcast - the command is this pack functional? )

[pack_id], [command 10],[len=0],[cid1],[23],[cid2 0],[cid3 1],[pack_id 1],[0x00],[CSUM],[0D0A]

Response from pack 1 (pack_id from message)

> <- 01,10,00,23,00, 04, 30, 0D, 0A   (pack is operational, no response is not)

BMS repeats ( for (pack_id = 1; pack_id <= 8; ++pack_id) )

> -> FF, 10, 00, 23, 00, 01,[pack_id], 00, [CSUM], [0D0A]

Response from pack_id's

> <- [pack_id],10,00,23,00, [CSUM], [0D0A]

Then BMS sends 

> -> FF, 03, 00, 00, 00, 08, [CSUM], [0D0A]

> -> FF, 03, 00, 00, 00, 08, [CSUM], [0D0A]

' there is no response from packs to this message - I assume it means next command coming is status '00, 00, 08'


Then BMS Sends pack statistic requests

> -> 01, 03, 00, 00, 00, 08, [CSUM], [0D0A]

Response from pack 1

> <- 01,03,[LEN],[DATA....*LEN], [CSUM], [0D0A]

(this is the normal response to request for pack statistics)

BMS repeats ( for (pack_id = 1; pack_id <= 8; ++pack_id) )

Response from pack_id's

[pack_id],03,[LEN],[DATA....*LEN], [CSUM], [0D0A]

Then BMS Sends pack statistic and status requests in groups

> -> 01, 03, 00, 00, 00, 08, [CSUM], [0D0A]

> <- [pack_id],03,[LEN],[DATA....*LEN], [CSUM], [0D0A]

> -> 01, 03, 00, 22, 00, 05, [CSUM], [0D0A]

> <- [pack_id],03,[LEN],[DATA....*LEN], [CSUM], [0D0A]

BMS repeats group 00,00,08 then 22,00,05 ( for (pack_id = 1; pack_id <= 8; ++pack_id) )

each pack responds

Then BMS moves on to information request

> -> 01, 03, 00, E0, 00, 09, [CSUM], [0D0A] ' send serial number

> <- 01, 03, [LEN=18], [DATA... * LEN], [CSUM], [0D0A]


where data is ascii serial number i.e. if first 5 bytes are 36, 30, 32, 48, 32, 36, 32, the serial number is '602H262xxxx'

BMS repeats ( for (pack_id = 1; pack_id <= 8; ++pack_id) )

AT THIS POINT THE RUNNING STATE IS NORMAL i.e. normal cyclical polling of packs

  
**Disclaimer**: Any information on this wiki is informal advice, and is not supported nor endorsed by FoxESS. 
There is no warranty expressed or implied: you take sole responsibility for everything you do with this information and your equipment. There is no warranty to the accuracy of the content, nor any affiliation, or in any way a connection to FoxESS Co. Ltd


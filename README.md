# FoxESS-BMS-to-Battery-Protocol-RS485
FoxESS BMS to Battery RS485 Protocol between FoxESS V1 BMS and V1 HV2600 Battery

RS485 @ 115,200 baud, N,8,1 - mostly big endian !

This is an early work (in progress) in decoding the messages between the FoxESS BMS and HV Batteries

## Basic Protocol

The RS485 should be configured with 115,200 baud, N, 8, 1 

So far I am only documenting the running state, it is clear that battery serial numbers are not transmitted in the normal real time messages, and as the BMS has this information it must be part of the start up negotiation (to be documented)

## Messages

Unlike other BMS protocols where the messages have an <SOI>...<EOI> this protocol is different....
  
As a general rule the BMS sends a 10 byte message which contains the <pack_id>,<function>,<length>,<command bytes 1,2,3>, followed by a 2 byte checksum and <0d>,<0a>.

### BMS Sends - 
  
  <01>,<03>,<00>,<00>,<00>,<08>,<44>,<0C>,<0D>,<0A>
    
  which is <Pack_id=1>,<Function=3>,<Length=0><CMD1=0>,<CMD2=0>,<CMD3=8>,<CSUM Bytes 1=44,2=0C>,<EOI=0D,0A>
    
When <Length>= 0 this is a request for data.


### Received message - The <pack_id> requested will respond with its data as -
    
  <01>,<03>,<10>,<00>,<00>,<67>,<E2>,<F7>,<B5>,<00>,<4C>,<0D>,<00>,<0C>,<FD>,<0B>,<0E>,<0A>,<D2>,<27>,<41>,<0D>,<0A>
    
which is <Pack_id=1>,<Function=3>,<Length=x10><Data Bytes 1...Length>,<CSUM Bytes 1=27,2=41>,<EOI=0D,0A>
  

Occasionally the BMS will send a broadcast message with pack_id = 0
    
    > 00,8C,FD,D4,81,0D,0A
    
    The packs do not respond to this message

The BMS breaks the basic rule (terminated by 0d,0a) occasionally by sending a message that does not have the 0d,0a terminator (it may be sent in a pair as it aways proceeds the next BMS request message which does have a terminator), this may be an instruction to all the packs? or the pack it is sending the next request to?, for now I simply log the message and will analyse it in more detail later once the basic pack information has been decoded, this is a sample of 3 of the messages -
    
    > 01,03,02,AB,02,81,15,15,92,F5
    
    > 01,03,00,5E,00,92,11,11,EC,7C
    
    > 01,03,0D,03,0C,FD,13,13,B3,40
    
It is *always* proceeded by the BMS Send <pack_id> request
.

### BMS Send Commands - 

The BMS send command is comprised of three command codes, 

<Command 1>,<Command 2>, <Command 3>  

  they are tabulated here
    
| Command 1  | Command 2 | Command 3 |            Notes                            |
| :---------: | :-------: | :-------: | :----------------------------------------: |
|  	 22      |    00     |    05     |   Send pack stats                           |
|    11      |    00     |    09     |   Send pack cell mV 1-9                     |
|    1A      |    00     |    08     |   Send pack temps                           |
|    00      |    00     |    08     |   Send pack status                          |
|    08      |    00     |    09     |   Send pack cell mV 10-18                   |

### Command by Command review
  
  Starting with BMS command (** <22,00,05> Pack Stats**) to pack_1
  
  the send message to pack_1 looks like this -
  > 01,03,00,22,00,05,25,C3,0D,0A
  
  and the pack_1 response is this -
  > 01,03,0A,00,00,01,00,58,33,00,0B,20,82,04,05,0D,0A
  
  The Length is 0A (decimal 10), the data begins at byte 4 for 10 bytes, followed by checksum of 04,05 and terminated by 0d,0a
  
  The data packet is <00,00,01,00,58,33,00,0B,20,82>
  
  So far all 16 bit integers have been the first byte is most significant.
  
  the data packet contains the following information
  
| Byte1   | Byte2  | Byte3   | Byte4  | Byte5  | Byte6  | Byte7   | Byte8   | Byte9   | Byte10  |      Notes                             |
|:-------:|:------:|:-------:|:------:|:------:|:------:|:-------:|:-------:|:-------:|:-------:|:---------------------------------------|
|	 <0x00> | <0x00> | Pack_id | <0x00> |  SoC   | Flags? |Cyclesmsb|Cycleslsb| f/w_ver |batt_type|                                        |
|   00    |   00   |   01    |   00   |	 58    |   33   |   00    |   0B    |   20    |   82    | Received packet                        |
|   00    |   00   |   01    |   00   |	 88%   |00110011|  Cycles |  =11    |   2.0   |   82    | Decoded info, SoC=88%, Flags=00110011, ver 2.0, battery_type=82, Cycles = 11 |
  

**another example using pack_4-**

Starting with BMS command (** <22,00,05> Pack Stats**) to pack_4
  
  the send message to pack_4 looks like this -
  > 04,03,00,22,00,05,25,96,0D,0A

  
  and the pack_4 response is this - 
  > 04,03,0A,00,00,04,00,46,19,00,90,20,82,A3,A8,0D,0A
  
  The Length is 0A (decimal 10), the data begins at byte 4 for 10 bytes, followed by checksum of A3,A8 and terminated by 0d,0a
  
  The data packet is <00,00,04,00,46,19,00,90,20,82>
  
  So far all 16 bit integers have been the first byte is most significant.
  
  the data packet contains the following information
  
| Byte1   | Byte2  | Byte3   | Byte4  | Byte5  | Byte6  | Byte7   | Byte8   | Byte9   | Byte10  |      Notes                             |
|:-------:|:------:|:-------:|:------:|:------:|:------:|:-------:|:-------:|:-------:|:-------:|:---------------------------------------|
|	 <0x00> | <0x00> | Pack_id | <0x00> |  SoC   | Flags? |Cyclesmsb|Cycleslsb| f/w_ver |batt_type|                                        |
|   00    |   00   |   04    |   00   |	 46    |   19   |   00    |   90    |   20    |   82    | Received packet                        |
|   00    |   00   |   04    |   00   |	 70%   |00011001|  Cycles |  =144   |   2.0   |   82    | Decoded info, SoC=70%, Flags=00110011, ver 2.0, battery_type=82, Cycles = 144 |
  
  
### Checksum

  I have not yet looked inot decoding the check sum - that is one for later, for now if anyone is interested in looking at it i've included a few BMS send messages (same command different pack) and a pack response message for reference - it's normally byte addition, some kind of modulus and reverse (but this is Fox... have to be different!)
  
  > 03,03,00,08,00,09,05,EC,0D,0A
  
  > 04,03,00,08,00,09,04,5B,0D,0A
  
  > 07,03,00,08,00,09,04,68,0D,0A
  
  > 08,03,00,08,00,09,04,97,0D,0A
  
  > 08,03,12,0C,FC,0C,FD,0C,FD,0C,FC,0C,FC,0C,FE,0C,FE,0C,FE,0C,FE,23,56,0D,0A
  
  
**Disclaimer**: Any information on this wiki is informal advice, and is not supported nor endorsed by FoxESS. 
There is no warranty expressed or implied: you take sole responsibility for everything you do with this information and your equipment. There is no warranty to the accuracy of the content, nor any affiliation, or in any way a connection to FoxESS Co. Ltd


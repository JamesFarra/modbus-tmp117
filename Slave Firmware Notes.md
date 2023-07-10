### TMP117
- Averaging
- Oneshot mode
- Sleeping
- Wait 2ms before first conversion
- Make sure a conversion is done/ reject -256 degrees
- Just pass temp register back - don't do maths on ATtiny

### Setup
- Leave everything at reset values except the ones we care about
![Configuration register table][conftable.png]

Last Bit is: read, don't write, so a 0 means write, and a 1 read

1. Write 0x28 (address), then 0x1 (config register), then 0x0E20 (for my desired settings)

## Loop
1. Write 0x28 (address), then 0x00, then read 0x28, then read 2 bytes
2. Send over serial
3. Wait 1s, look in an LA

## MPCM
Multi-processor-communication mode would be lovely, as it uses an extra bit at the end to denote an address (seen by all) or data (ignored by all with the mode on, but a matching slave will turn filter off)

Unfortunately, Pis do not support 9-bit communication, so I have to use a different protocol. Peter from work suggested RS485, so that's what I'm going with.

## Modbus
There are 2 ways you can send Modbus over serial: Modbus-RTU and Modbus-ASCII. Modbus-RTU is *much* more common according to the internet, but requires the use of non-standard 3.5+ character "breaks" where the line is held low. This is easy enough to do on the ATtiny with all its easily accessible timers, but it's just too hard on a Pi. SO instead I'll be going with Modbus-ASCII. 

## Where did I get to for Peter?
- Got lost in the weeds with returning an array
- Serial outputs stuff twice, but no more (undefined behaviour)
- I have much to learn but I tried pretty hard, see the last bit of [[Writeup]]

## Getting ahead of myself
- Need to bench test *just* serial
- Loop sending a few bytes, and just one
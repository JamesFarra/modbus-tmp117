Before you give this a read, know that I got caught up in the fun of designing a PCB, and forgot to document until I actually had the first boards in my hands. So I've probably forgotten some of the decisions I made along the way. But practice makes perfect, right?

For background, I'm studying first-year Mechatronics, and my dad is an EE specialising in control systems for pumps to maximise efficiency.

## The problem
My dad and I have been observing our Daikin air conditioner acting inefficiently (or so we believe). We *think* the following mistakes are being made.
- In "Dry" mode, the most efficient way to operate is to run the indoor fan as slow as possible, to allow the evaporator to reach the lowest temperature possible (and thus the highest amount of moisture drawn from the air). The inverter-driven compressor should "lop along" as fast as is needed to keep this state. Split systems do this well, but our ducted system instead chooses to drive the compressor hard in short bursts, with a decent indoor fan speed. I've been told as a general rule that things going slower are generally more efficient.
- In "Cool" mode, the control loop should drive the compressor fast enough to cool the house in a reasonable time frame, but no faster. Again, split systems do this well, and seem to be optimised to a given room size, but ducted systems are a lot more varied in what's asked of them, and as such tend to oscillate between a very high and very low compressor speed, less efficient than just remaining at an appropriate speed for your desired cooling rate (friction losses are usually a square or cube relationship to output, though I haven't looked into the specifics).

### What about the larger problem?
The problem could be framed as "the A/C uses a lot of energy", and that's not entirely wrong (it runs on a 32A breaker, potentially 7kW or so at full tilt), and you could consider adding solar, but that feels like treating the symptom, not the cause. In a similar vein, you could step further back to:
- The house **requires active heating or cooling at all**, do we knock it down and build something designed with passive temperature management in mind?
- **We feel bad about using energy**, could this be solved with solar (and possibly storage)? Do we look at a potential other solution like storage hot water control based on available solar energy?
- And of course, just use the A/C less, and suck up the discomfort?
These seem worse in due to cost/comfort factors, but I thought I'd mention them to try and show that I think my horse is before my cart.

### Thermal model
So we want to build a new control scheme for our A/C? Well in that case we need a good idea of how our house behaves, and what it's currently doing. For our house we've identified the key temperatures to track:
- Outdoor air temp - the main influence on house temp
- Indoor air temp - This is the one we're trying to keep within reasonable bounds
- Slab temperature - The concrete slab below our house is the largest thermal mass, and responds very differently than the air and the rest of the house, so we should take it into account
- Roof air temperature - This is another thermal body that's separated from the main house air by insulation, but is still an influence.
In addition, it'd be nice to know:
- The intake temperature of the indoor unit - this is faster moving air drawn from all the parts of the house you're currently cooling, so it could be good to know.
- The temperature of the evaporator itself - Is it doing a good job of drying?
- The current draw of the compressor - and thus its power state


### Existing system
Our ducted system is controlled by a locked-down Android tablet on the wall, that runs [the SkyZone app](https://play.google.com/store/apps/details?id=au.com.daikin.skyzone), and talks to an access point in the roof that sends commands to the indoor unit (which then talks to the outdoor unit). According to the indoor unit, the AP looks like one of those units you usually see on a ducted system on a wall with lots of buttons.

![The wall unit][wall_unit.jpg]
![The roof unit (Access point)][roof_unit.jpg]

As you might be able to see, the tablet has died, which gives me an excellent opportunity to come up with a system to replace it. IMO the current UI sucks, it wastes a bunch of space looking "pretty", and requires menu diving to do stuff like timers. I'd like my solution to replace the tablet. The [app page](https://play.google.com/store/apps/details?id=au.com.daikin.skyzone) has some screenshots to give you an idea of how much it sucks (also check out that 2.8 review score, looks like I'm not alone)


## Requirements
So what do I need my solution to do? Based on the above we came to:
1. **Measure the key temperatures** and log them so we can build a thermal model of our house. With [GIGO](https://en.wikipedia.org/wiki/Garbage_in,_garbage_out) in mind, we want, in descending importance:
	1. Reliability, if a sensor goes down, it could be a while until someone notices when datalogging. Giant holes in our data makes analysis harder, potentially impossible.
	2. Repeatability - The better this is, the more our data can be trusted over time
	3. Variance - The more the sensors agree with each other, the better we'll be able to compare them.
	4. Accuracy - I'm assuming here that maybe some physics maths will be at play, so a better result here means our maths is more accurate.
2. Once we have the model, **Control the A/C**
3. It shouldn't cost more than a few hundred dollars - this is about what I'd imagine this kind of thing would cost if you bought it as a product.
4. Provide a UI to show temperatures, logging, and controls. It should replace the 7" tablet on the wall.


## Previous Attempt
I'll let you in on a secret, this is actually attempt two! Before the Pi shortage I thought nothing of using a Zero W to hook up to a few DS18B20s. The DS18B20 is a pretty cool chip, it's a temp sensor that sits on a OneWire bus, and has a 40-bit unique ID or something like that, so you can just buy a bunch, figure out which is which, then stick labels on them. I ran the signals through a couple pairs of a CAT6 cable we had spare from an Ethernet install. They are reasonably accurate (spec is ±0.25°C), but the units I got were as much as a deree apart without calibration:

![DS18B20 Response Graph][old-graf-hd.png]

Running linux means a bunch of things I needed were already there:
- File system for storing a *ton* of measurement data
- `ssh` and `sftp` for uploading/running code while it's in my roof.
- Utilities like [Flask](https://palletsprojects.com/p/flask/) can really easily serve data to a webpage.

Unfortunately, the DS18B20s started dropping like flies over a few months. Perhaps the sensors were counterfeits (prolific with that particular chip), or were never intended to be used in that way. This means **I have cables in the roof/walls that I'd like to re-use** (they were a PITA to put in)

Here's me testing the first system, it's a protoboard on top of a Pi with some screw terminals in a 3D printed case:
![The guts of the "Master" node of my last solution][guts.jpg]

Here's the enclosures I CADed up and printed. The black one went in the roof (with holes for screws to screw into a beam) and the white one was supposed to match a wall plate and look decent as it would be visible on the wall:
![The open enclosures][open.jpg]

This is how it looked installed:
![The previous temperature sensor][old_temp.jpg]

## The Solution
![This is where the fun begins][fun.jpg]

### Measure the key temperatures
Selecting the right sensor is super important, luckily there is one standout contender - the TI TMP117 ([datasheet](https://www.ti.com/lit/ds/symlink/tmp117.pdf)). With the incredible ±0.1°C absolute accuracy, repeatability of 1 LSB (0.0078 degrees!!) when used correctly, and most units falling between 0.04 degrees of each other. Seriously, if anyone can find something better than this for under 50 bucks, I'll eat my metaphorical hat (my Tilley hemp hat will probably win).

The only downside is that it communicates over I²C, a short range protocol that can only be stretched out to about half a meter. This means I'll have to have a bridge of sorts between it and a more robust bus like RS485.

### Controlling the A/C
The access point that the tablet talks to actually justs accepts HTTP requests for setting/returning A/C parameters. Luckily, thanks to Ben Neumeister, the thorough [PyZone API](https://github.com/BenNeumeister/daikinPyZone) exists already for Python, and should mean that full control of all user-accessible controls for the AC are broken out for my code to interface with.

### UI and datalogging
A good way to simplify datalogging and UI is to just use linux on a Raspberry Pi. Sure, it's overkill, you could make it work on a fast Cortex-M microcontroller, but as I mentioned above, it simplifies a great many things
- Instead of a complex FSMC peripheral and parallel display, low latency, high resolution/depth displays are made easy by HDMI.
- Instead of using a microntroller-specific file system, just use Python/C++s standard file interface tools.
- Following on from the ["everything is a file"](https://en.wikipedia.org/wiki/Everything_is_a_file) philosophy, you can actually write to the framebuffer directly, this will make porting [LVGL](https://lvgl.io/) easier for me.

### System architecture
Here is the final architecture I decided on:
![Overarching design][overarching.excalidraw.png]
This meant I needed to design the master and slave boards, pick a touchscreen, and create some enclosures to put everything in.

I've done a couple board designs in the past (and some of them even worked!) so I'm comfortable making up my own PCBs for this, in order to keep things small, and to meet the slightly odd requirements. Yes I'm aware that a few Pico Ws and PiicoDev TMP117s would likely be cheaper and a lot easier, but I wanted to reuse the wiring in the walls, and I've never worked with my own comms protocol or RS485 before, so it sounded like fun (and [what's the point if we can't have fun?](https://thebaffler.com/salvos/whats-the-point-if-we-cant-have-fun)). All that I've learnt about PCB design has come from Michael Ruppe's excellent KiCad Zero to Hero series on YouTube, trial and error, and advice from friends. I get that I might be brushing over things in this process, as it's hard to know the target audience of write ups like these, so if you're not familiar, The Factory has a miniseries on the full process that goes into more detail about the process from idea to product, and they'll do a better job of explaining than I will (who knows, maybe I'll get better with practice).

### PCB Design
#### Constraints
1. Use the manhattan layout (front side of board goes E-W or N-S) the bottom goes 
2. Try to match CAT6 characteristic impedance (100 ohms) to avoid reflections.
3. Keep the "branches" or "drops" short.
4. Use only 0603 or larger components. I've hand-placed hundreds of 0402s for a project, never again. The TMP117 is unfortunately only available in tiny packages, but then it's only one thing to go wrong.

#### Slave Boards
These basically just need to sit on the bus and respond to the master with temperature data when asked. Here's what I came up with:
![Slave board schematic][schem.png]
![Slave board PCB layout][slaveboard.png]

##### Part Choices
- ATtiny202 - This was a lovely part to pick, the tinyAVR range of microcontrollers, whilst only 8-bit, are dead-simple to implement. instead of external crystals, flash and 10 decoupling caps, you only need one decoupling capacitor, that's it.  The 202 is the most basic available I believe, with only 5 GPIO, and 2K of flash, but a quick test of compiling some hello world code with the `Wire` and `Serial` libraries confirmed that this would be enough. [SpencerKonde's excellent megaTinyCore](https://github.com/SpenceKonde/megaTinyCore) means you can program these with the Arduino IDE, and you actually get a bit *more* functionality than with an official arduino board (when it comes to which peripherals get used for which helper functions like `PWM`. The PiicoDev Smart modules are based of this series, I'm definitely not the first to use them as an I/O bridge
- The RS485 transciever is actually a SN75HVD3082EDR ([datasheet](https://www.ti.com/lit/ds/symlink/sn65hvd3082e.pdf?ts=1639755616460&ref_url=https%253A%252F%252Fwww.ti.com%252Fsitesearch%252Fdocs%252Funiversalsearch.tsp%253FlangPref%253Den-US%2526searchTerm%253Dsn65hvd3082edr%2526nr%253D16)) Even though the schematic suggests otherwise (it's a similar part with the same pinout). I really need to get a better KiCad library going so I can stop doing janky stuff like this. In hindsight there are cheaper chips than this, but they all kinda do the same job (allowing a standard UART interface to hook up to an RS485 bus).
- There's pullups, test points, and a TVS diode to hopefully stop these boards dying on me. Real optical isolation would probably be better here, but it won't be in an electrically noisy environment.

##### Other Design Decisions
- I chose to use software flow control with a spare GPIO. This is because the ATtiny can only perform hardware flow control on GPIO0 which is also the programming pin. Since I don't have a HV programmer that can smack this pin with 12V when programming in order to force it back to being the programming pin, and I don't trust myself to program it right the first time, and the data rate would be low, software flow control is probably fine. I included a solder jumper you can cut and bridge to switch to hardware FC later.
- It's a 4 layer board with the middle 2 layers tied to GND. The RS485 connection passes along the back of the board with a differential pair tuned to 100 ohms (about the same as CAT6). A short stub branches off these lines towards the front of the board where it connects with the transciever.
- The TMP117 is as thermally isolated as I could get it over in the bottom right, with no copper fills except for that big square polygon connected to the thermal pad (but not soldered, [TI only recommends soldering if you can then calibrate out the thermal offset introduced by the bending of the sensor die](https://www.ti.com/lit/an/snoa986a/snoa986a.pdf?ts=1670572919998).). This should mean that it measures the temperature of still air really well. Or if I want to measure the slab, a bit of aluminium or Blu-tak should suffice.

##### Ideas for a revision 2
- Proper mounting holes that place no stress on the temp sensor, for now I just won't use the right hand one.
- A different form factor for waterproof probes.
- Better board aesthetics? Silkscreen positioning away from vias
- An address switch to set the address without having to hardcode it into the software.
- A power measurement PCB of a similar form factor that uses a good ADC to measure the differential voltage across a current clamp.
- Use the alert pin on the TMP117 to tell the ATtiny when to read the measurement and return it

#### Master Board
This one just has to take the UART pins from the 40-pin GPIO of the Pi, and hook it into the RS485 bus. Here's what I came up with:
![Master schematic][masterschem.png]
![Master Board Layout][masterboard.png]
It has the same transceiver chip, and a buffer IC to sink current for the indicator LEDs (good to know when something's happening, and Pi pins can't sink/source much at all IIRC). It uses a tiny MOSFET for logic level shifting, as the transciever chip wants 5V to my knowledge.

### Assembly
I ordered the boards and solder paste stencils from JLCPCB, with ENIG surface finish on the slaves (due to the exposed thermal pad) and controlled impedance (they have a calculator that makes it easy to pick the right stackup).

I exported the BOM from KiCad, and searched constructed a "MyList" on Digi-key of the parts required for one assembly. That way I can add, say, 3 slaves and 2 masters worth of parts to an order and qualify for free shipping. I usually go with at least 1 extra board's worth of components if you stuff up during assembly. Eventually I'd like my own stock management system, but that's a huuuge project that has a few projects in front of it for me

Once the boards, stencils, and components had arrived. I assembled one master board and 2 slave boards, as a small-scale test to see if everything worked, or if I needed to make revisions. 


![A microscope view of a slave board after pasting][microscope_paste.jpg]
![A microscope view of the master board][microscope_master.jpg]
![All the boards lined up][lined_up.jpg]

### Programming
Programming the microcontrollers is pretty easy. They only use a single "Unified Programming and Debugging Interface (UPDI)" pin for programming, and all the hardware you need is a USB-to-serial converter, a diode, and a resistor. I made up a little programming board in this fashion a while back, shown below. You can find out more about this process from [the GitHub page of SpencerKonde's excellent MegaTinyCore](https://github.com/SpenceKonde/megaTinyCore#updi-programming).

And that was the first hurdle I ran into with this project. I *thought* that I had done a dry-run compilation of the I2C and serial libraries in the Arduino IDE to work out how much flash memory I needed (and therefore which chip to pick), and worked out that the smallest value of 2KB would be enough, but *just the `Wire` library* went over budget by about 1.5KB. Uh-oh.

So I need to make my code smaller, a *lot* smaler. I took a squiz at the internals of the Wire library, and it was all a bit too abstract for me to make sense of, so I started looking into writing my own effectively, by manipulating registers directly. 

[This blog post](https://www.omzlo.com/articles/baremetal-programming-on-the-tinyavr-0-micro-controllers) got my foot in the door on that front, and in not time I was turning a pin high and low with just my programmer, text editor, and makefile.

### Writing the Firmware
#### Talking IIC
The first element I needed to get working was communication with the sensor. After much datasheet reading, my code looks something like this:
```c
void waitForWrite(){
    while (!(TWI0.MSTATUS & 0b00100000)){}; // wait for write to complete
}

void waitForRead(){
    while (!(TWI0.MSTATUS & 0b10000000)){}; // wait for read to complete
}

void readTemp(){
    TWI0.MADDR = (0x90); // Setup a write to the sensor
    waitForWrite();
    TWI0.MDATA = 0x01;
    waitForWrite();
    TWI0.MDATA = 0x0E;
    waitForWrite();
    TWI0.MDATA = 0x20;
    waitForWrite();
    
    _delay_ms(250); // sensor needs 250ms to do a measurement and average it
    
    TWI0.MADDR = (0x90); // Setup a write to the sensor
    waitForWrite();
    TWI0.MDATA = 0x00; // Tell it you want reg 0x0 (the temp register)
    waitForWrite();
    TWI0.MADDR = (0x91); // Set up a read
    waitForRead();
    tempHigh = TWI0.MDATA;
    waitForRead();
    TWI0.MCTRLB = 0x3;
    tempLow = TWI0.MDATA;
}
```

A bit ugly, but gets the job done. It was based on the sequence described in the TMP117 datasheet:
![TMP Sequence][tmp_protocol.jpg]

When I saw the first ACK in my Logic software it made me veeeery happy :)
![ACK Recieved!][ack.png]

#### Picking a protocol
So I knew the type of communication (UART) and electrical specs (RS485), but I didn't know how to get data from multiple slaves over a single line, i.e. what the protocol would be.

At first blush, the MPCM mode of the ATtinys seemed perfect, since my needs were very simple. It uses an extra bit at the end to denote an address (seen by all) or data (ignored by all with the mode on, but a matching slave will turn filter off)

Unfortunately, Pis do not support 9-bit communication, so I have to use a different protocol. 

A well-versed EE coworker recommended Modbus, an ancient protocol that's still in heavy use in industry decades after it's inception in the '70s. He argued that it's an open standard that's implemented in the most places out of any serial bus protocol, and that implementing some other person's protocol means that you don't have to explain it from scratch, you can just point to the Modbus docs. Speaking of those, the ones I read can be found below:
[Modbus Application Protocol v1.1b3](https://modbus.org/docs/Modbus_Application_Protocol_V1_1b3.pdf)
[Modbus over serial line v1.2](https://www.modbus.org/docs/Modbus_over_serial_line_V1_02.pdf)

#### Modbus
There are 2 ways you can send Modbus over serial: Modbus-RTU and Modbus-ASCII. Modbus-RTU is *much* more common according to the internet, but requires the use of non-standard 3.5+ character "breaks" where the line is held low. This is easy enough to do on the ATtiny with all its easily accessible timers, but it's just too hard on a Pi. SO instead I'll be going with Modbus-ASCII. 

![Modbus-ASCII at work][modbus_ascii.excalidraw.png]

So I whipped up some C code to do just that:
```c
uint8_t* parseChar(uint8_t inputByte){
    uint8_t outputBytes[2];
    uint8_t splitByte[2];

    splitByte[0] = inputByte >> 4; // shift right 4 bits to get the first character
    splitByte[1] = inputByte & 0x0F; // mask with 0b00001111 to get the second character

    for (uint8_t i = 0; i < 2; i++){
        if (splitByte[i] < 0xA){
            outputBytes[i] = (splitByte[i] + 48); // if it's 0-9, that's 48-56 on the ASCII table
        } else{
            outputBytes[i] = (splitByte[i] + 56); // if it's A-F, that's 65-70 on the ASCII table
        }
    }
    return outputBytes;
}
```

Next up is getting my code to follow the state diagram that describes the protocol:
![Protocol state diagram, now with extra scribbles][states.excalidraw.png]

## Pi<->ATtiny Unit Test
[Linux Serial Ports Using C/C++ | mbedded.ninja](https://blog.mbedded.ninja/programming/operating-systems/linux/linux-serial-ports-using-c-cpp/)



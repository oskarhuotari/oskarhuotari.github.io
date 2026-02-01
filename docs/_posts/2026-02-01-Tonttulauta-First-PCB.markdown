---
layout: post
title:  "My first ever custom PCB, Tonttulauta"
date:   2026-02-01 16:29:00 +0200
categories: update PCB
---

I am participating in the Finnish CanSat competition this spring. In a CanSat, especially in a drone CanSat, space is truly limited. The PCB intended for this competition was orders of magnitude too large. I already have experience with protoboard flight computers and they absolutely suck. Solder is all over the place and when something goes wrong, debugging is a nightmare. They also waste a lot of space. The ready made modules usually have a large surface area for a few square millimeter sized chips. Thus, I decided to try to design a custom pcb.

The competition also has a very tight schedule so I decided to go with a carrier board. In other words I'll use the very powerful teensy 4.0 and make a very small board that can do the following:
-Log data
-Measure temperature, light, acceleration, movement and magnetic fields

Our team's mechanical engineer told me that the optimal size would be about 70x20 mm. So those were the requirements. When I started designing the board, a lot of the communication protocols were already familiar for me: uart, spi, i2c. However, I wasn't familiar with concepts like decoupling or pull up/down resistors. I decided to go with KiCAD as it was open source and highly appraised. I bought an udemy course, but it turned out to be very shallow. I was like a plumber trying to learn their craft by learning about wrenches and pliers. Luckily, there were a lot of good tutorials on Altium's YT channel about spi and i2c wiring .I also learned about grounding through this absolute gem: 
- https://www.youtube.com/watch?v=ySuUZEjARPY

I do wish I would have watched it earlier. When I watched the video, I had already routed all of my signals on all 4 layers. A more experienced PCB designed could have surely routed this on a 2 layer board, but I definitely wasn't one. My 4 layer stackup was signals, ground, power, signals. That's not bad, but I could have lowered EMI by fusing together the last power and signal layer. I didn't want to change everything so I decided to just add a GND fill to the last layer as from the video, I had learned that electricity doesn't actually travel in wires. It travels around them. I also learned about decoupling and slowing signal edges by adding series resistors. Though, to be honest, I chose ballpark values by intuition and not by doing complex calculations. They seemed too scary.

Then it was time to actually orded the PCB. It had some small components so I decided to order a ready assembled one from JLCPCB. I choce that specific vendor, because it was recommended to me and the integration with KiCAD worked well. Ordering it was quite terrifying. It was my own design and a single mistake could ruin the whole thing. Mind you, this whole CanSat project is self-funded (although we have tried actively to look for sponsors).

Finally, the PCB arrived. I was thrilled when at least something, the LEDs, worked. I also wrote code to interface with all of the sensors on board. For the first time ever, I actually read the component datasheets. Writing code was a breeze for most of the sensors. I realised that writing own libraries for the sensors is a ton easier than using the ready made libraries. I get to make the API work exactly like I wish. It's very telling that when I tried to use a flash chip library, it wouldn't even initialize. But then when I programmed my own lib, I could go step by step to verify that everything was working.

Now I must confess something. I spent hours trying to debug the magnetometer, because it wouldn't work. Looking back I see multiple issues with my workflow:
- Firstly, I hadn't done the necessary calculations to evaluate whether I could actually get meaningfull data from the sensor. I have motor wires nearby and later I concluded that they produce too high a magnetic field for getting meaningfull data. So the whole magnetometer is useless.
- Secondly, I hadn't read the datasheet well before designing the PCB. When I later looked at the datasheet, I saw that it contained uncomfotably little information. Moreover, the datasheet had false statements about which SPI modes the device can use. I also think that the sensor ID is wrong.
- Thirdly, I tried to repair the PCB as ChatGPT told me that the line needed to be pulled down (I had it pulled up and the datasheet stated only "CS: For I2c, Pull to VCC"). This was total garbage and even a bad engineer would know that ChatGPT's response was BS. I still tried to change the 0603 resistor and probably broke the PCB using too high hot air temperature (although the solder paste wouldn't melt before 400 C?).

---
TLDR; I designed my first ever PCB. Here's some things I learned:
- Electricity doesn't travel in wires and if you have an x layer pcb with fast signals, you actually have x/2 layers for signals and power
- ChatGPT can give insane advice; it's advice should not be taken at face value
- Always read the datasheets of the components you're going to use. A sensor from a reliable brand (e.g. Bosch) can save you from a lot of troubles.
- Writing your own libraries for sensors is a great idea. You learn a lot about register, binary and hex values and communication protocols
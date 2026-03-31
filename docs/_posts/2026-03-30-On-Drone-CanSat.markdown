---
layout: post
title: "On drone CanSat"
date: 2026-03-30 12:00:00 +0200
categories: CanSat PCB code
---

# DRONESAT

Our team set out to build a CanSat that will fly back to the launch site. In the CanSat competition, teams build a standard soda can sized "satellite" that will be launched to an altitude of about 1 km. I was the team leader, and I was responsible for the PCB design ([see here](https://github.com/oskarhuotari/Tonttuboard-PCB)). I also wrote the low level code that interfaces with the sensors and the flash chip, some stabilization code, and tried to tune our PIDs. Most of the code (except Tonttulib) was written together with Nino Heino.

### Mechanical design
Another team member, Aleksi Huhtala, was responsible for the mechanical design, manufacturing and CAD modelling. The frame was made from 2 mm sheet aluminum and it had parts attached with pop rivets and bolts. Grid fins were added to the design in order to stabilize the drone's descent before powered flight. The motor arms were in the closed position inside the rocket, which launched our CanSat. They have rubber bands which are designed to pull them into the open position after deployment. Aleksi also chose the motors, the batteries, and the ESCs. Here's a very beautiful image of the initial design:
![Test]({{ '/assets/images/dronesatCAD.png' | relative_url }})

### Electronics design
The Tonttuboard had an IMU, barometer and a flash chip. It was designed to be as simple as possible so it was a carrier board for the Teensy 4.0. We wrote our code in the Arduino IDE. We also had 4x BlHeli_s ESCs (electronic speed controllers) and 4x brushless 11000 kv motors. Tonttuboard talks to the ESCs through PWM signals (very fast on and off switching). In addition, we had 2x 7.4 v LiPos. Our data was transmitted through ESP-NOW as we had a XIAO ESP32s3 as a bridge between the Teensy and the groundstation Wroom ESP32U. 
![Test]({{ '/assets/images/dronesatTonttuboard.png' | relative_url }})

### Sensor fusion
We decided to go with a simple complementary filter. We also tried a kalman filter (without a magnetometer) but there was a lot of cross coupling between pitch and roll, which contributed to violent oscillations around the upright position. Unlike many other 6x sensor fusion filters, the complementary filter has very little tuning. Here's how the angles can be obtained from the accelerometer:
```
    float rollFromAcc  = atan2f(ay, az);
    float pitchFromAcc = atan2f(-ax, sqrtf(ay * ay + az * az));
```
These return the angles in radians. However accelerometer readings are very noisy. Why not just use gyro integration then? Well it's not noisy, but it has slow drift, which the accelerometer does not have. That's actually where the complementary filter jumps in. It's configured to trust the gyro more, but also to correct the drift via the accelerometer.
```
roll  = 0.985f * (roll  + gx_rads * dt) + 0.015f * rollFromAcc;
pitch = 0.985f * (pitch + gy_rads * dt) + 0.015f * pitchFromAcc;
```

In the drone, the sensor fusion was done 500 hz and the gyro was configured to a 4khz output data rate and had a 100hz digital low pass filter. The accelerometer had a 50 hz digital low pass filter, and it's output data rate was 2 khz.

### Motor mixing
Drones have some way of knowing what they should do -- obviously. For example they know that they are tilted 5 degrees in the roll axis and thus I should make some motors spin faster and some slower. This is done in the motor mixing phase. There are many possible configurations, but the most common ones are an X frame and a + frame. In a + frame, the motors are aligned with the axes and it's a bit simpler. That's why we went with it. Yaw is applied to every motor, with a plus or minus sign depending on the motor's spin direction. Here's our motor mixing:
```
float m1 = baseThrust + rollOut  + yawOut;
float m2 = baseThrust + pitchOut - yawOut;
float m3 = baseThrust - rollOut  + yawOut;
float m4 = baseThrust - pitchOut - yawOut;
```
The base thrust can be controlled by a pilot or in our case it was controlled by a velocity PID system that tried to keep the descent velocity at 5 m/s (the slowest allowed in CanSat competitions).

### PID 
See for example [this video](https://www.youtube.com/watch?v=JBvnB0279-Q) (or just go read the Wikipedia if you're smart enough for that)

### The cascaded PID stabilization system
We had a world of troubles getting the drone's stabilization to work. First we tried an angle only PID system, but the drone did nothing and flipped or oscillated like crazy. I don't know if this was due to our 11000 kv motors, which could have made the IMU values noisy?

A rate-only PID system reacts to raw gyro values. It tries to oppose all movement but doesn't care about the drone's orientation. At some point I decided to try it out. Suddenly the drone was almost hovering and we could fix and then verify the motor mixing was correct. We could actually let go of the drone -- for 2 seconds but still. 

The rate PID system was the only thing that seemed to help so we had to keep it. We just added a angle PID system on top of it that outputs setpoints for the rate PID. It works like this:
```
angle_error = target_angle - measured_angle;
rate_setpoint = angle_kp * angle_error; + angle_ki*integrate(angle_error)   // outer loop 

rate_error = rate_setpoint - gyro_rate;
motor_output = rate_kp * rate_error + rate_ki * integrate(rate_error) + rate_kd * derivative(rate_error); // inner loop
```

This worked somewhat.

### PID tuning
At this point, we didn't have much time before the competition, because always when we had tried to get it flying stably, something had broke. First we tried to tune it by just feeling the PID, but because of mass imbalance we had to add the integrator part to the PIDs. That can not be accurately felt, because it accumulates and then causes oscillation. Luckily Aleksi Huhtala designed and built a gimbal setup where we could let the drone "fly" freely and then watch the roll and pitch graphs as well as the actual drone:

### Gimbal test setup
<video controls width="500">
  <source src="{{ '/assets/videos/dronesatGimbalTest.mp4' | relative_url }}" type="video/mp4">
</video>

It's not very good in the video, but it was a lot worse before. We had to put the integrator very high. It still oscillated but at least it stayed quite near the center. We just had to hope it would be good enough for the flight.

### State machine
We had 6 states that could be changed automatically and manually:
- IDLE: do nothing except transmit lipo voltage, flags, gps fix and state
- PRELAUNCH: save ground pressure and launch site coordinates, transmit the same as in idle
- ROCKET: transmit primary mission (pressure, temperature), gps, an attitude and the same as in idle. Log data to flash
- FREEFLIGHT: transmit primary mission (pressure, temperature), gps, an attitude and the same as in idle. Log data to flash
- POWERED: transmit primary mission (pressure, temperature), gps, an attitude and the same as in idle. Ramp-up the motors and use the PID system to try to navigate back to the launch site. Log data to flash
- GROUND: transmit the same as idle
- FAULT: transmit the same as idle

We also had the following automatic state transitions:
- PRELAUNCH->ROCKET: if acceleration is above 2g for 50 milliseconds
- ROCKET->FREEFLIGHT: if LDR value is above 0.5 volts (light is detected)
- FREEFLIGHT->POWERED: if 2 seconds have elapsed
- POWERED->GROUND: if barometric altitude is less than 1 meter.

The code goes in fault mode, if the latest state was not rocket, freeflight or powered and there is data on the flash (possible flight data). If it was in those states, it will continue logging from where the program left off (reset proofing). Otherwise it will go to the idle state.

### Other flight code

Another team member, Nino Heino, also wrote GPS course over ground navigation system. There was also the afore mentioned velocity PID system. 

For the actual code, see:
- [The high level code](https://github.com/swadyx/kansatti2)
- [The low level code](https://github.com/oskarhuotari/Tonttulib)

### The flight

This is how the flight went:
![Test]({{ '/assets/images/dronesatFuckedUp.png' | relative_url }})

We had a mechanical failure (one of the motor arms detached) due to the ejection charge from the rocket. However, that's not even the most critical failure. Our flight computer was turned off during the whole descent phase. It was on in the rocket and it turned back on on the ground. We think the ejection charge caused a reset due to the Teensy's loose reset button. Our CanSat was however reset proofed and it's still a mystery, why it didn't turn back on before landing on the ground. I think the most likely explanation is that drag and spinning could have continously triggered the reset button during the freefall phase. That's why almost the only data we actually got from the flight is that the mud was 7.7 degrees Celcius.

### What I learned?
Here's what I learned from this whole project:
- Order three times as much parts as you think you're going to use. Then you can break one part and still have a backup.
- How to make a drone kind of fly
- Writing own libraries for sensors and designing a PCB
- There has to be a clear time schedule shared with everybody. Then everybody knows what they should do and when.

Here's what I learned from the actual flight:
- Remove loose buttons for high-G flights.
- Test dropping your electronics from 1 meter to a table and see if they reset. Fix the reset problem if discovered and don't trust that the electronics would work after a reset.
- Add a supercapacitor to your power delivery stack so that voltage drops can be compensated for. 

---
layout: post
title: "On drone CanSat"
date: 2026-03-30 12:00:00 +0200
categories: cansat PCB code
---

# DRONESAT

Our team set out to build a cansat that will fly back to the launch site. In the cansat competition, teams build a standard soda can shaped "satellite" that will be launched to an altitude of about 1 km. I was resposible on the PCB design and I already talked about it on the previous post. I also wrote some stablization code and tried to tune our PIDs.

Here's a picture of my PCB on the drone:
### Tonttuboard being beautiful
![Test]({{ '/assets/images/dronesatTonttuboard.png' | relative_url }})

The Tonttuboard had an IMU, barometer and a flash chip. It was designed to be as simple as possible so it was a carrier board for the Teensy 4.0. We wrote our code in the arduino IDE. We also had 4x BlHeli_s ESCs (electronic speed controllers) and 4x brushless 11000 kv motors. Tonttuboard talks to the esc:s through PMW signals (very fast on and off switching). In addition, we had 2x 7.4 v lipos. Our data was transmitted through ESP-Now and we had a XIAO Esp32s3 as a bridge between the Teensy and the groundstation ESP. I wasn't in charge of the mechanical design or the CAD modelling but here's a very cool initial model my friend made:
![Test]({{ '/assets/images/dronesatCAD.png' | relative_url }})

### Sensor fusion
We decided to go with a simple complementary filter. We also tried a kalman filter (without a magnetometer) but there was a lot of cross coupling between pitch and roll, which contributed to violent oscillations around the upright position. Unlike many other 6x sensor fusion filters, the complementary filter also has very little tuning. Here's how the angles can be obtained from the accelerometer:
```
    float rollFromAcc  = atan2f(ay, az);
    float pitchFromAcc = atan2f(-ax, sqrtf(ay * ay + az * az));
```
These return the angles in radians. Then by summing the gyro integration multiplied by some gain and the accelerometer estimates multiplied by one minus the same gain, more accurate estimates can be obtained:
```
roll  = 0.985f * (roll  + gx_rads * dt) + 0.015f * rollFromAcc;
pitch = 0.985f * (pitch + gy_rads * dt) + 0.015f * pitchFromAc;
```
This is due to the following facts:
- Accelerometers have noisy but drift free readings
- Gyros have driting but accurate readings 

In the drone, the sensor fusion was done 500 hz and the IMU used had a 500hz digital low pass filter.

### Motor mixing
Drone's have some way of knowing what they should do -- obviosly. For example they know that I'm tilted 5 degrees in the roll axis and thus I should put some motors spin faster and some slower. This is done in the motor mixing phase. There are many possible confiqurations, but the most commons ones are an X frame and a + frame. In a + frame, the motors are aligned with the axes and it's a tad simpler. That's why went with it. Yaw is applied to every motor, with a plus or minus sign depending on the motor's spin direction. Here's our motor mixing:
```
float m1 = baseThrust + rollOut  + yawOut;
float m2 = baseThrust + pitchOut - yawOut;
float m3 = baseThrust - rollOut  + yawOut;
float m4 = baseThrust - pitchOut - yawOut;
```
The base thrust can be controlled by a pilot or on our case it was controlled by a velocity PID system that tried to keep the descent velocity at 5 m/s (the slowest allowed in CanSat competitions).

### PID 
See for example [this video](https://www.youtube.com/watch?v=JBvnB0279-Q) (or just go read the Wikipedia if you're smart enough for that)

### The cascaded PID stabilization system
We had a world of troubles getting the drone's stabilization to work. First we tried an angle only PID system, but the drone did nothing and flipped or oscillated like crazy. I don't know if this was due to our 11000 kv motors, which could have made the IMU values noisy?

A rate-only PID system reacts to raw gyro values. It tries to oppose all movement but doesn't care about the drone's orientation. At some point I decided to try it out. Suddenly the drone was almost hovering and we could fix and then verify the motor mixing was correct. We could actually let go of the drone -- for 2 seconds but still. 

The rate PID system was the only thing that seemed to help so I had to keep it. We just added a angle PID system on top of it that outputs setpoints for the rate PID. It works like this:
```
angle_error = target_angle - measured_angle;
rate_setpoint = angle_kp * angle_error; + angle_ki*integrate(angle_error)   // outer loop 

rate_error = rate_setpoint - gyro_rate;
motor_output = rate_kp * rate_error + rate_ki * integrate(rate_error) + rate_kd * derivative(rate_error); // inner loop
```

This worked somewhat and we could let go of the drone for about 5 seconds. But the drone was exactly upright and it oscillated a lot.

### PID tuning
At this point, we didn't have much time before the competition, because always when we had tried to get it flying stably, something had broke. First we tried to tune it by just feeling the PID, but because of mass imbalance we needed the integrator part. That can not be felt because it accumulates and then causes oscillation. Luckily one of our team members designed and built a gimbal setup where we could let the drone fly and then watch the roll and pitch graphs.

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

The code goes in fault mode, if the latest state was not rocket, freeflight or powered and there is data on the flash (possible flight data). If it was in those states, it will continue logging from where the program left off (reset proofing). Otherwise it will go to idle.

### Other flight code

My friend also wrote GPS course over ground navigation system. There was also the afore mentioned velocity PID system. 

For the actual code, see:
- [The high level code](https://github.com/swadyx/kansatti2)
- [The low level code](https://github.com/oskarhuotari/Tonttulib)

### The flight

This is how the flight went:
![Test]({{ '/assets/images/dronesatFuckedUp.png' | relative_url }})

We had a mechanical failure (one of the motor arms detached) due to the ejection charge from the rocket. However, that's not even the most critical failure. Our flight computer was turned off during the whole descent phase. It was on in the rocket and it turned back on on the ground. We think the ejection charge caused a reset due to the Teensy's loose reset button. Our cansat was however reset proofed and it's still a mystery, why it didn't turn back on before landing on the ground. I think the most likely explanation is that drag and spinning could have continously triggered the reset button during the freefall phase. That's why almost the only data we actually got from the flight is that the mud was 7.7 degrees celcius.

### What I learned?
Here's what I learned from this whole project:
- Order three times as much parts as you think you're going to use. Then you can break one part and still have a backup.
- How to make a drone kind of fly
- Writing own libraries for sensors and designing a PCB
- Many big Finnish tech companies lack a vision. They promote working for a brighter future, but then can't find a few hundred euros to spare, when eager high-schoolers are building a cool thing and are ready to promote them. Don't get me wrong, I'm not entitled to anyone's money but I can't really think of any better ways to inspire future engineering students and making sure that Finland stays relevant in the tech scene. I'm only talking about big companies though, because I understand that start-ups don't have any "extra" money.

Here's what I learned from the actual flight:
- Remove loose buttons for high-G flights.
- Test dropping your electronics from 1 meter to a table and see if they reset. Fix the reset proble if discovered and don't trust that the electronics would work after a reset.
- Add a supercapacitor to your power delivery stack so that voltage drops can be compensated for. 

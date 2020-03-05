# ClassicPID

### Classical PID controller algorithm, explained in (correct and almost complete) (pseudo) code.

_Leonid Titov, 2019-2020_

Once again, a friend of mine asked if I can help with some simple but effective regulator code for a DIY project. Because people used to see perfectly straight lines on equipment regulated by my own algorithms, they keep asking me for a regulation advice. For simple projects you don't need complex algorithms, even trivial classic PID will suffice. And since people keep asking, I wrote this.

It's not a theory nor a math (for that see the links at the end), it's 100% down to earth, practical thing, which will help you get the job done.

_Useful if you need to implement it on your own, for some reason. Note that if you use a stock
regulator (either physical controller, or a "block" in the PLC program), that ALL OF THEM WORK WITH SUBTLE DIFFERENCES,
their implementors can't agree neither on terminology (proportional band vs. gain, integration time vs. reset time, etc),
nor what exactly is an integral, how and of what it should be calculated of._

_I don't say the classic PID is perfect, nor do I advocate its use; it's just a clarification of what it is, and is
suitable for most of general applications._

### "Wikipedia version"

From Wikipedia, https://en.wikipedia.org/wiki/PID_controller:

	previous_error := 0
	integral := 0
	
	loop:
		error := setpoint − measured_value
		integral := integral + error × dt
		derivative := (error − previous_error) / dt
		output := Kp × error + Ki × integral + Kd × derivative
		previous_error := error
		wait(dt)
		goto loop

The above is a math formula written in pseudo-code.

### My version:

This is very simplified implementation, it doesn't tackle the (very strongly advisable, almost mandatory)
A snd SP scaling to standard scale, and dt is fixed to 100 ms, but otherwise you can copy/paste it and it __will__
work "out of the box".

	A:	Measured Parameter
	SP:	Set Point
	Q:	Output, 0..100%

	EVERY 0.1 sec DO
	
		IF mode=Heater THEN
			E := SP - A;
		END_IF
		IF mode=Cooler THEN
			E := A - SP;
		END_IF
		
		integral := integral + E * k3;
		IF integral < 0 THEN integral := 0; END_IF
		IF integral > 100 THEN integral := 100; END_IF
		
		derivative := (E - previous_E) * k5;
		previous_E := E;
		
		Q := (Kp * E) + integral + derivative;
		
		IF Q < 0 THEN Q := 0; END_IF
		IF Q > 100 THEN Q := 100; END_IF
	END_EVERY

Note the difference in using the `integral` part. In Wikipedia version, it's increment/decrement goes without coefficient, and is unlimited (this is a huge error for real applications). Rather, later in the final summation, it is multiplied by coefficient. In my version, we use it as is in final summation, and __it is limited__ by the standard range, and a coefficient is applied at the increment/decrement stage. I am deeply convinced that my approach is more natural, as k3 naturally defines the _fastness_ of integral part changing, bound inside the standard range. My opinion is that it's more convenient and easy to
understand and simple in the code.

The k3 is basically a _direct_ equivalent of Ki in Wikipedia version. Look:

- increase Ki, and you faster get to the destination value;
- increase k3, and you faster get to the destination value;

The only difference, is that in the Wikipedia version, the increment is additionally scaled by dt. Thus, the absolute numbers may differ between k3 and Ki, but the general behavious is identical.

Similarly, k5 is the equivalent of Kd.

### You can think that k3==Ki, and k5==Kd.

Effects of changing PID parameters individually are:

|	|Momentary error	|Rise time	|Overshoot	|Settling time	|Steady-state error	|Stability	|
| ---	| ---	| ---	| ---	| ---	| ---	| ---	|
|Kp plus	|momentary correction effect	|slighly faster	|increase	|	|	|deteriorate	|
|Kp minus	|	|normal, doesn't affect	|decrease	|	|	|	|
|Ki or k3 plus	|	|faster	|increase	|increase	|__eliminate__	|deteriorate	|
|Ki or k3 minus	|	|slower	|decrease	|decrease	|	|increase	|
|Kd or k5 plus	|	|	|decrease	|decrease	|	|degrade	|
|Kd or k5 minus	|	|	|	|	|	|	|


Again, I didn't say this "Classic PID" is perfect algorithm at all. It just exists, so this is kind of
a clarification of what it is. Not a slightest bit of advocacy.

For the temperature regulation applications like heating an in-flow pasteurizer, these shortcut parameters apply,
if dt=0.1 sec:

| Parameter	| Typical value	| comment	|
| ---	| ---	| ---	|
|Kp:	|=0	|setting it to anything >0 for application with significant inertial delay just doesn't make sense. ("Intertial delay" meaning the delay between regulation action on a system, and a measured responce on that action from the system. It may be due to slow sensors, or slow IO, or placement of sensors relative to actuators, and other factors. For many systems with high intertial delay, setting Kp doesn't make sense.)	|
|Ki, k3:	|=0.003..0.030	|decrease if there are oscillations, increase for faster reaction.	|
|Kd, k5:	|=0+	|you can try to increase it slightly above 0 to arrest overshoots, but as well can leave it 0.	|

##### Note: some stock PID controllers won't work at all with zero Kp, for some reason, contrary to any logic or common sense.

For very slow and intertial systems, e.g. HVAC, the Ki/k3 must be decreased by two orders of magnitude:

	Ki, k3:	   = 0.00003 .. 0.00030

And setting Kd,k5 doesn't make sense then. Because this "classic PID" has the design flaw, in which "derivative part" doesn't
work as intended for very slow processes.

Hope this helps.

### Physical meaning, explanation and clarification

In the classical PID controller, each component, if we “chew” the physics of the process, is responsible for:

1. The proportional part is responsible for the instantaneous corrective response to the instantaneous and unexpected interference. While in a stabilized state, if the measured value A goes away unexpectedly, and we instantly respond to it, with a force proportional to the magnitude of the error.

    In 90% of general industrial applications, this component can be nullified, as I wrote in tables above, because it is of no use. The reason is that in 90% of general industrial applications, possible interference is incommensurable with a delay in the actuation output channel and in the measurement channel. For example, from practice, if you measure the flow and the actual flow “reaches” the program after 3-5 seconds (flowmeter speed), then it is impossible to respond to interferences of like 0.5 seconds length, it will only worsen the process, possibly buildup oscillations. Another example, if, due to the thermal inertia of the equipment, the program sees the actual temperature with a delay of 10 seconds, and the impact of the program on the process leads to its change with a delay of 20 seconds, then again, trying to cancel instant deviations of 1 second length is simply meaningless . It will only build up oscillations.

1. The differential part. The physics of this process is as follows. It doesn’t matter if the differential is there or what, but the meaning is that we are trying to predict in advance if the A parameter’s value reaches the set point SP (if we weren’t on it, for example, during the initial heating). On a graph this would correspond to drawing a tangent to a certain distance. And as soon as we detect predicted A reaching SP, we reduce the output, in an effort to reduce the overshoot of A over an SP.

    Except in the initial stage of engaging the SP level, also called "the end of ramp", this component is not needed anywhere else.

1. The integral component is perhaps the most important, although someone blathers the other way around.

    So, first - THIS IS NOT THE DEFINITE INTEGRAL of calculus, as defined in https://en.wikipedia.org/wiki/Integral. Everywhere all over the books, manuals, and Internet, it is written in formulas as a (some form of) integral, and is called "integral", but when implemented, no one, ever, implements it as definite integral. Why is that? Because the "language of calculus" conveniently offers this concept, while in a program it is very difficult to calculate it properly (you need a sliding buffer, etc.), and there's no need for that. Therefore, the "integral part" here isn't "the sum of errors for some period of time". Instead it's more like an __antiderivative__, but even that is not exact.

    Secondly, the physical meaning of this "integral" component is as follows. I will give a few examples.

    - Example 1. A swinging bar, such as a scale. There is a ball on it. Tilting it, you need to hold the ball exactly in the middle. The ball can be pushed with a finger, and the system must stabilize itself back. This example is one of the few where the proportional component is really needed, and it works, but the integral one is not necessary. Clear?

        Now, let's complicate the experiment a little bit: we will tie the ball with an elastic band so that it constantly pulls it in one direction (or we may blow it with a fan). Now, for it to stabilize in the middle, we need to constantly keep the bar tilted. Here, without an integral component, IT WILL NOT WORK. Why? Its "job" includes, if you want, a "zero offset" in the actuation output channel. But without this, the system cannot be stabilized. Is the physical meaning clear?

    - Example 2. The task is to add water to the tank so that the level is constant, while water constantly flows out from the tank. The initial state is an empty tank, the topping valve is closed.

        Without the integral component, only the proportional one: Since the error is big, we open the tap. Then there are two options. If the proportional coefficient is small, then the valve will open less than there is the outflow, and we will never reach the setpoint level. The second option, the coefficient is large, the level will increase, the error will disappear, and it may even succeed in changing the sign, in any case, the formula will immediately close the tap, the level will begin to fall. THIS IS AN OSCILLATION SYSTEM, which by its nature always sways. On average it can and will pour water, but the swings, as a rule, ON ALL TYPES OF EQUIPMENT, are limited in amplitude only by a breakdown of the equipment, or, in this case, by a periodic overflow and the "empty tank" states.

        Now, if there's an integral component present. In any case, whether the proportional component is large or zero at all, the integral part will gradually grow, and will “offset the zero” of the the valve, and the valve will gradually take such a position to accurately compensate for the water flowing out. The level, though not immediately, but evetually will stabilize.

    - Example 3. The same as 2, but with the heating of an object, which is constantly and simultaneously affected by some cooling. Also, the task of the integral component is to “offset the zero” in heating channel to compensate for simultaneous cooling.

In aviation, the “integral component” is called a “trim”. The term “trimmed plane” means a plane in which, when you release your arms and legs from the controls, it continues to fly straight. For example, being in manual mode, on any plane, deflecting the helm or side stick from yourself or towards yourself, you deviate the pitch of plane. If the plane "constantly climbs into the sky", for example, then you constantly need to keep your control "a little off yourself" to fly smoothly. In order not to keep it like this all the time, on any plane from small to large, it is possible to "trim" the elevators, or turn the wheel mechanically, or do it with electric motor switches, or somehow else. The trimming is the 100% equivalent of the “integral component” in the non-aviation field.

The design from Wikipedia is incorrect, because after a long period off set point, the "integral" part will leave the allowable range and will not return back in a reasonable time. In practice, it looks like equipment where the heating is turned on, and it does not heat up, the valve is open at 0%, up to half an hour sometimes... Helps to wait, or reboot. Apparently, they copied the code from Wikipedia. In my version, this drawback is carefully eliminated.

### Further links

Some links for further exploration, in an unlikely case you are not satisfied with classic algorithm:
  - http://ctms.engin.umich.edu/CTMS/index.php?example=Introduction&section=ControlPID
  - https://cbasso.pagesperso-orange.fr/Downloads/PPTs/Chris%20Basso%20APEC%20seminar%202012.pdf
  - https://en.wikipedia.org/wiki/Control_theory
  - https://en.wikipedia.org/wiki/PID_controller
  - https://de.wikipedia.org/wiki/Regler
  - https://en.wikipedia.org/wiki/Robust_control
  - https://en.wikipedia.org/wiki/Iso-damping
  - https://en.wikipedia.org/wiki/Active_disturbance_rejection_control
  - https://en.wikipedia.org/wiki/Root_locus
  - https://en.wikipedia.org/wiki/Transfer_function
  - https://en.wikipedia.org/wiki/Laplace_transform
  - https://en.wikipedia.org/wiki/Linear%E2%80%93quadratic_regulator
  

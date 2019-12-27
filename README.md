# ClassicPID

### Classical PID algorithm, explained in (correct and almost complete) (pseudo) code.

_Leonid Titov, 2019_

Useful if you need to implement it on your own, for some reason. Not useful if you use stock
regulator (either physical controller, or a "block" in the PLC program), because ALL OF THEM WORK DIFFERENTLY,
their implementors can't agree neither on terminology (proportional band vs. gain, integration time vs. reset time, etc),
nor what exactly is an integral, how and of what it should be calculated of.

_Disclaimer: I don't say the classic PID perfect, nor do I advocate to use it; it's just a clarification of what it is, and is
suitable for like 97% of applications. Again, this might help if you need to implement one on your own._

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

The above is incomplete to the degree of being almost incorrect. I.e. you can't just copy and paste and use it,
it won't work properly.

### My version #1:

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
		
		integral := integral + E;
		IF integral < 0 THEN integral := 0; END_IF
		IF integral > (100 / Ki) THEN integral := (100 / Ki); END_IF
		
		derivative := (E - previous_E);
		previous_E := E;
		
		Q := (Kp * E) + (Ki * integral) + (Kd * derivative);
		
		IF Q < 0 THEN Q := 0; END_IF
		IF Q > 100 THEN Q := 100; END_IF
	END_EVERY


### My version #2, functional equivalent of #1:

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
	
The k3 is basically the equivalent of Ki. My opinion is that it's more convenient and easy to
understand and simple in the code. Similarly, k5 is the equivalent of Kd. You can think that
k3==Ki, and k5==Kd.

Effects of changing parameters individually:

|	|Rise time	|Overshoot	|Settling time	|Steady-state error	|Stability	|
| ---	| ---	| ---	| ---	| ---	| ---	|
|Kp ++	|faster	|increase	|	|decrease	|degrade	|
|Kp --	|slower	|decrease	|	|	|	|
|Ki, k3 ++	|faster	|increase	|increase	|ELIMINATE	|degrade	|
|Ki, k3 --	|slower	|decrease	|decrease	|	|increase	|
|Kd, k5 ++	|	|DECREASE	|DECREASE	|	|degrade	|
|Kd, k5 --	|	|	|	|	|	|


Again, I didn't say this "Classic PID" is perfect algorithm at all. It just exists, so this is kind of
a clarification of what it is. Not a slightest bit of advocacy.

For 90% of slow temperature regulation applications, like heating a in-flow pasteurizer, these shortcut parameters apply,
if dt=0.1 sec:

| Parameter	| Typical value	| comment	|
| ---	| ---	| ---	|
|Kp:	|=0	|setting it to anything >0 for slow application with significant inertial delay just doesn't make sense.	|
|Ki, k3:	|=0.003..0.030	|decrease if there are oscillations, increase for faster reaction.	|
|Kd, k5:	|=0+	|you can try to increase it slightly above 0 to arrest overshoots, but as well can leave it 0.	|

For very slow and intertial systems, e.g. HVAC, the Ki,k3 must be decreased by two orders of magnitude:

	Ki, k3:	   = 0.00003 .. 0.00030

And setting Kd,k5 doesn't make sense then. Because this "classic PID" has the design flaw, in which "derivative part" doesn't
work as intended for very slow processes.

Hope this helps.

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
  

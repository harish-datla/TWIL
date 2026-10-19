# Continuous collision detection

Discrete collision detection tests shapes at their positions at the end of the timestep. A fast-moving object can pass entirely through a thin wall between two frames. This is the tunneling problem: the object was on one side at t, on the other side at t + dt, and the collision detector never saw an overlap.

Worked example

A 10 cm ball thrown at 30 m/s moves 0.5 m per 1/60 s step. Against a 20 cm wall, a discrete test catches it only if one of its sampled positions overlaps the wall, a window 0.2 + 0.1 = 0.3 m wide. A 0.5 m step can land on either side of that window, so whether the ball is caught depends on where its steps happen to fall.

CCD fixes this by computing the TOI (time of impact): the earliest time during the timestep at which two swept shapes first touch. The simulation is advanced to the TOI, the collision is resolved, and the remainder of the timestep continues.

Speculative contacts are a cheaper alternative. Instead of computing exact TOI, the solver adds a constraint for any pair that will be within contact distance at the end of the step (based on current velocity). The bias velocity is set to prevent penetration at the predicted position. Speculative contacts catch much of the tunneling that discrete detection misses, at a fraction of the cost of swept-volume CCD. Jolt creates speculative contacts within a small distance (2 cm by default) during its normal discrete collision pass, and runs a swept cast only for bodies whose motion quality is set to LinearCast[9].
Live · CCD tunneling demo
fire
reset
ball speed 800 units/s
Enable CCD
mode
CCD
result
...
A ball fired at a thin wall, with mode and result readouts. Uncheck CCD, raise the speed, and fire. The faint circles are the positions the collision test saw, one per 1/60 s step. The discrete test registers a hit only if one of those positions overlaps the wall, a window 20 units wide: 4 units of wall plus the ball's 16-unit diameter. Below 1,200 units/s a step is shorter than that window, so some position always lands in it and discrete detection catches the ball. Above it a single step can jump the whole window, and whether it does depends on where the steps happen to fall: from this start, 1,250, 1,500, 1,700, 1,900 and 1,950 units/s tunnel while the speeds between them are caught. With CCD on, the swept path is tested against the wall and every speed is caught.
Check Your Understanding Q1 of 1

A bullet in your game moves at 900 m/s. At 120 Hz physics, it travels 7.5 m per step. Walls are 0.2 m thick. Which approach prevents tunneling?
Make walls thicker (at least 7.5 m) so discrete detection catches the overlap.
Use CCD (swept-volume or speculative contacts) for the bullet body.
Increase the physics rate to 10,000 Hz so the bullet moves only 0.09 m per step.
0/1 correct

# Model Predictive Control Project

The objective of this project is to implement Model Predictive Control (MPC) to drive the car around the track.

---
## Table of contents

1. MPC Model Overview
2. Description of N and dt
3. Preprocessing Steps
3. Latency Effect

## 1. MPC Model Overview

We will first describe the state, actuator and update equations (ingredients in the model) and then go on to describe the MPC model setup and loop. We will then discuss tuning parameters.

#### State [x,y,$$\psi$$,v]
Our state can be described using four components:
1. x: the x-position
- y: the y-position
- ψ: the vehicle orientation
- v: the velocity

#### Actuators [δ, a]
* An actuator is a component that controls a system. Here we have two actuators: 
    * δ (the steering angle) and 
    * a (acceleration, i.e. throttle and brake pedals).

#### Update equations (Vehicle Dynamics)
* x = x + v\*cos(ψ)\* dt 
* y = y + v sin(psi) dt
* v=v+a∗dt
    * a in [-1,1]
* ψ=ψ+(v/L_f)*δ∗dt

#### MPC Setup:
1. Define the length of the trajectory, N, and duration of each timestep, dt.
    * See below for definitions of N, dt, and discussion on parameter tuning.
* Define vehicle dynamics and actuator limitations along with other constraints.
    * See the state, actuators and update equations above.
* Define the cost function.
    * Cost in this MPC increases with: (see `MPC.cpp` lines 80-101)
        * Difference from reference state (cross-track error, orientation and velocity)
        * Use of actuators (steering angle and acceleration)
        * Value gap between sequential actuators (change in steering angle and change in acceleration).
    * We take the deviations and square them to penalise over and under-shooting equally.
        * This may not be optimal.
    * Each factor mentioned above contributed to the cost in different proportions. We did this by multiplying the squared deviations by weights unique to each factor.

#### MPC Loop:
1. We **pass the current state** as the initial state to the model predictive controller.
* We call the optimization solver. Given the initial state, the solver will ***return the vector of control inputs that minimizes the cost function**. The solver we'll use is called Ipopt.
* We **apply the first control input to the vehicle**.
* Back to 1.

*Reference: Setup and Loop description taken from Udacity's Model Predictive Control lesson.*


#### N and dt
* N is the number of timesteps the model predicts ahead. As N increases, the model predicts further ahead.
* dt is the length of each timestep. As dt decreases, the model re-evaluates its actuators more frequently. This may give more accurate predictions, but will use more computational power. If we keep N constant, the time horizon over which we predict ahead also decreases as dt decreases.

#### Tuning N and dt
* I started with (N, dt) = (10, 0.1) (arbitrary starting point). The green panth would often curve to the right or left near the end, so I tried increasing N so the model would try to fit more of the upcoming path and would be penalised more if it curved off erratically after 10 steps.
* Increasing N to 15 improved the fit and made the vehicle drive smoother. Would increasing N further improve performance?
* Increasing N to 20 (dt = 0.1) made the vehicle weave more (drive less steadily) especially after the first turn. 
    * The weaving was exacerbated with N = 50 - the vehicle couldn't even stay on the track for five seconds. 
* Increasing dt to 0.2 (N = 10) made the vehicle too slow to respond to changes in lane curvature. E.g. when it reached the first turn, it only started steering left when it was nearly off the track. This delayed response is expected because it re-evaluates the model less frequently. 
* Decreasing dt to 0.05 made the vehicle drive in a really jerky way.
* So I chose N = 15, dt = 0.1.
* It would be better to test variations in N and dt more rigorously and test different combinations of N, dt and the contributions of e.g. cross-track error to cost. 
* It would also be good to discuss variations in N and dt without holding N or dt fixed at 10 and 0.1 respectively.

### Latency
* If we don't add latency, the car will be steering and accelerating/braking based on what the model thinks it should've done 100ms ago. The car will respond too slowly to changes in its position and thus changes in cost. 
	* It may not start steering e.g. left when it goes round a curve, leading it to veer off the track. 
	* Likewise, it may continue steering even when the path stops curving. 
	* The faster the vehicle speed, the worse the effects of latency that is unaccounted for.
* **Implementation**: We used the kinematic model to predict the state 100ms ahead of time and then feed that predicted state into the MPC solver.
	* We chose 100ms because that's the duration of the latency. That is, we try to predict where the car will be when our instructions reach the car so the steering angle and throttle will be appropriate for when our instructions reach the car (100ms later).
	* The code can be found in `main.cpp`.

### Other comments
* Strangely, when tackling the case where there is 100ms latency, predicting the state 100ms ahead gave a worse outcome than not predicting the state.

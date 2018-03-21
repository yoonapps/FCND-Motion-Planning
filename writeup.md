## Project: 3D Motion Planning
![Quad Image](./misc/enroute.png)

---


# Required Steps for a Passing Submission:
1. Load the 2.5D map in the colliders.csv file describing the environment.
2. Discretize the environment into a grid or graph representation.
3. Define the start and goal locations.
4. Perform a search using A* or other search algorithm.
5. Use a collinearity test or ray tracing method (like Bresenham) to remove unnecessary waypoints.
6. Return waypoints in local ECEF coordinates (format for `self.all_waypoints` is [N, E, altitude, heading], where the drone’s start location corresponds to [0, 0, 0, 0].
7. Write it up.
8. Congratulations!  Your Done!

## [Rubric](https://review.udacity.com/#!/rubrics/1534/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it! Below I describe how I addressed each rubric point and where in my code each point is handled.

### Explain the Starter Code

#### 1. Explain the functionality of what's provided in `motion_planning.py` and `planning_utils.py`
The starter code provides a very simple flight path of travelling 10 meters north and east, without diagonal movement and any pruning of the waypoints.

![Start Code](./misc/starter.png)

The `motion_planning.py` defines the `MotionPlanning` `Drone` subclass. It also defintes the `States` enum. When the drone gets initialized, it sets initial values for its properties, most importantly it sets the `self.flight_state` to `MANUAL`. Additionally, the class subscribes to several callbacks for local position, local velocity and state changes. These callbacks are crucial for executing the flight plan.

As the state is changed to `MANUAL`, the `state_callback` calls the `arming_transition` function, which arms the drone and sets the state to `ARMING`. Again, the state change callback fires. This time the `plan_path` function is called.

The `plan_path` function is where the grid, heuristic, and a star generated for the flight plan. The state at this time is `PLANNING`. The first target position is set, which is the drone's `TARGET_ALTITUDE`. `create_grid`, `heuristic`, and `a_star` are helper functions from `planning_utils.py`. `create_grid` generates a 2D grid with the value `1` marking obstacles that are higher than the safety clearance required for the drone's altitude. The heuristic is euclidean. Then the `a_star` function generates a path from start to goal based on the grid and the heuristic. The goal for the starter code is simply 10 meters north and east from the start position. Finally, this path results in the waypoints that the drone will fly to.

The state change callback will transition the drone to `TAKEOFF` and raise it to it's target altitude. As the drone flies up, the `local_position_callback` is called. Once it is 95% at it's target position, which is the target altitude, the `waypoint_transition` function is called.

Now the drone is in the `WAYPOINT` state. The first waypoint in the list is accessed and also removed from the waypoint list. The waypoint is set as the target and the drone is commanded to fly there. The `local_position_callback` is called back as the drone moves, and when it is within 1 cubic meter of it's target waypoint, it executes the next waypoint if there is one. As the `waypoint_transition` pops through the waypoints and the drone travels to it's last waypoint, the drone will slow down. This drop in velocity triggers the `landing_transition` callback. 

The `landing_transition` commands the drone to land. As the drone lands, the `velocity_callback` is continuously called. When the difference between the altitudes of `global_position` and `global_home` are less than 0.1 meters, and the altitude of the `local_position` is less than 0.01 meters, the `disarming_transition` function is called. The drone is disarmed, released of control, state becomes `DISARMING`, and which in turn, the state callback will call the  `manual_transition` which terminates the mission.

### Implementing Your Path Planning Algorithm

#### 1. Set your global home position
The global home position is set by reading the first line of data from the `colliders.csv` file. The `np.genfromtxt` has a parameter `max_rows` which makes it easy to grab just the first line of the file. A comma delimiter is used so the `split()` function will easily break both the latitude and longitude apart from their labels. These values are casted as `float`s and passed into `set_home_position`.

#### 2. Set your current local position
Converting the current position from global to local is done simply by passing the `global_position` and `global_home` into the `global_to_local` function.

#### 3. Set grid start position from local position
In order to set the starting grid position from the local position rather than the map center, the current local position obtained above must be offset by the minimum north and east values returned from the `create_grid`. This is because the southwest corner is the origin of the grid.

#### 4. Set grid goal position from geodetic coords
The grid goal position is obtained in a similar manner to the grid start position. The arbitrary global goal position is converted to local by using `global_to_local`. The resulting local coordinates are then offset by the minimum north and east values from the grid.

#### 5. Modify A* to include diagonal motion (or replace A* altogether)
Enabling diagonal movement is done by modifying the `Action` enum class. Four additional cases were added: 
	```
	NORTHEAST = (-1, 1, np.sqrt(2))
    NORTHWEST = (-1, -1, np.sqrt(2))
    SOUTHEAST = (1, 1, np.sqrt(2))
    SOUTHWEST = (1, -1, np.sqrt(2))
    ```
Each case has the appropriate deltas corresponding with the diagonal direction as well as the cost for the action.

Additionally, the `valid_actions` requires 4 more checks on whether the grid space in relation to the action has an obstacle. Each check remove the related diagonal action from the list of valid actions.

By modifying the `Action` enum class, the A* implementation works as is.

#### 6. Cull waypoints 
The pruning method used for this project is the collinearity check. The path found by the A* algorithm is iterated with a `while` loop. 3 path values are checked each iteration. The determinant of the three vectors are calculated. If the determinant is zero (or almost zero, in this case less than 0.000001), then the three points are deemed in a single line. This removes the path coordinate from the list of paths.


### Execute the flight
#### 1. Does it work?
It works!

### Double check that you've met specifications for each of the [rubric](https://review.udacity.com/#!/rubrics/1534/view) points.
  
# Extra Challenges: Real World Planning

For an extra challenge, consider implementing some of the techniques described in the "Real World Planning" lesson. You could try implementing a vehicle model to take dynamic constraints into account, or implement a replanning method to invoke if you get off course or encounter unexpected obstacles.



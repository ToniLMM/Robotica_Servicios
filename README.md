# Blog-Robotica-Servicios
This will be Toni's Services Robotics blog, where the status of the practices, their news, progress, errors and solutions will be reported.

# Index

* [Index][Ind]
* [Practice 1][p1]
* [Practice 2][p2]
* [Practice 3][p3]
* [Practice 4][p4]
* [Practice 5][p5]
* [Practice 6][p6]

[Ind]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#index
[p1]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#practice-1-localized-vacuum-cleaner
[p2]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#practice-2-rescue-people
[p3]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#practice-3-autoparking
[p4]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#practice-4-amazon-warehouse
[p5]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#practice-5-laser-mapping
[p6]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#practice-6-marker-based-visual-loc

## Practice 1: Localized Vacuum Cleaner

### Goal

The objective of this exercise is to implement the logic of a navigation algorithm for an autonomous vacuum cleaner by making use of the location of the robot. The robot is equipped with a map and knows it’s current location in it. The main objective will be to cover the largest area of ​​a house using the programmed algorithm


## Practice 2: Rescue People

### Goal

The goal of this exercise is to implement the logic that allows a quadrotor to recognize the faces of lost people and save their locations in order to perform a subsequent rescue maneuver.

### Get the coordinates

#### 1º - Converting Degrees–Minutes–Seconds to Decimal Degrees
Before the drone can navigate autonomously, we need to translate geographic coordinates (latitude and longitude) into local metric coordinates (X, Y) that the control system can use for movement. This conversion process involves three main steps:

The original GPS positions were given in the DMS (Degrees, Minutes, Seconds) format:

- Safety boat: 40º16’48.2” N, 3º49’03.5” W
- Survivors: 40º16’47.23” N, 3º49’01.78” W

To simplify calculations, these were converted into decimal degrees using an online converter:
Montana State Converter: https://rcn.montana.edu/Resources/Converter.aspx

The result was:
```py
SAFETY_BOAT_GPS = (40.280055555555556, -3.817638888888889)
SURVIVORS_GPS = (40.27978611111111, -3.817161111111111)
```
#### 2º - Understanding UTM Coordinates

The UTM (Universal Transverse Mercator) system divides the Earth into 60 longitudinal zones, each covering 6° of longitude. Instead of degrees, UTM uses meters as units.

For each point, UTM provides:

- Easting (X): Distance in meters from the zone’s central meridian.
- Northing (Y): Distance in meters from the Equator.

In our case (Madrid area), both points are located in Zone 30T of the UTM grid.
Using the theory from MapTools’ UTM Guide (https://www.maptools.com/tutorials/utm/quick_guide), we can understand that small changes in latitude and longitude correspond to roughly constant meter distances:

- 1° of latitude ≈ 111,320 m
- 1° of longitude ≈ 111,320 × cos(latitude)

#### 3º - Converting to Local Coordinates

Since the drone’s control system operates in a local frame centered on the takeoff point, we compute the relative displacement between the two positions in meters:

```py
lat_diff = (target_lat - reference_lat) * 111320
lon_diff = (target_lon - reference_lon) * 111320 * cos(reference_lat)
x = -lat_diff
y = -lon_diff
```

- The X-axis represents the North–South direction.
- The Y-axis represents the East–West direction.
- The negative signs ensure the correct orientation according to the simulation’s coordinate system.

### Face detection

During the search mission, the drone uses two onboard cameras — a frontal camera and a ventral camera.

- The frontal camera provides a general view of the environment for monitoring through the GUI.
- The ventral camera is mainly used for automatic face detection while the drone follows its search pattern.

Face detection is implemented using OpenCV’s Haar Cascade classifier, a machine-learning model trained to recognize human faces.
Each frame from the ventral camera is converted to grayscale and processed by the cascade detector:

```py
faces = face_cascade.detectMultiScale(gray, scaleFactor=1.1, minNeighbors=4,
                                      minSize=(20, 20), maxSize=(100, 100))
```

Whenever one or more faces are detected, the system:

1. Draws a blue rectangle around each face in the image.

2. Checks if the detection corresponds to a new survivor by comparing the current drone position with previously recorded detections.

3. If it’s a new person, the current drone coordinates (X, Y, Z) are stored in a list called found_faces.

This way, the drone builds a record of all detected survivors along its path, avoiding duplicate entries and enabling a summary of their estimated positions once the mission ends.

### Final version

<img width="186" height="618" alt="image" src="https://github.com/user-attachments/assets/9230907f-5108-4faa-85c4-094b7a313138" /> <img width="209" height="422" alt="image" src="https://github.com/user-attachments/assets/372c3fcc-633a-4714-9969-b7613d24bcd4" />
![WhatsApp Image 2025-10-26 at 19 51 30](https://github.com/user-attachments/assets/f669e148-6229-433d-8521-10ba87a19263)

This version is a state machine where I use 5 states:

- TAKEOFF: The drone takes off vertically until it reaches a predefined search altitude. Once the target height is reached, it transitions to the NAVIGATE state.
- NAVIGATE: The drone uses the GPS coordinates of the safety boat and the survivor area to compute a local reference frame. It flies toward the search zone following the calculated vector and orientation. When the drone gets close enough to the search point, it transitions to the SEARCH state.
- SEARCH: The drone performs a lawnmower search pattern at constant altitude to scan the area efficiently. The search pattern is divided into 3 phases:
  - Phase 0: Move along the positive X direction.
  - Phase 1: Move along the positive Y direction (incrementally increasing the target each time).
  - Phase 2: Move along the negative X direction.
  
  The pattern alternates between these phases to cover the whole region. At each step, the ventral camera is used to detect faces using a Haar Cascade classifier, while the frontal camera provides real-time visualization. Every time a new survivor is detected, their position is recorded. Once all survivors are found or the full area is covered, the drone transitions to RETURN.
- RETURN: The drone computes a direct path back to its takeoff point. It maintains a slightly higher altitude during the return flight for safety. When it gets within 1 meter of the starting position, it transitions to the LAND state.
- LAND: The drone performs a controlled descent until it reaches a safe altitude. Then it executes an automatic landing using the HAL.land() command.

### Difficulties

Once I had completed the entire route, there were many faces that the drone didn't recognize. Talking to people in class who had the same problem, they told me that one solution could be to rotate the image at different angles to better detect faces. In this way I rotated the image 360º divided into 45º turns resulting in 8 turns per iteration:

```py
rotation_angles = [0, 45, -45, 90, -90, 135, -135, 180]
```

### Final video

The video is accelerated x4 the original duration is 13.07 minutes:


https://github.com/user-attachments/assets/223ec903-b07e-4466-a8e9-b2ef89bb3f58


## Practice 3: Autoparking

### Goal

The objective of this exercise is to implement the logic of a navigation algorithm for an automated vehicle. The vehicle must find a parking space and park properly

### First approach

During the first days I was trying so that the car would get as close as possible to the parked cars in order to maneuver better once a space was found.

<img width="1120" height="586" alt="Screenshot from 2025-11-04 17-57-06" src="https://github.com/user-attachments/assets/977a5c89-a169-45d6-8e65-00e208097396" />


### Laser data

This practice is based on the interpretation of laser data. That's why we have 3 lasers: one front, one rear, and one side (right).

Rear and front lasers are used for the same, avoid colliding with obstacles. They both scan all laser readings.

On the other hand right side laser is used to identify suitable parking spaces. This function analyzes consecutive laser readings to find gaps between obstacles and calculates estimated gap width using trigonometric formulas
```py
def detect_parking_gap(laser_data):
    # [Implementation details...]
    if (estimated_width >= MIN_GAP_WIDTH and 
        gap_depth >= MIN_GAP_DEPTH and 
        max_gap_width >= 50):
        return True, estimated_width, gap_depth
```


### Final algorithm

This is a pretty accurate drawing about the functioning of my algorithm (I forgot to draw the left turn when the approaching to the cars is completed)

![WhatsApp Image 2025-11-07 at 18 40 49](https://github.com/user-attachments/assets/d5c93547-dc0a-49e1-b62a-4849f7c4c183)

My final algorithm is composed of 9 states:
```
STATE_MOVE_RIGHT = 0
STATE_TURN_LEFT = 1
STATE_SEARCH_GAP = 2
STATE_ADVANCE_TO_PARK = 3
STATE_BACKWARD_RIGHT = 4
STATE_BACKWARD_STRAIGHT = 5
STATE_BACKWARD_LEFT = 6
STATE_PARKING = 7
STATE_PARKED = 8
```

1. The process begins with STATE_MOVE_RIGHT, where the vehicle approaches the parked cars while moving forward and turning right. During this phase, the system continuously monitors front obstacle distances, and when an obstacle gets too close, it transitions to the next state to begin the proper alignment for parking.

2. Once the vehicle detects a frontal obstacle, it enters STATE_TURN_LEFT to properly orient itself parallel to the parking spaces. This careful angular adjustment ensures the vehicle is perfectly aligned for the parking space search phase that follows.

3. The STATE_SEARCH_GAP represents the critical detection phase where the system scans for suitable parking spaces using the right-side laser sensor. The vehicle moves forward slowly while the algorithm analyzes distance patterns to identify gaps between vehicles that meet the minimum requirements of 2.5 meters width and 4 meters depth.

4. After identifying a suitable parking spot, the vehicle enters STATE_ADVANCE_TO_PARK, positioning itself optimally relative to the target space. The system advances forward until the rear obstacle distance reaches the optimal threshold, ensuring the vehicle is perfectly positioned to begin the reverse parking maneuver. This careful positioning is crucial for executing the subsequent parking steps accurately.

5. The actual parking maneuver consists of three coordinated reverse states. First, STATE_BACKWARD_RIGHT initiates the reverse parking by moving backward while turning right, adjusting the vehicle's angle. This creates the initial entry angle into the parking space.

6. Then STATE_BACKWARD_STRAIGHT continues the reverse motion in a straight line for exactly 3 meters, carefully monitoring the distance traveled using position tracking.

7. The final reverse phase, STATE_BACKWARD_LEFT, completes the parking alignment by moving backward while turning left to straighten the vehicle within the parking space. This state monitors both the vehicle's orientation and rear obstacle distance. If the system detects an obstacle too close behind (less than 0.5 meters), it triggers a safety maneuver to adjust position, otherwise it continues until achieving the target orientation.

8. The system concludes with STATE_PARKING, which performs fine adjustments by moving forward and turning right to achieve perfect alignment within the parking space. This final adjustment ensures the vehicle is centered and properly oriented. Finally, STATE_PARKED represents the completion state where all systems are stopped and the parking maneuver is successfully completed

### Final video

Video between 2 cars, the video is accelerated x2:

https://github.com/user-attachments/assets/aa511a0a-e0a2-4d1d-9ef1-76c5cd8e028c

Video without a front car, the video is accelerated x2:

https://github.com/user-attachments/assets/7f6e17c2-04a3-4012-87e4-0eb200ed1a05

This video was recorded after I sent the code un Aula virtual, so I did some changes:

In STATE_ADVANCE_TO_PARK if the back_dist is over 9.5m the car changes state to STATE_BACKWARD_RIGHT to start the parking maneuver:
```
elif current_state == STATE_ADVANCE_TO_PARK:
        print(f"ACERCANDOSE A COCHE DELANTERO: Distancia trasera: {back_dist:.2f}m")
        
        if back_dist > 9.5:
            change_state(STATE_BACKWARD_RIGHT, current_yaw_deg)
        if back_dist > BACK_OBSTACLE_DISTANCE:
            HAL.setV(LINEAR_SPEED)
            HAL.setW(0.0)
        else:
            change_state(STATE_BACKWARD_RIGHT, current_yaw_deg)
```

I added some details in detect_parking_gap(). When there is no car in front, the gap continues indefinitely. This modified code detects these "open-ended" gaps at the end of the sensor's range, while the Aula Virtual code only detects gaps that are "closed" by obstacles:

```
def detect_parking_gap(laser_data):
    readings = laser_data.values
    
    gap_start = None
    max_gap_width = 0
    gap_depth = 0
    best_gap_depth = 0
    
    for i, distance in enumerate(readings):
        if laser_data.minRange < distance < laser_data.maxRange:
            if distance > LASER_DISTANCE_THRESHOLD:
                if gap_start is None:
                    gap_start = i
                    gap_depth = distance
                
                if distance > gap_depth:
                    gap_depth = distance
            else:
                if gap_start is not None:
                    gap_end = i
                    gap_width = gap_end - gap_start
                    
                    if gap_width > max_gap_width:
                        max_gap_width = gap_width
                        best_gap_depth = gap_depth
                    
                    gap_start = None
                    gap_depth = 0
    
    if gap_start is not None:
        gap_end = len(readings) - 1
        gap_width = gap_end - gap_start
        
        if gap_width > max_gap_width:
            max_gap_width = gap_width
            best_gap_depth = gap_depth
    
    if best_gap_depth > 0:
        gap_depth = best_gap_depth
    
    gap_angle_width = max_gap_width * (180.0 / len(readings))
    estimated_width = gap_depth * math.tan(math.radians(gap_angle_width / 2)) * 2
    
    if (estimated_width >= MIN_GAP_WIDTH and 
        gap_depth >= MIN_GAP_DEPTH and 
        max_gap_width >= 50):
        return True, estimated_width, gap_depth
    
    return False, estimated_width, gap_depth
```

At last BACK_OBSTACLE_DISTANCE must be changed to 2.2:
```
BACK_OBSTACLE_DISTANCE = 2.2
```

With this changes both cases works perfectly.

Final result:

<img width="1114" height="292" alt="image" src="https://github.com/user-attachments/assets/d870d655-6d50-4934-8ee7-132480596605" />

### Difficulties

At first, when I only entered linear speed, the car seemed to veer off course, however, I then realized that the road wasn't completely straight. This way my turns aren't entirely precise and the parking is done with angles that aren't completely accurate. For example, at the end of the video you can see that my car ends up at -89.3º when the straight car should be at -94 (approximately). However, in practical terms, this hardly affects the functioning or the human eye.

## Practice 4: Amazon Warehouse

### Goal

The objective of this exercise is to implement the logic that allows a logistics robot to deliver shelves to the required place by making use of the location of the robot. The robot is equipped with a map and knows its current location in it. The main objective will be to find the shortest path to complete the task.

### Werehouse 1

These are the measures and the axis of the map:

![WhatsApp Image 2025-11-24 at 00 06 26](https://github.com/user-attachments/assets/273acd7a-bdad-4e1f-a64e-af80e57b1164)

There are 6 shelves and these are their coordinates: 

1. (3.728, 0.579) 
2. (3.728, -1.242)
3. (3.728, -3.039) 
4. (3.728, -4.827)
5. (3.728, -6.781) 
6. (3.728, -8.665)

The shelf dimensions are 2x0.8 meters:

<img width="1110" height="429" alt="image" src="https://github.com/user-attachments/assets/6944ae99-0ae3-482b-a1d9-c91665646b46" />

The simulator uses world coordinates (meters), while the map image uses pixel coordinates.  
To connect both systems, an affine transform is computed using 'cv2.getAffineTransform'.

This transform allows:
- converting robot poses from world to pixel for collision checking,
- converting planned pixel paths to world coordinates for navigation,
- drawing the planned route on the map.

This transformation is essential because OMPL operates in world space, while obstacle checking is done in pixel space.

### OMPL

OMPL is a powerful library for motion planning that provides algorithms to find paths for robots while avoiding obstacles. It supports different types of planners, from sampling-based algorithms like RRT (Rapidly-exploring Random Tree) and RRT* to control-based planners suitable for non-holonomic robots, such as car-like robots. OMPL operates over a State Space, where each state represents a possible configuration of the robot (position, orientation, etc.), and uses a State Validity Checker to ensure that states and transitions are collision-free.


#### OMPL Parameters 

| Parameter | Code | Description | Typical Value / Notes |
|-----------|------|-------------|---------------------|
| **Turning Radius** | `space = ob.ReedsSheppStateSpace(turning_radius)` | Minimum turning radius of the robot. Ensures paths respect the robot's kinematic limits. | `0.5 m` |
| **State Bounds** | `bounds.setLow(0, -warehouse_width/2); bounds.setHigh(0, warehouse_width/2)` | Defines the allowed `(x, y)` planning area, keeping the robot inside the warehouse. | Warehouse size: `20.62 x 13.6 m` |
| **State Validity Checking Resolution** | `si.setStateValidityCheckingResolution(0.01)` | Fraction of each path segment checked for collisions. Smaller values = more precise checks, slower planning. | `0.01` (1%) |
| **Planner Choice** | `planner = og.RRTstar(si)` | Sampling-based planner that finds feasible paths and optimizes them over time. | `RRT*` |
| **Planner Range** | `planner.setRange(10)` | Maximum distance a new node can extend from an existing node. | `10` meters (adjust for map scale) |
| **Goal Bias** | `planner.setGoalBias(0.8)` | Probability of sampling directly towards the goal, speeding up convergence. | `0.8` (80%) |
| **Goal Threshold** | `goal_region.setThreshold(0.12)` | Distance tolerance for reaching the goal. | `0.12 m` |
| **Path Interpolation** | `solution_path.interpolate(10)` | Adds intermediate waypoints for smoother robot motion. | `10` points between waypoints |
| **Custom Exploration Sampler** | `exploration_sampler = ExplorationStateSampler(space, rrt_explored)` | Records all sampled states for visualization and debugging. | N/A |

These parameters ensure that the robot can generate collision-free and optimized paths in the warehouse environment.

### isStateValid

The function 'isStateValid(state)' is responsible for determining whether a robot configuration (x, y, yaw) is collision-free. OMPL calls this function repeatedly during planning to ensure every sampled state and motion segment is safe.

In this code, isStateValid performs the following checks:

1. **Convert robot pose to map coordinates**  
   The robot's world position is transformed into pixel coordinates using the affine transform created earlier.

2. **Compute the robot’s footprint**  
   Based on the robot’s size (normal or carrying a shelf), the algorithm builds a rectangular polygon representing the robot in the map.  
   This polygon is rotated according to the robot’s yaw.

3. **Fill the polygon on a temporary mask**  
   A binary mask is created with cv2.fillPoly, marking all pixels the robot would occupy in that state.

4. **Collision test with obstacles**  
   The mask is compared with obst_bool, the obstacle map.  
   If any pixel of the robot overlaps with an obstacle, the state is invalid.

5. **Return result**  
   - True → the state is safe and OMPL can use it.  
   - False → the state is rejected and the planner must sample another one.

This validity checker is crucial because OMPL only produces paths composed of states that pass this function. In other words, no path segment is accepted unless every pose along it is collision-free.


### Navigation 

The navigation logic is implemented as a state machine that guides the robot through planning, movement, lifting the shelf, and returning to the start. It is divided into well-defined phases to keep the behavior predictable and modular.

#### 1. WAITING_PLAN
The robot waits for its initial pose and then calls the OMPL planner to generate a path to the target shelf.  
- Computes the path using Reeds–Shepp planning.  
- Visualizes the explored nodes and planned route.

#### 2. NAVIGATING
The robot follows the generated waypoints sequentially.  
Navigation is handled by improved_navigation(), which has two internal modes:
- **TURN**: Rotate until the robot faces the waypoint.
- **FWD**: Move forward while applying small angular corrections.  

Once the robot reaches a waypoint, it switches back to TURN for the next one.

#### 3. LIFTING
When the robot reaches the shelf:
- It lifts the shelf using HAL.lift().
- Robot dimensions are updated (wider + longer).
- The state validity checker is rebuilt to use the new footprint.

#### 4. WAITING_RETURN_PLAN
A new OMPL plan is computed to return to the origin (0,0), now with the shelf attached.

#### 5. RETURNING
The same navigation logic is used to follow the return path.  
The target is the origin, visualized with a separate marker.

#### 6. COMPLETED
The robot stops, puts the shelf down, and the process ends.

This structure allows the robot to perform a full warehouse pick-up and return cycle robustly and safely.

### Final Videos

#### Holonomic robot widened obstacles

Shelf 1:

[Screencast from 2025-11-24 22-53-15.webm](https://github.com/user-attachments/assets/c16481be-4461-4b2d-8a07-7655fbe66125)

Shelf 3:

https://github.com/user-attachments/assets/517a39e2-066a-45e7-9fa7-64bce05d7cfc

Shelf 5:

https://github.com/user-attachments/assets/2ab80d14-0f04-45d9-82ce-c9bb64240089


#### Holonomic robot only geometric constraints

Shelf 2:

https://github.com/user-attachments/assets/9b2afc79-d5b6-40b0-a898-7f11ac5b2546

Shelf 3:

[Screencast from 2025-11-25 12-24-38.webm](https://github.com/user-attachments/assets/e848a421-6680-4677-b55e-4d31f57c99e7)


## Practice 5: Laser Mapping

### Goal

The goal of this exercise is to develop a navigation algorithm that allows a robot to autonomously explore a warehouse environment while generating an accurate map of the area using LIDAR sensor data.

### Probabilistic Occupancy Map & Bayesian Updates

The robot builds its environment map using a probabilistic occupancy grid, where each cell holds a probability between 0 and 1. A value of 0 indicates the cell is free, 1 means occupied, and 0.5 represents unknown space.

The mapping process starts with a completely unknown grid where all cells are set to 0.5. As the robot moves, it uses its LiDAR sensor to scan the surroundings. For every laser ray, the cells along the path are likely free, while the endpoint of the ray, if an obstacle is detected, is likely occupied. The likelihoods are represented with fixed probabilities: P_FREE = 0.3 and P_OCCUPIED = 0.9. These probabilities are not absolute but serve as evidence to incrementally update the map.

To combine prior knowledge with new sensor readings, the robot uses Bayes’ theorem. Each cell’s probability is updated according to the formula:

**Posterior = (Measurement × Prior) / (Measurement × Prior + (1 - Measurement) × (1 - Prior))**

This ensures that multiple readings gradually adjust the cell’s probability. If a cell is consistently seen as free, its probability decreases toward 0, and if it is repeatedly detected as occupied, it moves toward 1. In cases of conflicting readings, the probabilities adapt smoothly rather than flipping abruptly, making the map robust to sensor noise.

Finally, the probability grid is converted into a grayscale map for visualization. Cells with high occupancy probability become black, free cells become white, and uncertain cells appear gray. This results in a clear and continuously improving representation of the robot’s environment that guides navigation and decision-making.

### NAVIGATION

The robot uses a finite state machine (FSM), this design keeps the navigation simple, robust, and easy to expand. These are the 5 states used:

**INITIAL_TURN**: The robot performs an initial 90° rotation to align itself with a consistent global direction. This ensures that all future movements follow a stable reference orientation.

**FORWARD**: In this main exploration state, the robot drives straight ahead while monitoring obstacles with the LiDAR. If something is detected within a safety distance, the robot immediately stops and prepares to turn.

**TURN_LEFT / TURN_RIGHT**: These states rotate the robot by a fixed angle when an obstacle blocks the path or when the exploration pattern requires a directional shift. A proportional controller ensures accurate rotation before moving on.

**FORWARD_1M**: The robot advances exactly one meter using odometry feedback. This helps enforce a structured, grid-like exploration pattern. If an obstacle appears during this movement, the robot interrupts the motion and transitions into a turn.

### Final Viedos


Without noise (getPose3d()):

https://github.com/user-attachments/assets/5e83a5b3-69af-4216-9bb1-e617abeea652

With some noise (HAL.getOdom2()):

https://github.com/user-attachments/assets/58e66fc2-edd0-4af8-beee-57704e04277c

With much more noise (HAL.getOdom3()):

https://github.com/user-attachments/assets/5fe1aff4-b7da-443e-8758-6cc087c3975e

#### Final Result

Without noise (getPose3d()):

<img width="1114" height="312" alt="Screenshot from 2025-12-08 20-03-23" src="https://github.com/user-attachments/assets/29bf84a7-4db2-434a-a57b-04937424d01b" />

With some noise (HAL.getOdom2()):

<img width="1113" height="264" alt="image" src="https://github.com/user-attachments/assets/dacdb63a-2d38-4013-a210-904c8e2e54ab" />

With much more noise (HAL.getOdom3()):

<img width="1352" height="353" alt="image" src="https://github.com/user-attachments/assets/1fd02965-a640-438e-84f7-ae41dfd8b295" />


## Practice 6: Marker Based Visual Loc

### Goal

The goal of this exercise is to estimate the position and orientation (pose) of a robot in a 2D space by detecting and analyzing visual markers, specifically AprilTags. This process involves using computer vision to identify the tags in the robot’s environment and mathematical methods to derive its relative pose to the detected tags.

The green robot represents the real position. The blue robot represents the position from the odometry (with noise). The red robot represents the user estimated position.

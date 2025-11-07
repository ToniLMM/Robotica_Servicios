# Blog-Robotica-Servicios
This will be Toni's Services Robotics blog, where the status of the practices, their news, progress, errors and solutions will be reported.

# Index

* [Index][Ind]
* [Practice 1][p1]
* [Practice 2][p2]
* [Practice 3][p3]

[Ind]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#index
[p1]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#practice-1-localized-vacuum-cleaner
[p2]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#practice-2-rescue-people
[p3]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#practice-3-autoparking

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


Final result:

<img width="1114" height="292" alt="image" src="https://github.com/user-attachments/assets/d870d655-6d50-4934-8ee7-132480596605" />

### Difficulties

At first, when I only entered linear speed, the car seemed to veer off course, however, I then realized that the road wasn't completely straight. This way my turns aren't entirely precise and the parking is done with angles that aren't completely accurate. For example, at the end of the video you can see that my car ends up at -89.3º when the straight car should be at -94 (approximately). However, in practical terms, this hardly affects the functioning or the human eye.


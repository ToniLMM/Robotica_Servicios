# Blog-Robotica-Servicios
This will be Toni's Services Robotics blog, where the status of the practices, their news, progress, errors and solutions will be reported.

# Index

* [Index][Ind]
* [Practice 1][p1]
* [Practice 2][p2]

[Ind]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#index
[p1]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#practice-1-localized-vacuum-cleaner
[p2]: https://github.com/ToniLMM/Robotica_Servicios/blob/main/README.md#practice-2-rescue-people

## Practice 1: Localized Vacuum Cleaner

### Goal

The objective of this exercise is to implement the logic of a navigation algorithm for an autonomous vacuum cleaner by making use of the location of the robot. The robot is equipped with a map and knows it’s current location in it. The main objective will be to cover the largest area of ​​a house using the programmed algorithm


## Practice 2: Rescue People

### Goal

The goal of this exercise is to implement the logic that allows a quadrotor to recognize the faces of lost people and save their locations in order to perform a subsequent rescue maneuver.

### Get the coordinates

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

### Final video

The video is accelerated x4 the original duration is 13.07 minutes:


https://github.com/user-attachments/assets/223ec903-b07e-4466-a8e9-b2ef89bb3f58


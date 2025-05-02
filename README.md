# Multi-Map Navigation with Wormholes – ANSCER Robotics Assignment

## Project Overview

This ROS project implements a multi-map navigation system using TurtleBot3, where the robot:
- Navigates within and across separately mapped rooms
- Transitions through overlapping wormhole regions between maps
- Uses a custom action server to receive goals and determine how to move based on the target map

---

##Features

- Multi-map navigation via wormholes
- SQLite-based database for wormhole lookups
- Action server using `NavigateToMap.action`
- `move_base` integration for autonomous navigation
- OOP-based modular design with `NavigationServer` class
- Supports 3 mapped rooms: `room1`, `room2`, `room3`

---

## Folder Structure

```
catkin_ws/
├── build/
├── devel/
├── src/
│   ├── CMakeLists.txt
│   └── anscer_nav/
│       ├── action/
│       │   └── NavigateToMap.action
│       ├── CMakeLists.txt
│       ├── coordinates.yaml
│       ├── include/
│       │   └── anscer_nav/
│       │       └── db_utils.h
│       ├── maps/
│       │   ├── room1.yaml
│       │   ├── room1.pgm
│       │   ├── room2.yaml
│       │   ├── room2.pgm
│       │   ├── room3.yaml
│       │   └── room3.pgm
│       ├── package.xml
│       ├── src/
│       │   └── scripts/
│       │       ├── db_utils.cpp
│       │       ├── navigation.cpp
│       │       └── wormhole_test.cpp
│       └── wormholes.db

```

---

## Dependencies

- ROS Noetic
- turtlebot3
- move_base, amcl
- actionlib, actionlib_msgs, move_base_msgs
- sqlite3 (system: `sudo apt install libsqlite3-dev`)

---

## Build Instructions

```
cd ~/catkin_ws
catkin_make
source devel/setup.bash
```

---

## How It Works

### Step 1: Mapping Each Room Individually

I created separate maps for each room (Room 1, Room 2, Room 3) using TurtleBot3 in the Gazebo `turtlebot3_house` environment. For each room:

1. Launch the robot in Gazebo.
2. Launch SLAM using `gmapping`.
3. Drive the robot only inside that room until a full map is built in RViz.
4. Save the map using:
   `rosrun map_server map_saver -f ~/catkin_ws/src/anscer_nav/maps/room1`
5. Repeat the process for the other rooms.

This resulted in individual map files: `room1.yaml`, `room2.yaml`, and `room3.yaml`.

### Step 2: Selecting Wormhole Points (Overlapping Regions)

A wormhole is a specific position (x, y) that represents a physical doorway or overlap between two maps. To define wormholes:

1. Load each room's map using `map_server` and RViz.
2. Use the `Publish Point` tool in RViz.
3. Click near the hallway or door that connects two rooms.
4. Copy the coordinates shown in the terminal.
5. Repeat this for all valid connections (room1 to room2, room2 to room3, etc.).

### Step 3: Creating the Wormhole Database

I used SQLite to store the wormhole coordinates in a table called `wormholes`. Each entry includes:

1. `from_map`: The current map (e.g., room1)
2. `to_map`: The destination map (e.g., room2)
3. `x`: X-coordinate of the wormhole
4. `y`: Y-coordinate of the wormhole

Example entries:

```sql
INSERT INTO wormholes VALUES ('room1', 'room2', -5.0, 3.1);
INSERT INTO wormholes VALUES ('room2', 'room3', 2.5, -1.0);
INSERT INTO wormholes VALUES ('room2', 'room1', -5.0, 3.1);
```

This database is stored in `wormholes.db`.

### Step 4: Accessing the Wormhole DB from Code

I created a function in `db_utils.cpp`:
This opens `wormholes.db`, runs a SQL query, and returns the wormhole (x, y) if it exists. If no match is found, it returns (-1.0, -1.0).

### Step 5: The Action Server (navigation\_node.cpp)

I implemented a custom action server using `actionlib` with the `NavigateToMap.action` file. The server:

1. Accepts a goal: `x`, `y`, and `target_map`.
2. Checks if the robot is already in the target map (based on `/current_map` parameter).
3. If in the same map:
   * Sends the goal directly to `move_base`.
4. If in a different map:
   * Queries the wormhole (x, y) using `getWormholeCoordinates`.
   * Sends the robot to the wormhole.
   * Simulates map switching by updating `/current_map` and waiting 3 seconds.
   * Sends the final goal (x, y) in the new map.



### Step 6: Sending Navigation Goals

Goals are sent to the action server using `rostopic pub`:

```
rostopic pub /navigate_to_map/goal anscer_nav/NavigateToMapActionGoal "header:
  seq: 0
  stamp: {secs: 0, nsecs: 0}
  frame_id: ''
goal_id:
  stamp: {secs: 0, nsecs: 0}
  id: ''
goal:
  x: -6.3
  y: 0.972
  target_map: 'room3'"
```

The server handles the rest: going to the wormhole if needed, waiting for map switch, and moving to the final target.

### Summary

- Each room was mapped and saved individually.
- Wormhole positions were manually identified and stored in a SQLite database.
- The action server uses that database to decide whether to switch maps.
- The robot first goes to a wormhole, waits for the new map, and then moves to the final goal.
- Actual map switching and pose setting are manual and must be done by the user.
- Navigation is handled by `move_base` in each map.

---

## Manual Testing Steps

1. Launch Gazebo:

`roslaunch turtlebot3_gazebo turtlebot3_house.launch`


2. Launch map navigation stack:

`roslaunch turtlebot3_navigation turtlebot3_navigation.launch map_file:=/path/to/room1.yaml`
`rosparam set /current_map room1`


3. Run the action server:

`rosrun anscer_nav navigation_node`


4. Send a test goal:

```
rostopic pub /navigate_to_map/goal anscer_nav/NavigateToMapActionGoal "header:
  seq: 0
  stamp: {secs: 0, nsecs: 0}
  frame_id: ''
goal_id:
  stamp: {secs: 0, nsecs: 0}
  id: ''
goal:
  x: -5
  y: 3
  target_map: 'room3'"
```

5. After robot reaches wormhole:
   - Manually kill `/map_server` and `/amcl`
   - Launch new map with `room2.yaml`
   - Set robot pose near wormhole using 2D Pose Estimate in RViz

6. Robot completes goal in new map

---

## OOP and Code Design

- `NavigationServer` class encapsulates action server logic
- `db_utils` module handles SQLite queries
- Clean separation between test logic, DB, and navigation

---

## Final Notes

- This system simulates map switching — actual map reloading must be done manually
- The robot moves between rooms using wormholes stored in `wormholes.db`
- Fully functional with action client, navigation stack, and Gazebo

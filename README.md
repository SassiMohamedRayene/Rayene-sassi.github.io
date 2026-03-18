# Search-Based Fuzzing for Robot Controller Testing
**iRobot Create — Webots Simulation**
**Author:** Rayene

---

## What This Project Does

This project implements a robot controller for the **iRobot Create** robot in the **Webots** simulator. The robot drives from a starting position to a target point while automatically avoiding obstacles.

The project also runs two extra tasks **in parallel** (at the same time as the robot), using Python threads:

1. **A Genetic Algorithm (GA)** — evolves the string `"hello world"` as a parallel computation example.
2. **A Search-Based Fuzzer** — automatically generates obstacles in the environment to stress-test the robot controller.

---

## Project Structure

The entire project is one Python file: `robot_controller_with_avoidance.py`

It contains three parts that run simultaneously:

| Part | What it does |
|------|--------------|
| **Robot Controller** | Reads sensors, avoids obstacles, drives to target |
| **GA Thread** | Evolves `"hello world"` in a background thread |
| **Fuzzer Thread** | Places obstacles using a hillclimber search algorithm |

---

## Part 1 — Genetic Algorithm

The GA starts with a population of 200 random strings and evolves them toward `"hello world"` using:
- **Selection** — keep the best individuals
- **Crossover** — combine two parents to make a child
- **Mutation** — randomly change a character with 1% probability

It runs in a **daemon thread**, so it does not block the robot. Both finish independently and results are printed together at the end.

---

## Part 2 — Search-Based Fuzzer (Hillclimber)

This is the main research contribution of the project.

**The idea:** Instead of testing the robot with fixed obstacles, the fuzzer *searches* for the obstacle placement that disturbs the robot the most.

**How it works:**

1. Generate a random set of 3 obstacles along the robot's path
2. Place them in the Webots scene using the **Supervisor API**
3. Wait and observe the robot's reaction (speed drop, collisions)
4. Compute a **fitness score** — higher = more disturbing
5. Mutate one obstacle slightly (shift its position or size)
6. Keep the mutation only if fitness improves *(hillclimber rule)*
7. Repeat for 25 iterations

**Fitness function:**
```
fitness = speed_drop + (collision_count × 5)
```
The fuzzer finds the worst-case environment for the robot automatically, without a human placing obstacles manually.

> This follows the Search-Based Fuzzing approach from [fuzzingbook.org](https://www.fuzzingbook.org/html/SearchBasedFuzzer.html).

---

## Part 3 — Obstacle Avoidance

The robot uses the **real sensors** of the iRobot Create model in Webots:

| Sensor | Purpose |
|--------|---------|
| `bumper_left` / `bumper_right` | Detects physical contact with an obstacle |
| `cliff_front_left` / `cliff_front_right` | Detects edges or drops ahead |

**Avoidance logic (state machine):**

- **FORWARD** → drive straight toward target
- **BACKWARD** → back up for ~15 steps after a collision
- **TURN** → rotate left or right for ~25 steps, then return to FORWARD

The direction of the turn depends on which bumper was hit.

---

## How to Run

1. Open Webots and load the iRobot Create world
2. In the scene tree, select the robot node and set `supervisor = TRUE`
3. Set the controller to `robot_controller_with_avoidance.py`
4. Press **Play**

The robot will start moving, the GA thread and fuzzer thread will launch automatically after a short warm-up period.

---

## Output

At the end of the simulation, the console prints a full report:

```
=============================================
   ROBOT PERFORMANCE RESULTS
=============================================
Total Time        : 12.340 s
Distance Traveled : 2.187 m
Average Speed     : 0.177 m/s
Total Collisions  : 3
---------------------------------------------
   GA RESULTS
---------------------------------------------
GA Time           : 4.2731 s
GA Generations    : 187
GA Solution       : 'hello world'
---------------------------------------------
   FUZZER RESULTS
---------------------------------------------
Fuzzer Iterations : 25
Best Fitness      : 0.8420
Best Layout       : [(0.6, 0.1, 0.15), (1.2, -0.2, 0.18), (1.7, 0.3, 0.12)]
=============================================
```

---

## Key Concepts Used

- **Fuzz Testing** — automated generation of test inputs to find unexpected behavior
- **Search-Based Testing** — using a fitness function to guide input generation toward a goal
- **Hillclimbing** — a local search algorithm that keeps improvements and discards bad mutations
- **Parallel Threads** — running multiple tasks at the same time without blocking each other
- **Webots Supervisor API** — allows modifying the simulation scene at runtime (adding/removing objects)

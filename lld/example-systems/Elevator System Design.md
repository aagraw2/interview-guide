
## 1. Functional Requirements

- User can request an elevator (up/down) from a floor
    
- User can select destination floor inside elevator
    
- System assigns best elevator to request
    
- Elevator moves between floors
    
- Maintain elevator state (idle, moving, stopped)
    
- Handle multiple elevators
    
- Optimize for minimum wait time
    

---

## 2. Non-Functional Requirements

- Low latency elevator assignment
    
- High concurrency (multiple requests simultaneously)
    
- Scalable to multiple buildings/floors
    
- Fault-tolerant (handle elevator failure)
    
- Extensible for new scheduling algorithms
    
- Thread-safe operations
    

---

## 3. Core Entities

|Entity|Responsibility|
|---|---|
|Elevator|Represents elevator with state and movement|
|Floor|Represents a floor|
|Request|External/Internal request|
|ElevatorController|Controls a single elevator|
|ElevatorSystem|Manages all elevators and dispatching|
|SchedulerStrategy|Decides which elevator to assign|
|State|Represents elevator state|

---

## 4. Flows (Use Cases)

### External Request (Main Flow)

1. User presses up/down button on a floor
    
2. System creates request
    
3. SchedulerStrategy selects best elevator
    
4. Request assigned to elevator
    
5. Elevator moves toward source floor
    

---

### Internal Request (Main Flow)

1. User selects destination inside elevator
    
2. Request added to elevator queue
    
3. Elevator moves toward destination
    
4. Stops at requested floor
    

---

### Elevator Movement (Supporting Flow)

1. Elevator checks current direction
    
2. Moves one floor at a time
    
3. Stops if request exists for that floor
    
4. Opens/closes doors
    
5. Updates state
    

---

### Scheduling (Supporting Flow)

1. Iterate over all elevators
    
2. Evaluate based on:
    
    - Distance
        
    - Direction
        
    - Current load/state
        
3. Select best suited elevator
    

---

## 5. Code

```java
import java.util.*;

// MODELS

enum Direction {
    UP, DOWN, IDLE
}

enum ElevatorStateType {
    IDLE, MOVING, STOPPED
}

class Request {
    int sourceFloor;
    int destinationFloor;
    Direction direction;

    public Request(int sourceFloor, int destinationFloor) {
        this.sourceFloor = sourceFloor;
        this.destinationFloor = destinationFloor;
        this.direction = destinationFloor > sourceFloor ? Direction.UP : Direction.DOWN;
    }
}

// State Pattern for Elevator behavior
interface ElevatorState {
    void handle(Elevator elevator);
}

class IdleState implements ElevatorState {
    public void handle(Elevator elevator) {
        if (!elevator.requests.isEmpty()) {
            elevator.setState(new MovingState());
        }
    }
}

class MovingState implements ElevatorState {
    public void handle(Elevator elevator) {
        if (elevator.direction == Direction.UP) {
            elevator.currentFloor++;
        } else if (elevator.direction == Direction.DOWN) {
            elevator.currentFloor--;
        }

        if (elevator.requests.contains(elevator.currentFloor)) {
            elevator.setState(new StoppedState());
        }
    }
}

class StoppedState implements ElevatorState {
    public void handle(Elevator elevator) {
        elevator.requests.remove(elevator.currentFloor);
        elevator.setState(new IdleState());
    }
}

class Elevator {
    int id;
    int currentFloor;
    Direction direction;
    ElevatorState state;
    Set<Integer> requests;

    public Elevator(int id) {
        this.id = id;
        this.currentFloor = 0;
        this.direction = Direction.IDLE;
        this.state = new IdleState();
        this.requests = new TreeSet<>();
    }

    public void addRequest(int floor) {
        requests.add(floor);
        if (floor > currentFloor) direction = Direction.UP;
        else if (floor < currentFloor) direction = Direction.DOWN;
    }

    public void move() {
        state.handle(this);
    }

    public void setState(ElevatorState state) {
        this.state = state;
    }
}

// Strategy Pattern for scheduling
interface SchedulerStrategy {
    Elevator selectElevator(List<Elevator> elevators, Request request);
}

// Simple nearest elevator strategy
class NearestElevatorStrategy implements SchedulerStrategy {
    @Override
    public Elevator selectElevator(List<Elevator> elevators, Request request) {
        Elevator best = null;
        int minDistance = Integer.MAX_VALUE;

        for (Elevator e : elevators) {
            int distance = Math.abs(e.currentFloor - request.sourceFloor);
            if (distance < minDistance) {
                minDistance = distance;
                best = e;
            }
        }
        return best;
    }
}

// SERVICES

// Singleton Elevator System
class ElevatorSystem {
    private static ElevatorSystem instance;

    private List<Elevator> elevators;
    private SchedulerStrategy scheduler;

    private ElevatorSystem(int numElevators) {
        elevators = new ArrayList<>();
        for (int i = 0; i < numElevators; i++) {
            elevators.add(new Elevator(i));
        }
        scheduler = new NearestElevatorStrategy();
    }

    public static ElevatorSystem getInstance(int numElevators) {
        if (instance == null) {
            synchronized (ElevatorSystem.class) {
                if (instance == null) {
                    instance = new ElevatorSystem(numElevators);
                }
            }
        }
        return instance;
    }

    // Handle external request
    public void requestElevator(int source, int destination) {
        Request request = new Request(source, destination);

        Elevator elevator = scheduler.selectElevator(elevators, request);
        if (elevator == null) throw new RuntimeException("No elevator available");

        elevator.addRequest(source);
        elevator.addRequest(destination);
    }

    // Simulate movement
    public void step() {
        for (Elevator e : elevators) {
            e.move();
        }
    }
}
```

---

## 6. Complexity Analysis

### External Request

- Iterate elevators → O(N)
    
- Assign request → O(log K) (TreeSet insert)
    

**Overall: O(N)**

---

### Internal Request

- Add request → O(log K)
    

**Overall: O(log K)**

---

### Elevator Movement

- State handling → O(1)
    
- Request lookup → O(log K)
    

**Overall: O(log K)**

---

### Scheduling

- Scan all elevators → O(N)
    

**Overall: O(N)**

---

## 7. Key Design Decisions

- State Pattern cleanly models elevator behavior transitions
    
- Strategy Pattern allows pluggable scheduling algorithms
    
- Singleton ensures centralized elevator system control
    
- TreeSet used for ordered request handling
    
- Separation between scheduling and elevator logic improves extensibility
    
- Simple nearest strategy can be replaced with smarter algorithms (SCAN, LOOK)

## 1. Functional Requirements

- Vehicle enters parking lot
    
- System assigns nearest available slot
    
- Generate parking ticket
    
- Vehicle exits using ticket
    
- Calculate parking fee based on duration
    
- Support multiple vehicle types (Car, Bike, Truck)
    
- Support multiple floors
    
- Track slot availability (Free/Occupied)
    

---

## 2. Non-Functional Requirements

- Low latency slot allocation
    
- High concurrency support (multiple entries/exits)
    
- Scalable to multiple floors and large capacity
    
- Extensible for new vehicle types or pricing rules
    
- Fault-tolerant ticket handling
    
- Thread-safe operations
    

---

## 3. Core Entities

|Entity|Responsibility|
|---|---|
|Vehicle|Represents a vehicle entering the lot|
|ParkingSlot|Represents a slot with type and availability|
|Floor|Contains multiple parking slots|
|Ticket|Tracks entry time, vehicle, and assigned slot|
|ParkingLot|Manages floors and slot allocation|
|FeeStrategy|Calculates parking fee|
|SlotAllocationStrategy|Finds appropriate slot|
|ParkingService|Handles entry and exit operations|

---

## 4. Flows (Use Cases)

### Vehicle Entry (Main Flow)

1. Vehicle arrives at entry gate
    
2. System identifies vehicle type
    
3. SlotAllocationStrategy finds nearest available slot
    
4. Slot marked as occupied
    
5. Ticket generated with entry time and slot
    
6. Ticket returned to user
    

---

### Vehicle Exit (Main Flow)

1. User provides ticket
    
2. System fetches ticket details
    
3. Calculate duration of parking
    
4. FeeStrategy calculates fee
    
5. Slot marked as free
    
6. Ticket closed
    

---

### Slot Allocation (Supporting Flow)

1. Iterate through floors
    
2. For each floor, check slots of required vehicle type
    
3. Return first available slot
    
4. If none found → return null
    

---

### Fee Calculation (Supporting Flow)

1. Calculate parking duration
    
2. Apply pricing rule (hourly / slab-based)
    
3. Return total fee
    

---

## 5. Code

```java
import java.util.*;

// MODELS

enum VehicleType {
    BIKE, CAR, TRUCK
}

class Vehicle {
    String number;
    VehicleType type;

    public Vehicle(String number, VehicleType type) {
        this.number = number;
        this.type = type;
    }
}

class ParkingSlot {
    int id;
    VehicleType type;
    boolean isFree;

    public ParkingSlot(int id, VehicleType type) {
        this.id = id;
        this.type = type;
        this.isFree = true;
    }
}

class Floor {
    int id;
    List<ParkingSlot> slots;

    public Floor(int id, List<ParkingSlot> slots) {
        this.id = id;
        this.slots = slots;
    }
}

class Ticket {
    String id;
    Vehicle vehicle;
    ParkingSlot slot;
    long entryTime;
    long exitTime;

    public Ticket(String id, Vehicle vehicle, ParkingSlot slot) {
        this.id = id;
        this.vehicle = vehicle;
        this.slot = slot;
        this.entryTime = System.currentTimeMillis();
    }
}

// Strategy Pattern for slot allocation
interface SlotAllocationStrategy {
    ParkingSlot allocateSlot(List<Floor> floors, VehicleType type);
}

// Nearest slot allocation
class NearestSlotStrategy implements SlotAllocationStrategy {
    @Override
    public ParkingSlot allocateSlot(List<Floor> floors, VehicleType type) {
        for (Floor floor : floors) {
            for (ParkingSlot slot : floor.slots) {
                if (slot.isFree && slot.type == type) {
                    return slot;
                }
            }
        }
        return null;
    }
}

// Strategy Pattern for fee calculation
interface FeeStrategy {
    double calculateFee(long entryTime, long exitTime);
}

// Simple hourly pricing
class HourlyFeeStrategy implements FeeStrategy {
    @Override
    public double calculateFee(long entryTime, long exitTime) {
        long duration = (exitTime - entryTime) / (1000 * 60 * 60);
        duration = Math.max(1, duration);
        return duration * 20.0;
    }
}

// Factory Pattern for creating vehicle
class VehicleFactory {
    public static Vehicle createVehicle(String number, VehicleType type) {
        return new Vehicle(number, type);
    }
}

// SERVICES

// Singleton ParkingLot
class ParkingLot {
    private static ParkingLot instance;

    private List<Floor> floors;
    private SlotAllocationStrategy slotStrategy;

    private ParkingLot(List<Floor> floors) {
        this.floors = floors;
        this.slotStrategy = new NearestSlotStrategy();
    }

    public static ParkingLot getInstance(List<Floor> floors) {
        if (instance == null) {
            synchronized (ParkingLot.class) {
                if (instance == null) {
                    instance = new ParkingLot(floors);
                }
            }
        }
        return instance;
    }

    public ParkingSlot allocateSlot(VehicleType type) {
        return slotStrategy.allocateSlot(floors, type);
    }
}

// Facade Pattern to simplify client interaction
class ParkingService {
    private ParkingLot parkingLot;
    private FeeStrategy feeStrategy;
    private Map<String, Ticket> ticketStore = new HashMap<>();

    public ParkingService(ParkingLot parkingLot) {
        this.parkingLot = parkingLot;
        this.feeStrategy = new HourlyFeeStrategy();
    }

    // Entry flow
    public Ticket parkVehicle(String number, VehicleType type) {
        Vehicle vehicle = VehicleFactory.createVehicle(number, type);

        ParkingSlot slot = parkingLot.allocateSlot(type);
        if (slot == null) throw new RuntimeException("No slots available");

        slot.isFree = false;

        Ticket ticket = new Ticket(UUID.randomUUID().toString(), vehicle, slot);
        ticketStore.put(ticket.id, ticket);

        return ticket;
    }

    // Exit flow
    public double exitVehicle(String ticketId) {
        Ticket ticket = ticketStore.get(ticketId);
        if (ticket == null) throw new RuntimeException("Invalid ticket");

        ticket.exitTime = System.currentTimeMillis();

        double fee = feeStrategy.calculateFee(ticket.entryTime, ticket.exitTime);

        ticket.slot.isFree = true;
        ticketStore.remove(ticketId);

        return fee;
    }
}
```

---

## 6. Complexity Analysis

### Vehicle Entry

- Slot search (worst case) → O(F × S)
    
- Insert into map → O(1)
    

**Overall: O(F × S)**

---

### Vehicle Exit

- Ticket lookup → O(1)
    
- Fee calculation → O(1)
    
- Slot update → O(1)
    

**Overall: O(1)**

---

### Slot Allocation

- Iterate floors and slots → O(F × S)
    

**Overall: O(F × S)**

---

### Fee Calculation

- Constant time computation
    

**Overall: O(1)**

---

## 7. Key Design Decisions

- Strategy Pattern for flexible slot allocation and pricing rules
    
- Singleton ParkingLot ensures single source of truth
    
- Facade (ParkingService) simplifies interaction for clients
    
- Factory Pattern decouples vehicle creation
    
- Separation of concerns between allocation, pricing, and orchestration
    
- In-memory storage for tickets enables fast lookup; can be extended to DB for persistence
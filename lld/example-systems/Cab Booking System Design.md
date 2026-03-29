
# 1. Functional Requirements

- User can request a cab
    
- System matches user with nearest available driver
    
- Driver can accept/reject ride
    
- Track ride status (REQUESTED → ACCEPTED → STARTED → COMPLETED → CANCELLED)
    
- Calculate fare
    
- Support multiple vehicle types (Mini, Sedan, SUV)
    
- Handle cancellations
    
- Maintain ride history
    

---

# 2. Non-Functional Requirements

- Low latency driver matching
    
- High availability
    
- Scalable system
    
- Fault-tolerant
    
- Real-time updates
    
- Thread-safe operations
    

---

# 3. Core Entities

|Entity|Responsibility|
|---|---|
|User|Ride requester|
|Driver|Cab driver|
|Vehicle|Driver’s vehicle|
|Ride|Trip details|
|Location|Latitude & Longitude|
|RideStatus|Lifecycle state|
|DriverRepository|Stores drivers|
|RideRepository|Stores rides|
|MatchingService|Driver allocation|
|FareStrategy|Fare calculation|
|RideService|Ride orchestration|

---

# 4. Flows

---

## Request Ride

1. User sends request with pickup & drop
    
2. MatchingService finds nearest available driver
    
3. Create ride (REQUESTED)
    
4. Mark driver unavailable
    

---

## Accept Ride

1. Driver accepts
    
2. Status → ACCEPTED
    

---

## Start Ride

1. Driver starts trip
    
2. Status → STARTED
    

---

## Complete Ride

1. Driver ends trip
    
2. Fare calculated
    
3. Status → COMPLETED
    

---

## Cancel Ride

1. User/Driver cancels
    
2. Status → CANCELLED
    
3. Driver marked available
    

---

# 5. Code

```java
import java.util.*;
import java.util.concurrent.*;

// ENUMS
enum RideStatus {
    REQUESTED, ACCEPTED, STARTED, COMPLETED, CANCELLED
}

enum VehicleType {
    MINI, SEDAN, SUV
}

// MODELS
class Location {
    double lat, lon;
    public Location(double lat, double lon) {
        this.lat = lat;
        this.lon = lon;
    }
}

class User {
    String id;
}

class Vehicle {
    String id;
    VehicleType type;
}

class Driver {
    String id;
    Location location;
    boolean available = true;
    Vehicle vehicle;
}

class Ride {
    String id;
    String userId;
    String driverId;
    Location pickup;
    Location drop;
    RideStatus status;
    double fare;

    public Ride(String id, String userId, String driverId, Location pickup, Location drop) {
        this.id = id;
        this.userId = userId;
        this.driverId = driverId;
        this.pickup = pickup;
        this.drop = drop;
        this.status = RideStatus.REQUESTED;
    }
}

// REPOSITORIES
class DriverRepository {
    private Map<String, Driver> drivers = new ConcurrentHashMap<>();

    public void save(Driver d) { drivers.put(d.id, d); }

    public List<Driver> getAvailableDrivers() {
        List<Driver> list = new ArrayList<>();
        for (Driver d : drivers.values()) {
            if (d.available) list.add(d);
        }
        return list;
    }

    public Driver get(String id) { return drivers.get(id); }
}

class RideRepository {
    private Map<String, Ride> rides = new ConcurrentHashMap<>();

    public void save(Ride r) { rides.put(r.id, r); }

    public Ride get(String id) { return rides.get(id); }
}

// STRATEGY: Matching
interface MatchingStrategy {
    Driver match(List<Driver> drivers, Location pickup);
}

class NearestDriverStrategy implements MatchingStrategy {
    public Driver match(List<Driver> drivers, Location pickup) {
        Driver best = null;
        double minDist = Double.MAX_VALUE;

        for (Driver d : drivers) {
            double dist = Math.sqrt(
                Math.pow(d.location.lat - pickup.lat, 2) +
                Math.pow(d.location.lon - pickup.lon, 2)
            );
            if (dist < minDist) {
                minDist = dist;
                best = d;
            }
        }
        return best;
    }
}

// STRATEGY: Fare
interface FareStrategy {
    double calculate(Ride ride);
}

class DefaultFareStrategy implements FareStrategy {
    public double calculate(Ride ride) {
        double dist = Math.sqrt(
            Math.pow(ride.pickup.lat - ride.drop.lat, 2) +
            Math.pow(ride.pickup.lon - ride.drop.lon, 2)
        );
        return dist * 10;
    }
}

// SERVICES
class MatchingService {
    private DriverRepository repo;
    private MatchingStrategy strategy;

    public MatchingService(DriverRepository repo, MatchingStrategy strategy) {
        this.repo = repo;
        this.strategy = strategy;
    }

    public Driver findDriver(Location pickup) {
        return strategy.match(repo.getAvailableDrivers(), pickup);
    }
}

class RideService {
    private RideRepository rideRepo;
    private MatchingService matchingService;
    private FareStrategy fareStrategy;
    private DriverRepository driverRepo;

    public RideService(RideRepository rideRepo,
                       MatchingService matchingService,
                       FareStrategy fareStrategy,
                       DriverRepository driverRepo) {
        this.rideRepo = rideRepo;
        this.matchingService = matchingService;
        this.fareStrategy = fareStrategy;
        this.driverRepo = driverRepo;
    }

    public Ride requestRide(String userId, Location pickup, Location drop) {
        Driver driver = matchingService.findDriver(pickup);
        if (driver == null) throw new RuntimeException("No drivers");

        driver.available = false;

        Ride ride = new Ride(UUID.randomUUID().toString(), userId, driver.id, pickup, drop);
        rideRepo.save(ride);
        return ride;
    }

    public void acceptRide(String rideId) {
        rideRepo.get(rideId).status = RideStatus.ACCEPTED;
    }

    public void startRide(String rideId) {
        rideRepo.get(rideId).status = RideStatus.STARTED;
    }

    public void completeRide(String rideId) {
        Ride ride = rideRepo.get(rideId);
        ride.fare = fareStrategy.calculate(ride);
        ride.status = RideStatus.COMPLETED;
        driverRepo.get(ride.driverId).available = true;
    }

    public void cancelRide(String rideId) {
        Ride ride = rideRepo.get(rideId);
        ride.status = RideStatus.CANCELLED;
        driverRepo.get(ride.driverId).available = true;
    }
}
```

---

# 6. Complexity Analysis

- Request Ride → O(N)
    
- Fare Calculation → O(1)
    
- State Updates → O(1)
    

---

# 7. Design Patterns Used

- Strategy Pattern → Matching, Fare calculation
    
- Repository Pattern → Data access abstraction
    
- Dependency Injection → Loose coupling
    
- (Optional Extension) State Pattern → Ride lifecycle
    

---

# 8. Key Design Decisions

- Matching and fare logic kept pluggable
    
- Driver availability handled atomically
    
- Ride lifecycle clearly defined via enum
    
- Services separated for modularity
    
- Thread-safe repositories used
    
- Simple distance metric (replaceable with geo-indexing)
    
- System extensible for surge pricing, pooling, scheduling
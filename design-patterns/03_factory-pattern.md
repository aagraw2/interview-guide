# Factory Pattern

## Intent
The Factory pattern provides an interface for creating objects while allowing subclasses or logic to decide which object to create.

## When to Use
- Object creation is complex
- Type decided at runtime
- Decouple creation from usage

## Example
```java
public interface Vehicle{
    String getName();
    int getPrice();
}
public class Bike implements Vehicle{
    public String getName(){
        return "BIKE";
    }

    public int getPrice(){
        return 200000;
    }
}

public class Car implements Vehicle{
    public String getName(){
        return "CAR";
    }

    public int getPrice(){
        return 1500000;
    }
}

public class Truck implements Vehicle{
    public String getName(){
        return "TRUCK";
    }

    public int getPrice(){
        return 3000000;
    }
}

public class VehicleFactory{
    private final Map<String, Supplier<Vehicle>> registry = new HashMap<>();

    public void register(String name, Supplier<Vehicle> creator) {
        registry.put(name, creator);
    }

    public Vehicle getVehicle(String name) {

        Supplier<Vehicle> supplier = registry.get(name);

        if (supplier == null) {
            throw new IllegalArgumentException("Vehicle not found");
        }

        return supplier.get();
    }
}

public class Main {

    public static void main(String[] args) {

        VehicleFactory factory = new VehicleFactory();

        factory.register("CAR", Car::new);
        factory.register("BIKE", Bike::new);
        factory.register("TRUCK", Truck::new);

        Vehicle car = factory.getVehicle("CAR");
        Vehicle bike = factory.getVehicle("BIKE");

        System.out.println(car.getName());
        System.out.println(bike.getPrice());
    }
}
```
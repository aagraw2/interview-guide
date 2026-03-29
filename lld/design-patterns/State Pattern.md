
## Intent
The State pattern changes object behavior based on internal state.

## When to Use
- State-driven behavior
- Avoid conditionals
- Multiple states

## Example
```java
public interface TrafficLightState {
    void handle(TrafficLight trafficLight);
}

public class RedState implements TrafficLightState {

    @Override
    public void handle(TrafficLight trafficLight) {
        System.out.println("Red Light - STOP");
        trafficLight.setState(new GreenState());
    }
}

public class GreenState implements TrafficLightState {

    @Override
    public void handle(TrafficLight trafficLight) {
        System.out.println("Green Light - GO");
        trafficLight.setState(new YellowState());
    }
}

public class YellowState implements TrafficLightState {

    @Override
    public void handle(TrafficLight trafficLight) {
        System.out.println("Yellow Light - SLOW DOWN");
        trafficLight.setState(new RedState());
    }
}

public class TrafficLight {

    private TrafficLightState state;

    public TrafficLight() {
        this.state = new RedState();
    }

    public void setState(TrafficLightState state) {
        this.state = state;
    }

    public void change() {
        state.handle(this);
    }
}

public class Main {

    public static void main(String[] args) {

        TrafficLight light = new TrafficLight();

        light.change();
        light.change();
        light.change();
        light.change();
    }
}
```

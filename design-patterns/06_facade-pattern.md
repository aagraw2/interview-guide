# Facade Pattern

## Intent
The Facade pattern provides a simplified interface (single entry point) to a complex subsystem.

## When to Use
- Many subsystems
- Simplify usage
- Hide complexity

## Example
```java
public class TV {

    public void turnOn() {
        System.out.println("TV is ON");
    }

    public void turnOff() {
        System.out.println("TV is OFF");
    }
}

public class Lights {

    public void dim() {
        System.out.println("Lights are dimmed");
    }

    public void on() {
        System.out.println("Lights are ON");
    }
}

public class SoundSystem {

    public void turnOn() {
        System.out.println("Sound system is ON");
    }

    public void turnOff() {
        System.out.println("Sound system is OFF");
    }

    public void setVolume(int level) {
        System.out.println("Volume set to " + level);
    }
}

public class HomeTheaterFacade {

    private TV tv;
    private Lights lights;
    private SoundSystem soundSystem;

    public HomeTheaterFacade(TV tv, Lights lights, SoundSystem soundSystem) {
        this.tv = tv;
        this.lights = lights;
        this.soundSystem = soundSystem;
    }

    public void watchMovie() {

        System.out.println("Starting movie...");

        lights.dim();
        tv.turnOn();
        soundSystem.turnOn();
        soundSystem.setVolume(10);
    }

    public void endMovie() {

        System.out.println("Shutting down theater...");

        tv.turnOff();
        soundSystem.turnOff();
        lights.on();
    }
}

public class Main {

    public static void main(String[] args) {

        TV tv = new TV();
        Lights lights = new Lights();
        SoundSystem sound = new SoundSystem();

        HomeTheaterFacade theater = new HomeTheaterFacade(tv, lights, sound);

        theater.watchMovie();

        theater.endMovie();
    }
}
```

# Builder Pattern

## Intent
The Builder pattern helps in constructing complex objects step by step. It separates object construction logic from representation (how the object is structured internally).

## When to Use
- Many optional parameters
- Constructor becomes too large
- Need immutable objects
- Step-by-step creation required

## Example
```java
public class Computer {
    private final String brand;
    private final int ram;
    private final int gpu;
    private final int cpu;
    private final int ssd;

    private Computer(Builder b){
        this.brand = b.brand;
        this.ram = b.ram;
        this.gpu = b.gpu;
        this.cpu = b.cpu;
        this.ssd = b.ssd;
    }

    public static class Builder {
        private String brand;
        private int ram;
        private int gpu;
        private int cpu;
        private int ssd;

        public Builder(String brand){
            this.brand = brand;
        }

        public Builder setRAM(int ram){
            this.ram = ram;
            return this;
        }

        public Builder setGPU(int gpu){
            this.gpu = gpu;
            return this;
        }

        public Builder setCPU(int cpu){
            this.cpu = cpu;
            return this;
        }

        public Builder setSSD(int ssd){
            this.ssd = ssd;
            return this;
        }

        public Computer build(){
            return new Computer(this);
        }
    }
}

public class Main {
    public static void main(String[] args) {

        Computer computer = new Computer.Builder("Dell")
                .setRam(16)
                .setCpu(8)
                .setGpu(6)
                .setSsd(512)
                .build();

        System.out.println(computer);
    }
}
```

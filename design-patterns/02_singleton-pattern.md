# Singleton Pattern

## Intent
This pattern ensures that only one instance of a class exists and provides a global access point (accessible from anywhere in code).

## When to Use
- Only one instance needed
- Shared resource management
- Global access required

## Example
```java
public class Logger {
    private static Logger instance;

    private Logger(){}

    public static Logger getLogger(){
        if(instance==null){
            instance = new Logger();
        }
        return instance;
    }

    public void log(String message){
        System.out.println(message);
    }
}

public class Main {
    public static void main(String[] args) {

        Logger logger = Logger.getLogger();

        logger.log("First message");
        logger.log("Second message");
    }
}
```

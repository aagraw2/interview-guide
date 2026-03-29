
## Intent
The Observer pattern defines a one-to-many relationship where multiple objects are notified of changes.

## When to Use
- Event systems
- Notifications
- Publish-subscribe

## Example
```java
public interface Observer {
    void update(Subject subject);
}

public class User implements Observer {

    private String name;

    public User(String name) {
        this.name = name;
    }

    @Override
    public void update(Subject subject) {

        Product product = (Product) subject;

        System.out.println(
            name + " notified: " +
            product.getName() +
            " price changed to " +
            product.getPrice()
        );
    }
}

public interface Subject {

    void registerObserver(Observer observer);

    void removeObserver(Observer observer);

    void notifyObservers();
}

import java.util.*;

public class Product implements Subject {

    private List<Observer> observers = new ArrayList<>();
    private String name;
    private int price;

    public Product(String name, int price) {
        this.name = name;
        this.price = price;
    }

    public void setPrice(int price) {
        this.price = price;
        notifyObservers();
    }

    public int getPrice() {
        return price;
    }

    public String getName() {
        return name;
    }

    @Override
    public void registerObserver(Observer observer) {
        observers.add(observer);
    }

    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }

    @Override
    public void notifyObservers() {
        for (Observer o : observers) {
            o.update(this);
        }
    }
}

public class Main {

    public static void main(String[] args) {

        Product iphone = new Product("iPhone", 100000);

        User user1 = new User("Amit");
        User user2 = new User("Rahul");

        iphone.registerObserver(user1);
        iphone.registerObserver(user2);

        iphone.setPrice(95000);
        iphone.setPrice(90000);
    }
}
```

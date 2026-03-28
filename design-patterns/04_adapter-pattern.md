# Adapter Pattern

## Intent
The Adapter pattern allows incompatible interfaces (classes that cannot directly work together) to collaborate.

## When to Use
- Integrating third-party systems
- Legacy code reuse
- Interface mismatch

## Example
```java
public class PayPalPayment { //Existing Class

    public void makePayment(double amount) {
        System.out.println("Processing PayPal payment of $" + amount);
    }
}

public interface PaymentProcessor { //Target Interface
    void processPayment(double amount);
}

public class PayPalAdapter implements PaymentProcessor {

    private PayPalPayment payPalPayment;

    public PayPalAdapter(PayPalPayment payPalPayment) {
        this.payPalPayment = payPalPayment;
    }

    @Override
    public void processPayment(double amount) {
        payPalPayment.makePayment(amount);
    }
}

public class Main {

    public static void main(String[] args) {

        PayPalPayment payPal = new PayPalPayment();

        PaymentProcessor processor = new PayPalAdapter(payPal);

        processor.processPayment(500);
    }
}
```
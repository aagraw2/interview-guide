
## Intent
This pattern passes a request along a chain of handlers until it is processed.

## When to Use
- Multiple handlers
- Sequential processing
- Decoupled sender

## Example
```java
public class ExpenseRequest {

    private String employee;
    private int amount;

    public ExpenseRequest(String employee, int amount) {
        this.employee = employee;
        this.amount = amount;
    }

    public int getAmount() {
        return amount;
    }

    public String getEmployee() {
        return employee;
    }
}

public abstract class Approver {

    protected Approver next;

    public void setNext(Approver next) {
        this.next = next;
    }

    public void approve(ExpenseRequest request) {

        if (canApprove(request)) {
            process(request);
        } else if (next != null) {
            next.approve(request);
        } else {
            System.out.println("Request cannot be approved");
        }
    }

    protected abstract boolean canApprove(ExpenseRequest request);

    protected abstract void process(ExpenseRequest request);
}

public class Manager extends Approver {

    @Override
    protected boolean canApprove(ExpenseRequest request) {
        return request.getAmount() <= 1000;
    }

    @Override
    protected void process(ExpenseRequest request) {
        System.out.println("Manager approved " + request.getAmount());
    }
}

public class Director extends Approver {

    @Override
    protected boolean canApprove(ExpenseRequest request) {
        return request.getAmount() <= 5000;
    }

    @Override
    protected void process(ExpenseRequest request) {
        System.out.println("Director approved " + request.getAmount());
    }
}

public class CEO extends Approver {

    @Override
    protected boolean canApprove(ExpenseRequest request) {
        return request.getAmount() <= 20000;
    }

    @Override
    protected void process(ExpenseRequest request) {
        System.out.println("CEO approved " + request.getAmount());
    }
}

public class Main {

    public static void main(String[] args) {

        Approver manager = new Manager();
        Approver director = new Director();
        Approver ceo = new CEO();

        manager.setNext(director);
        director.setNext(ceo);

        ExpenseRequest r1 = new ExpenseRequest("Amit", 500);
        ExpenseRequest r2 = new ExpenseRequest("Rahul", 3000);
        ExpenseRequest r3 = new ExpenseRequest("Priya", 15000);

        manager.approve(r1);
        manager.approve(r2);
        manager.approve(r3);
    }
}
```

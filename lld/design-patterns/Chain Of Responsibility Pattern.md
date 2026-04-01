## 1. What is Chain of Responsibility Pattern?

Chain of Responsibility passes a request along a chain of handlers. Each handler decides either to process the request or pass it to the next handler in the chain.

```
Without Chain:
  if (amount <= 1000) manager.approve();
  else if (amount <= 5000) director.approve();
  else if (amount <= 20000) ceo.approve();
  // Tight coupling

With Chain:
  manager → director → ceo
  Request flows through chain until handled
```

**Key principle:** Decouple sender from receiver by giving multiple objects a chance to handle the request.

---

## 2. Implementation Guide

### Step 1: Define Handler Interface/Abstract Class

```java
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
```

### Step 2: Create Request Class

```java
public class ExpenseRequest {
    private String employee;
    private double amount;
    private String purpose;
    
    public ExpenseRequest(String employee, double amount, String purpose) {
        this.employee = employee;
        this.amount = amount;
        this.purpose = purpose;
    }
    
    public String getEmployee() { return employee; }
    public double getAmount() { return amount; }
    public String getPurpose() { return purpose; }
}
```

### Step 3: Implement Concrete Handlers

```java
public class Manager extends Approver {
    @Override
    protected boolean canApprove(ExpenseRequest request) {
        return request.getAmount() <= 1000;
    }
    
    @Override
    protected void process(ExpenseRequest request) {
        System.out.println("Manager approved $" + request.getAmount() + 
                         " for " + request.getEmployee());
    }
}

public class Director extends Approver {
    @Override
    protected boolean canApprove(ExpenseRequest request) {
        return request.getAmount() <= 5000;
    }
    
    @Override
    protected void process(ExpenseRequest request) {
        System.out.println("Director approved $" + request.getAmount() + 
                         " for " + request.getEmployee());
    }
}

public class CEO extends Approver {
    @Override
    protected boolean canApprove(ExpenseRequest request) {
        return request.getAmount() <= 20000;
    }
    
    @Override
    protected void process(ExpenseRequest request) {
        System.out.println("CEO approved $" + request.getAmount() + 
                         " for " + request.getEmployee());
    }
}
```

### Step 4: Build Chain and Use

```java
// Build chain
Approver manager = new Manager();
Approver director = new Director();
Approver ceo = new CEO();

manager.setNext(director);
director.setNext(ceo);

// Send requests through chain
ExpenseRequest r1 = new ExpenseRequest("Alice", 500, "Office supplies");
ExpenseRequest r2 = new ExpenseRequest("Bob", 3000, "Conference");
ExpenseRequest r3 = new ExpenseRequest("Charlie", 15000, "Equipment");
ExpenseRequest r4 = new ExpenseRequest("David", 25000, "Renovation");

manager.approve(r1);  // Manager approves
manager.approve(r2);  // Director approves
manager.approve(r3);  // CEO approves
manager.approve(r4);  // Cannot be approved
```

---

## 3. Common Interview Questions

### Q1: Support Ticket System

```java
public abstract class SupportHandler {
    protected SupportHandler next;
    
    public void setNext(SupportHandler next) {
        this.next = next;
    }
    
    public void handleTicket(SupportTicket ticket) {
        if (canHandle(ticket)) {
            process(ticket);
        } else if (next != null) {
            next.handleTicket(ticket);
        } else {
            System.out.println("Ticket escalated to management");
        }
    }
    
    protected abstract boolean canHandle(SupportTicket ticket);
    protected abstract void process(SupportTicket ticket);
}

public class SupportTicket {
    private String issue;
    private String priority;  // LOW, MEDIUM, HIGH, CRITICAL
    
    public SupportTicket(String issue, String priority) {
        this.issue = issue;
        this.priority = priority;
    }
    
    public String getIssue() { return issue; }
    public String getPriority() { return priority; }
}

public class Level1Support extends SupportHandler {
    @Override
    protected boolean canHandle(SupportTicket ticket) {
        return ticket.getPriority().equals("LOW");
    }
    
    @Override
    protected void process(SupportTicket ticket) {
        System.out.println("Level 1 handling: " + ticket.getIssue());
    }
}

public class Level2Support extends SupportHandler {
    @Override
    protected boolean canHandle(SupportTicket ticket) {
        return ticket.getPriority().equals("MEDIUM");
    }
    
    @Override
    protected void process(SupportTicket ticket) {
        System.out.println("Level 2 handling: " + ticket.getIssue());
    }
}

public class Level3Support extends SupportHandler {
    @Override
    protected boolean canHandle(SupportTicket ticket) {
        return ticket.getPriority().equals("HIGH") || 
               ticket.getPriority().equals("CRITICAL");
    }
    
    @Override
    protected void process(SupportTicket ticket) {
        System.out.println("Level 3 handling: " + ticket.getIssue());
    }
}

// Usage
SupportHandler level1 = new Level1Support();
SupportHandler level2 = new Level2Support();
SupportHandler level3 = new Level3Support();

level1.setNext(level2);
level2.setNext(level3);

level1.handleTicket(new SupportTicket("Password reset", "LOW"));
level1.handleTicket(new SupportTicket("Software bug", "MEDIUM"));
level1.handleTicket(new SupportTicket("System down", "CRITICAL"));
```

### Q2: Logging System

```java
public abstract class Logger {
    public static final int INFO = 1;
    public static final int DEBUG = 2;
    public static final int ERROR = 3;
    
    protected int level;
    protected Logger next;
    
    public void setNext(Logger next) {
        this.next = next;
    }
    
    public void log(int level, String message) {
        if (this.level <= level) {
            write(message);
        }
        if (next != null) {
            next.log(level, message);
        }
    }
    
    protected abstract void write(String message);
}

public class ConsoleLogger extends Logger {
    public ConsoleLogger(int level) {
        this.level = level;
    }
    
    @Override
    protected void write(String message) {
        System.out.println("Console: " + message);
    }
}

public class FileLogger extends Logger {
    public FileLogger(int level) {
        this.level = level;
    }
    
    @Override
    protected void write(String message) {
        System.out.println("File: " + message);
    }
}

public class ErrorLogger extends Logger {
    public ErrorLogger(int level) {
        this.level = level;
    }
    
    @Override
    protected void write(String message) {
        System.out.println("Error: " + message);
    }
}

// Usage
Logger consoleLogger = new ConsoleLogger(Logger.INFO);
Logger fileLogger = new FileLogger(Logger.DEBUG);
Logger errorLogger = new ErrorLogger(Logger.ERROR);

consoleLogger.setNext(fileLogger);
fileLogger.setNext(errorLogger);

consoleLogger.log(Logger.INFO, "This is info");
consoleLogger.log(Logger.DEBUG, "This is debug");
consoleLogger.log(Logger.ERROR, "This is error");
```

### Q3: ATM Dispenser

```java
public abstract class CashDispenser {
    protected CashDispenser next;
    protected int denomination;
    
    public CashDispenser(int denomination) {
        this.denomination = denomination;
    }
    
    public void setNext(CashDispenser next) {
        this.next = next;
    }
    
    public void dispense(int amount) {
        if (amount >= denomination) {
            int count = amount / denomination;
            int remainder = amount % denomination;
            
            System.out.println("Dispensing " + count + " x $" + denomination);
            
            if (remainder != 0 && next != null) {
                next.dispense(remainder);
            }
        } else if (next != null) {
            next.dispense(amount);
        }
    }
}

public class Hundred extends CashDispenser {
    public Hundred() {
        super(100);
    }
}

public class Fifty extends CashDispenser {
    public Fifty() {
        super(50);
    }
}

public class Twenty extends CashDispenser {
    public Twenty() {
        super(20);
    }
}

public class Ten extends CashDispenser {
    public Ten() {
        super(10);
    }
}

// Usage
CashDispenser atm = new Hundred();
atm.setNext(new Fifty());
atm.getNext().setNext(new Twenty());
atm.getNext().getNext().setNext(new Ten());

atm.dispense(380);
// Output:
// Dispensing 3 x $100
// Dispensing 1 x $50
// Dispensing 1 x $20
// Dispensing 1 x $10
```

---

## 4. Interview Tricks & Pitfalls

### Pitfall 1: Breaking the Chain

```java
// WRONG: Not passing to next
public void handle(Request request) {
    if (canHandle(request)) {
        process(request);
    }
    // Forgot to pass to next!
}

// CORRECT: Always pass to next if can't handle
public void handle(Request request) {
    if (canHandle(request)) {
        process(request);
    } else if (next != null) {
        next.handle(request);
    }
}
```

### Pitfall 2: No End of Chain Handling

```java
// WRONG: No handling when chain ends
// CORRECT: Handle case when no handler can process
else {
    System.out.println("No handler available");
    // Or throw exception
}
```

### Pitfall 3: Handler Modifies Request

```java
// PROBLEM: Handler modifies request, affecting next handlers
// SOLUTION: Make request immutable or clone before passing
```

---

## 5. When to Use

✅ **Use when:**
- Multiple objects can handle request
- Handler not known in advance
- Want to decouple sender from receiver
- Want to add/remove handlers dynamically

❌ **Don't use when:**
- Only one handler
- Handler is known
- Request must be handled (no guarantee in chain)

---

## 6. Real-World Examples

- **Servlet Filters**: Request processing chain
- **Event Bubbling**: DOM event propagation
- **Logging Frameworks**: Multiple log handlers
- **Exception Handling**: Try-catch blocks

---

## 7. Quick Checklist

1. ☐ Define handler interface/abstract class
2. ☐ Handler has reference to next handler
3. ☐ Implement concrete handlers
4. ☐ Each handler decides: process or pass to next
5. ☐ Build chain by linking handlers
6. ☐ Handle case when no handler can process
7. ☐ Demonstrate request flowing through chain

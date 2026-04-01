## 1. What is Command Pattern?

Command pattern encapsulates a request as an object, allowing you to parameterize clients with different requests, queue requests, log requests, and support undoable operations.

```
Without Command:
  button.onClick() { service.doSomething(); }
  // Tight coupling, hard to undo/queue

With Command:
  button.setCommand(new DoSomethingCommand(service));
  button.click();  // Executes command
  // Can undo, queue, log commands
```

**Key principle:** Encapsulate requests as objects.

---

## 2. Implementation Guide

### Step 1: Define Command Interface

```java
public interface Command {
    void execute();
    void undo();  // Optional for undo support
}
```

### Step 2: Create Receiver (Does Actual Work)

```java
public class Light {
    private boolean isOn = false;
    
    public void turnOn() {
        isOn = true;
        System.out.println("Light is ON");
    }
    
    public void turnOff() {
        isOn = false;
        System.out.println("Light is OFF");
    }
    
    public boolean isOn() {
        return isOn;
    }
}
```

### Step 3: Implement Concrete Commands

```java
public class LightOnCommand implements Command {
    private Light light;
    
    public LightOnCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.turnOn();
    }
    
    @Override
    public void undo() {
        light.turnOff();
    }
}

public class LightOffCommand implements Command {
    private Light light;
    
    public LightOffCommand(Light light) {
        this.light = light;
    }
    
    @Override
    public void execute() {
        light.turnOff();
    }
    
    @Override
    public void undo() {
        light.turnOn();
    }
}
```

### Step 4: Create Invoker

```java
public class RemoteControl {
    private Command command;
    private Stack<Command> history = new Stack<>();
    
    public void setCommand(Command command) {
        this.command = command;
    }
    
    public void pressButton() {
        command.execute();
        history.push(command);
    }
    
    public void pressUndo() {
        if (!history.isEmpty()) {
            Command lastCommand = history.pop();
            lastCommand.undo();
        }
    }
}
```

### Step 5: Usage

```java
Light light = new Light();
Command lightOn = new LightOnCommand(light);
Command lightOff = new LightOffCommand(light);

RemoteControl remote = new RemoteControl();

remote.setCommand(lightOn);
remote.pressButton();  // Light ON

remote.setCommand(lightOff);
remote.pressButton();  // Light OFF

remote.pressUndo();    // Light ON (undo)
remote.pressUndo();    // Light OFF (undo)
```

---

## 3. Common Interview Questions

### Q1: Text Editor with Undo

```java
public interface Command {
    void execute();
    void undo();
}

public class TextEditor {
    private StringBuilder text = new StringBuilder();
    
    public void write(String text) {
        this.text.append(text);
        System.out.println("Text: " + this.text);
    }
    
    public void delete(int length) {
        int start = text.length() - length;
        if (start >= 0) {
            text.delete(start, text.length());
            System.out.println("Text: " + text);
        }
    }
    
    public String getText() {
        return text.toString();
    }
}

public class WriteCommand implements Command {
    private TextEditor editor;
    private String text;
    
    public WriteCommand(TextEditor editor, String text) {
        this.editor = editor;
        this.text = text;
    }
    
    @Override
    public void execute() {
        editor.write(text);
    }
    
    @Override
    public void undo() {
        editor.delete(text.length());
    }
}

public class EditorInvoker {
    private Stack<Command> history = new Stack<>();
    
    public void executeCommand(Command command) {
        command.execute();
        history.push(command);
    }
    
    public void undo() {
        if (!history.isEmpty()) {
            Command command = history.pop();
            command.undo();
        }
    }
}

// Usage
TextEditor editor = new TextEditor();
EditorInvoker invoker = new EditorInvoker();

invoker.executeCommand(new WriteCommand(editor, "Hello "));
invoker.executeCommand(new WriteCommand(editor, "World"));
invoker.undo();  // Removes "World"
invoker.undo();  // Removes "Hello "
```

### Q2: Task Queue System

```java
public interface Command {
    void execute();
}

public class EmailService {
    public void sendEmail(String to, String message) {
        System.out.println("Sending email to " + to + ": " + message);
    }
}

public class PaymentService {
    public void processPayment(String user, double amount) {
        System.out.println("Processing payment of $" + amount + " for " + user);
    }
}

public class SendEmailCommand implements Command {
    private EmailService emailService;
    private String to;
    private String message;
    
    public SendEmailCommand(EmailService service, String to, String message) {
        this.emailService = service;
        this.to = to;
        this.message = message;
    }
    
    @Override
    public void execute() {
        emailService.sendEmail(to, message);
    }
}

public class ProcessPaymentCommand implements Command {
    private PaymentService paymentService;
    private String user;
    private double amount;
    
    public ProcessPaymentCommand(PaymentService service, String user, double amount) {
        this.paymentService = service;
        this.user = user;
        this.amount = amount;
    }
    
    @Override
    public void execute() {
        paymentService.processPayment(user, amount);
    }
}

public class TaskQueue {
    private Queue<Command> queue = new LinkedList<>();
    
    public void addTask(Command command) {
        queue.add(command);
    }
    
    public void processTasks() {
        while (!queue.isEmpty()) {
            Command command = queue.poll();
            command.execute();
        }
    }
}

// Usage
EmailService emailService = new EmailService();
PaymentService paymentService = new PaymentService();

TaskQueue taskQueue = new TaskQueue();
taskQueue.addTask(new SendEmailCommand(emailService, "user@example.com", "Welcome!"));
taskQueue.addTask(new ProcessPaymentCommand(paymentService, "John", 100.0));
taskQueue.addTask(new SendEmailCommand(emailService, "admin@example.com", "Payment processed"));

taskQueue.processTasks();
```

### Q3: Smart Home System

```java
public interface Command {
    void execute();
    void undo();
}

public class TV {
    public void on() { System.out.println("TV is ON"); }
    public void off() { System.out.println("TV is OFF"); }
    public void setVolume(int level) { System.out.println("Volume: " + level); }
}

public class AC {
    public void on() { System.out.println("AC is ON"); }
    public void off() { System.out.println("AC is OFF"); }
    public void setTemperature(int temp) { System.out.println("Temperature: " + temp); }
}

public class TVOnCommand implements Command {
    private TV tv;
    
    public TVOnCommand(TV tv) { this.tv = tv; }
    
    @Override
    public void execute() { tv.on(); }
    
    @Override
    public void undo() { tv.off(); }
}

public class ACOnCommand implements Command {
    private AC ac;
    private int previousTemp;
    
    public ACOnCommand(AC ac) { this.ac = ac; }
    
    @Override
    public void execute() {
        ac.on();
        ac.setTemperature(24);
    }
    
    @Override
    public void undo() {
        ac.off();
    }
}

public class MacroCommand implements Command {
    private List<Command> commands;
    
    public MacroCommand(List<Command> commands) {
        this.commands = commands;
    }
    
    @Override
    public void execute() {
        for (Command command : commands) {
            command.execute();
        }
    }
    
    @Override
    public void undo() {
        for (int i = commands.size() - 1; i >= 0; i--) {
            commands.get(i).undo();
        }
    }
}

// Usage
TV tv = new TV();
AC ac = new AC();

List<Command> movieMode = Arrays.asList(
    new TVOnCommand(tv),
    new ACOnCommand(ac)
);

Command movieModeCommand = new MacroCommand(movieMode);
movieModeCommand.execute();  // Turn on TV and AC
movieModeCommand.undo();     // Turn off both
```

---

## 4. Interview Tricks & Pitfalls

### Pitfall 1: Command Holds Too Much State

```java
// WRONG: Command stores result
public class Command {
    private Result result;
    public void execute() {
        result = receiver.doSomething();
    }
}

// CORRECT: Command is stateless or minimal state
public class Command {
    public void execute() {
        receiver.doSomething();
    }
}
```

### Pitfall 2: Not Implementing Undo

```java
// If undo is required, must store previous state
public class SetVolumeCommand implements Command {
    private int previousVolume;
    private int newVolume;
    
    public void execute() {
        previousVolume = tv.getVolume();
        tv.setVolume(newVolume);
    }
    
    public void undo() {
        tv.setVolume(previousVolume);
    }
}
```

### Pitfall 3: Forgetting to Add to History

```java
// WRONG: Execute but don't track
public void execute(Command cmd) {
    cmd.execute();
    // Forgot to add to history!
}

// CORRECT: Track for undo
public void execute(Command cmd) {
    cmd.execute();
    history.push(cmd);
}
```

---

## 5. When to Use

✅ **Use when:**
- Need to queue operations
- Need undo/redo functionality
- Need to log operations
- Need to parameterize objects with operations
- Need macro commands (composite)

❌ **Don't use when:**
- Simple method calls suffice
- No need for undo/queue/log
- Adds unnecessary complexity

---

## 6. Real-World Examples

- **GUI Buttons**: Button actions as commands
- **Transaction Systems**: Database transactions
- **Job Schedulers**: Queued tasks
- **Macro Recording**: Record and replay actions
- **Thread Pools**: Runnable interface

---

## 7. Quick Checklist

1. ☐ Define Command interface with execute()
2. ☐ Create receiver classes (do actual work)
3. ☐ Implement concrete commands
4. ☐ Commands hold reference to receiver
5. ☐ Create invoker to execute commands
6. ☐ Add undo() if needed
7. ☐ Track command history for undo
8. ☐ Demonstrate execute and undo

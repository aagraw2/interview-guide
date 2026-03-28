# Command Pattern

## Intent
The Command pattern encapsulates requests as objects, allowing parameterization and queuing.

## When to Use
- Undo/redo
- Task queues
- Logging actions

## Example
```java
public interface Command{
    void execute();
}

public class EmailService {

    public void sendEmail(String user) {
        System.out.println("Sending email to " + user);
    }
}

public class ReportService {

    public void generateReport() {
        System.out.println("Generating report...");
    }
}

public class SendEmailCommand implements Command{
    private EmailService emailService;
    private String user;

    public SendEmailCommand(EmailService emailService, String user) {
        this.emailService = emailService;
        this.user = user;
    }

    @Override
    public void execute() {
        emailService.sendEmail(user);
    }
}

public class GenerateReportCommand implements Command {

    private ReportService reportService;

    public GenerateReportCommand(ReportService reportService) {
        this.reportService = reportService;
    }

    @Override
    public void execute() {
        reportService.generateReport();
    }
}

public class JobExecutor {

    private Queue<Command> queue = new LinkedList<>();

    public void submit(Command command) {
        queue.add(command);
    }

    public void run() {
        while (!queue.isEmpty()) {
            Command command = queue.poll();
            command.execute();
        }
    }
}

public class Main {

    public static void main(String[] args) {

        EmailService emailService = new EmailService();
        PaymentService paymentService = new PaymentService();
        ReportService reportService = new ReportService();

        JobExecutor executor = new JobExecutor();

        executor.submit(new SendEmailCommand(emailService, "Amit"));
        executor.submit(new ProcessPaymentCommand(paymentService, "Amit", 500));
        executor.submit(new GenerateReportCommand(reportService));

        executor.run();
    }
}
```
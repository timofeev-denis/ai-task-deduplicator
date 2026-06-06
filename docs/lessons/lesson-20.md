# Lesson 20: Configure SMTP

## What We're Building

Spring Mail configured to send emails via Mailpit (local development) and a real SMTP server
(production), with a base email service abstraction.

---

## Technologies

### Spring Mail (`JavaMailSender`)
Spring Boot auto-configures `JavaMailSender` when `spring-boot-starter-mail` is on the classpath
and SMTP connection properties are in `application.yml`. `JavaMailSender` is the interface you
inject into your services:

```java
@Service
public class EmailService {
    private final JavaMailSender mailSender;

    public EmailService(JavaMailSender mailSender) {
        this.mailSender = mailSender;
    }
}
```

### `MimeMessage` vs `SimpleMailMessage`
- `SimpleMailMessage`: plain text only. Good for simple notifications.
- `MimeMessage`: supports HTML, attachments, inline images. More powerful but more code.

For this project, HTML emails look professional and are worth the extra effort.

### Mailpit
Mailpit (configured in Lesson 02) acts as a fake SMTP server. It accepts all emails and
shows them in a web UI at `http://localhost:8025`. Key advantages for development:
- No real email addresses needed
- No risk of accidentally emailing real users
- View email HTML rendering in the browser
- Inspect raw email headers

---

## Step-by-Step

### Step 1: Add the Spring Mail dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

### Step 2: Configure SMTP in `application.yml`

Development profile (uses Mailpit):
```yaml
spring:
  mail:
    host: localhost
    port: 1025
    username: test
    password: test
    properties:
      mail:
        smtp:
          auth: false
          starttls:
            enable: false
```

Production profile (`application-prod.yml`):
```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: ${SMTP_USERNAME}
    password: ${SMTP_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
```

### Step 3: Use Spring profiles for environment switching

In `application.yml`, define the active profile:
```yaml
spring:
  profiles:
    active: dev
```

`application-dev.yml` → development settings (Mailpit)
`application-prod.yml` → production settings (real SMTP)

Switch profiles with:
```bash
java -jar app.jar --spring.profiles.active=prod
```

### Step 4: Create a base `NotificationService`

```java
@Service
public class NotificationService {

    private final JavaMailSender mailSender;

    @Value("${notification.from-email:noreply@deduplicator.local}")
    private String fromEmail;

    public NotificationService(JavaMailSender mailSender) {
        this.mailSender = mailSender;
    }

    protected void sendHtmlEmail(String to, String subject, String htmlBody) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
            helper.setFrom(fromEmail);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(htmlBody, true); // true = isHtml
            mailSender.send(message);
        } catch (MessagingException e) {
            throw new EmailException("Failed to send email to " + to, e);
        }
    }
}
```

### Step 5: Verify SMTP works with a smoke test

```java
@SpringBootTest
class SmtpConnectionTest {

    @Autowired JavaMailSender mailSender;

    @Test
    void canConnectToSmtp() {
        // Just verify the connection works — don't actually send
        assertThatCode(() -> {
            MimeMessage msg = mailSender.createMimeMessage();
            new MimeMessageHelper(msg).setTo("test@test.com");
        }).doesNotThrowAnyException();
    }
}
```

---

## Practical Tips

**Always check Mailpit after implementing email features.** Every time you add a new email
type, start the app, trigger the relevant event, and check `http://localhost:8025` to see
the email. Verify the HTML renders correctly, the subject is right, and links work.

**Use Spring profiles to manage environment differences.** Never put production SMTP credentials
in the main `application.yml`. Use `application-prod.yml` and environment variables.
The `spring.profiles.active` setting in the main YAML controls the default profile.

**Understand `MimeMessageHelper`.** It simplifies the verbose JavaMail API:
- `helper.setText(html, true)` — set HTML body
- `helper.addAttachment(name, file)` — add attachment
- `helper.setReplyTo(email)` — set reply-to address

**Never block on email sending.** Email sending can be slow (especially with real SMTP).
Use `@Async` or publish a domain event to send emails in a background thread,
so the main processing pipeline isn't delayed.

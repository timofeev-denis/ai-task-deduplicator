# Lesson 21: Send Duplicate Notification

## What We're Building

An HTML email notification sent to the issue author when one or more DUPLICATE issues are found,
containing issue numbers, similarity scores, and links.

---

## Technologies

### HTML Email Templates
HTML emails are regular HTML, but with severe constraints:
- No external CSS files (most email clients block them)
- No JavaScript
- Use inline CSS styles (`style="..."` on every element)
- Tables are the most reliable layout mechanism
- Test in multiple clients (Gmail, Outlook render differently)

For this project, a simple inline-styled HTML string is sufficient.
For complex templates, use Thymeleaf as a template engine.

### Thymeleaf (optional but recommended)
Thymeleaf is Spring's default template engine. You define templates in
`src/main/resources/templates/*.html` with Thymeleaf expressions:

```html
<p th:text="${issue.title}">Title placeholder</p>
<ul th:each="duplicate : ${duplicates}">
  <li th:text="${duplicate.issueUrl}"></li>
</ul>
```

Then render it in Java:
```java
Context ctx = new Context();
ctx.setVariable("issue", issue);
ctx.setVariable("duplicates", duplicateList);
String html = templateEngine.process("duplicate-notification", ctx);
```

---

## Step-by-Step

### Step 1: Add Thymeleaf dependency (optional)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

### Step 2: Create `DuplicateNotificationData`

```java
public record DuplicateNotificationData(
    long sourceIssueIid,
    String sourceIssueTitle,
    String sourceIssueUrl,
    String authorName,
    List<DuplicateEntry> duplicates
) {}

public record DuplicateEntry(
    long issueIid,
    String title,
    String issueUrl,
    double confidence,
    RelationType relationType
) {}
```

### Step 3: Create the email template

`src/main/resources/templates/emails/duplicate-notification.html`:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><meta charset="UTF-8"/></head>
<body style="font-family: Arial, sans-serif; color: #333;">

  <h2 style="color: #c0392b;">Похожие задачи обнаружены</h2>

  <p>Здравствуйте, <span th:text="${data.authorName}">User</span>!</p>

  <p>Для вашей задачи
    <a th:href="${data.sourceIssueUrl}" th:text="'#' + ${data.sourceIssueIid}">
      #123
    </a>
    (<span th:text="${data.sourceIssueTitle}">Title</span>)
    обнаружены похожие Issues:
  </p>

  <table style="border-collapse: collapse; width: 100%;">
    <thead>
      <tr style="background: #f2f2f2;">
        <th style="padding: 8px; text-align: left;">Issue</th>
        <th style="padding: 8px; text-align: left;">Тип</th>
        <th style="padding: 8px; text-align: left;">Уверенность</th>
      </tr>
    </thead>
    <tbody>
      <tr th:each="dup : ${data.duplicates}">
        <td style="padding: 8px; border-bottom: 1px solid #ddd;">
          <a th:href="${dup.issueUrl}" th:text="'#' + ${dup.issueIid} + ': ' + ${dup.title}">
            #42: Issue title
          </a>
        </td>
        <td style="padding: 8px; border-bottom: 1px solid #ddd;" th:text="${dup.relationType}">
          DUPLICATE
        </td>
        <td style="padding: 8px; border-bottom: 1px solid #ddd;"
            th:text="${#numbers.formatPercent(dup.confidence, 0, 0)}">
          91%
        </td>
      </tr>
    </tbody>
  </table>

  <p style="color: #888; font-size: 12px; margin-top: 24px;">
    Это автоматическое уведомление от AI Issue Deduplicator.
  </p>
</body>
</html>
```

### Step 4: Create `DuplicateNotificationService`

```java
@Service
public class DuplicateNotificationService extends NotificationService {

    private final TemplateEngine templateEngine;
    private final GitLabProperties gitLabProperties;

    public DuplicateNotificationService(JavaMailSender mailSender,
                                        TemplateEngine templateEngine,
                                        GitLabProperties gitLabProperties) {
        super(mailSender);
        this.templateEngine = templateEngine;
        this.gitLabProperties = gitLabProperties;
    }

    public void sendDuplicateNotification(Issue sourceIssue,
                                          List<IssueClassification> classifications,
                                          String authorEmail) {
        List<DuplicateEntry> entries = classifications.stream()
            .map(c -> new DuplicateEntry(
                c.candidate().issue().getGitlabIssueIid(),
                c.candidate().issue().getTitle(),
                buildIssueUrl(c.candidate().issue()),
                c.classification().confidence(),
                c.classification().classification()
            ))
            .toList();

        var data = new DuplicateNotificationData(
            sourceIssue.issueIid(),
            sourceIssue.title(),
            buildIssueUrl(sourceIssue),
            sourceIssue.authorName(),
            entries
        );

        Context ctx = new Context();
        ctx.setVariable("data", data);
        String html = templateEngine.process("emails/duplicate-notification", ctx);

        sendHtmlEmail(authorEmail, "Обнаружены похожие задачи: #" + sourceIssue.issueIid(), html);
    }

    private String buildIssueUrl(ProcessedIssue issue) {
        return gitLabProperties.baseUrl() + "/-/issues/" + issue.getGitlabIssueIid();
    }
}
```

---

## Practical Tips

**Test the HTML in a browser first.** Open the template file directly in Chrome before
integrating it with Thymeleaf. This gives you instant visual feedback without starting the app.

**Use Mailpit to verify rendering.** After integration, trigger the notification and open
Mailpit at `http://localhost:8025`. It shows the rendered HTML exactly as the email client would.

**Keep emails simple.** Elaborate HTML emails break in Outlook. Stick to tables, inline styles,
and basic formatting. Less is more.

**Send emails asynchronously.** SMTP calls can take 1-3 seconds. Mark the method `@Async`
or fire a domain event for email sending — don't block the main processing thread.

**Handle missing author email gracefully.** GitLab issues don't always have an author email
available via the API (privacy settings). Check for null before sending:
```java
if (authorEmail == null || authorEmail.isBlank()) {
    log.warn("No email for issue #{} author — skipping notification", issueIid);
    return;
}
```

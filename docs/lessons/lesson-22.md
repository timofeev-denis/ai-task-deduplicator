# Lesson 22: Send Related Notification

## What We're Building

An email notification for RELATED issues — similar to the duplicate notification but with
different messaging and without the `duplicate-detected` label mention.

---

## Technologies

This lesson reuses everything from Lesson 21. The focus is on:
- Applying the DRY principle by extracting shared email logic
- Understanding when to share code vs duplicate it
- Template reuse with Thymeleaf

---

## Step-by-Step

### Step 1: Evaluate what's different from the duplicate notification

| Aspect | Duplicate notification | Related notification |
|--------|----------------------|---------------------|
| Subject | "Дублирующиеся задачи обнаружены" | "Связанные задачи обнаружены" |
| Heading | Red, urgent tone | Neutral, informational |
| Content | Links list, confidence | Links list, confidence |
| Label mention | "Добавлена метка duplicate-detected" | Not mentioned |
| Emoji | 🔴 or ⚠️ | 🔗 or ℹ️ |

The content is ~80% identical. This is a good case for a shared base template with a variable header.

### Step 2: Refactor the template for reuse

Instead of two separate templates, create one parameterized template:

`src/main/resources/templates/emails/issue-notification.html`:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<body style="font-family: Arial, sans-serif; color: #333;">

  <h2 th:style="${isDuplicate} ? 'color: #c0392b;' : 'color: #2980b9;'"
      th:text="${isDuplicate} ? 'Дублирующиеся задачи обнаружены' : 'Связанные задачи обнаружены'">
    Heading
  </h2>

  <p>Здравствуйте, <span th:text="${data.authorName}">User</span>!</p>

  <p>Для задачи
    <a th:href="${data.sourceIssueUrl}" th:text="'#' + ${data.sourceIssueIid}">
      #123
    </a>
    обнаружены
    <span th:text="${isDuplicate} ? 'дубликаты' : 'связанные задачи'">связи</span>:
  </p>

  <!-- same table as in Lesson 21 -->

  <p th:if="${isDuplicate}" style="background: #ffeeba; padding: 8px; border-radius: 4px;">
    Метка <strong>duplicate-detected</strong> добавлена к задаче.
  </p>
</body>
</html>
```

### Step 3: Create `RelatedNotificationService`

```java
@Service
public class RelatedNotificationService extends NotificationService {

    private final TemplateEngine templateEngine;

    public RelatedNotificationService(JavaMailSender mailSender,
                                      TemplateEngine templateEngine) {
        super(mailSender);
        this.templateEngine = templateEngine;
    }

    public void sendRelatedNotification(Issue sourceIssue,
                                        List<IssueClassification> relatedClassifications,
                                        String authorEmail) {
        if (authorEmail == null || authorEmail.isBlank()) {
            log.warn("No email for issue #{} author — skipping notification", sourceIssue.issueIid());
            return;
        }

        var data = buildNotificationData(sourceIssue, relatedClassifications);

        Context ctx = new Context();
        ctx.setVariable("data", data);
        ctx.setVariable("isDuplicate", false);

        String html = templateEngine.process("emails/issue-notification", ctx);
        sendHtmlEmail(authorEmail,
            "Обнаружены связанные задачи: #" + sourceIssue.issueIid(),
            html);
    }
}
```

### Step 4: Refactor `DuplicateNotificationService` to use the shared template

```java
ctx.setVariable("isDuplicate", true);
String html = templateEngine.process("emails/issue-notification", ctx);
```

### Step 5: Combine notification sending in the orchestrator

```java
List<IssueClassification> duplicates = results.stream()
    .filter(r -> r.classification().classification() == RelationType.DUPLICATE)
    .toList();

List<IssueClassification> related = results.stream()
    .filter(r -> r.classification().classification() == RelationType.RELATED)
    .toList();

if (!duplicates.isEmpty()) {
    duplicateNotificationService.sendDuplicateNotification(issue, duplicates, issue.authorEmail());
}

if (!related.isEmpty()) {
    relatedNotificationService.sendRelatedNotification(issue, related, issue.authorEmail());
}
```

---

## Practical Tips

**Don't send two separate emails if both duplicates and related issues are found.**
Consider sending one combined email listing all findings. This is better UX — one email
instead of two arriving seconds apart. Use the `isDuplicate` flag in the template to
control the heading, and pass a combined list of all findings.

**Understand when to share code and when not to.** The notifications are 80% similar —
sharing the template is justified. But if the content diverges significantly in the future,
two separate templates will be easier to maintain than one heavily conditional shared template.

**Learn Thymeleaf's conditional and iteration syntax.** Key expressions:
- `th:if="${condition}"` — show/hide element
- `th:each="item : ${list}"` — iterate over a list
- `th:text="${variable}"` — set text content
- `th:style="${cond} ? 'a' : 'b'"` — conditional style

**Test both notification types with Mailpit.** Create test scenarios:
1. An issue that only has DUPLICATE findings
2. An issue that only has RELATED findings
3. An issue that has both

Verify in Mailpit that each case produces the correct email content.

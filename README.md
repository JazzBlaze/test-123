Below is a complete step‐by‐step implementation of the `/createAlert` endpoint based on your specifications. In this example, we use a DTO (placed in your existing `beans` folder), a custom class-level validator for the reminderDays rule, an entity for database mapping, a repository, a service layer that computes the reminder date, and a controller to expose the REST endpoint.

For clarity, we assume your package structure is similar to:

```
com.dbs.capi.sg.encryption.utilities
 ├── beans             // for DTOs (your request classes)
 ├── controller        // REST controllers
 ├── entity            // JPA entities
 ├── repository        // Spring Data JPA repositories
 ├── service           // Service interfaces
 ├── serviceImpl       // Service implementations
 └── validation        // Custom validation annotations and validators
```

---

## 1. DTO with Validations

**File:** `AlertRequest.java` (in `beans` package)

This DTO maps the incoming JSON and uses standard Bean Validation annotations plus a custom validator (explained next) for the `reminderDays` constraint.

```java
package com.dbs.capi.sg.encryption.utilities.beans;

import com.dbs.capi.sg.encryption.utilities.validation.ValidReminderDays;
import jakarta.validation.constraints.*;
import lombok.Getter;
import lombok.Setter;

import java.time.Instant;
import java.util.List;

@Getter
@Setter
@ValidReminderDays  // Custom validator to ensure reminderDays is <= (expiryDate - now)
public class AlertRequest {

    @NotBlank(message = "Project name is required")
    @Size(max = 100, message = "Project name must not exceed 100 characters")
    private String projectName;

    @NotBlank(message = "Project description is required")
    @Size(max = 300, message = "Project description must not exceed 300 characters")
    private String projectDesc;

    @NotBlank(message = "Owner name is required")
    @Size(max = 100, message = "Owner name must not exceed 100 characters")
    private String ownerName;

    @NotNull(message = "Expiry date is required")
    @Future(message = "Expiry date must be in the future")
    private Instant expiryDate;

    @Min(value = 1, message = "Reminder days must be at least 1")
    private int reminderDays;

    @NotEmpty(message = "At least one notification email is required")
    private List<@Email(message = "Invalid email format") String> notificationEmails;
}
```

*Note:*  
• The expiry date is defined as an `Instant` (which maps to ISO date strings).  
• The list of emails is validated so that each email in the list is a valid email.

---

## 2. Custom Class-Level Validator for `reminderDays`

Since the rule for `reminderDays` must compare it with the number of days between now and the expiry date, we implement a custom annotation and its validator.

### 2.1 Create the Annotation

**File:** `ValidReminderDays.java` (in `validation` package)

```java
package com.dbs.capi.sg.encryption.utilities.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = ValidReminderDaysValidator.class)
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidReminderDays {
    String message() default "Reminder days must be less than or equal to the number of days between now and expiry date";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

### 2.2 Create the Validator

**File:** `ValidReminderDaysValidator.java` (in the same `validation` package)

```java
package com.dbs.capi.sg.encryption.utilities.validation;

import com.dbs.capi.sg.encryption.utilities.beans.AlertRequest;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

public class ValidReminderDaysValidator implements ConstraintValidator<ValidReminderDays, AlertRequest> {

    @Override
    public boolean isValid(AlertRequest alertRequest, ConstraintValidatorContext context) {
        if (alertRequest == null) {
            return true; // Other validations (e.g., @NotNull) should handle nulls.
        }
        Instant expiryDate = alertRequest.getExpiryDate();
        int reminderDays = alertRequest.getReminderDays();
        Instant now = Instant.now();

        // Ensure expiryDate is in the future (already handled by @Future) and non-null
        if (expiryDate == null || expiryDate.isBefore(now)) {
            return false;
        }
        long daysDifference = ChronoUnit.DAYS.between(now, expiryDate);
        return reminderDays <= daysDifference;
    }
}
```

---

## 3. Database Entity

**File:** `AlertEntity.java` (in `entity` package)

This entity maps to your MariaDB table. We use an `@ElementCollection` for the list of notification emails so that JPA stores them in a separate table.

```java
package com.dbs.capi.sg.encryption.utilities.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

import java.time.Instant;
import java.util.List;

@Getter
@Setter
@Entity
@Table(name = "alerts")
public class AlertEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long alertId;

    @Column(name = "project_name", nullable = false, length = 100)
    private String projectName;

    @Column(name = "project_desc", nullable = false, length = 300)
    private String projectDesc;

    @Column(name = "owner_name", nullable = false, length = 100)
    private String ownerName;

    @Column(name = "expiry_date", nullable = false)
    private Instant expiryDate;

    @Column(name = "reminder_date", nullable = false)
    private Instant reminderDate;

    @ElementCollection
    @CollectionTable(name = "alert_notification_emails", joinColumns = @JoinColumn(name = "alert_id"))
    @Column(name = "notification_email", nullable = false)
    private List<String> notificationEmails;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;
}
```

---

## 4. Repository Interface

**File:** `AlertRepository.java` (in `repository` package)

This interface extends Spring Data JPA’s `JpaRepository` for standard CRUD operations.

```java
package com.dbs.capi.sg.encryption.utilities.repository;

import com.dbs.capi.sg.encryption.utilities.entity.AlertEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface AlertRepository extends JpaRepository<AlertEntity, Long> {
    // Custom query methods can be added here if needed.
}
```

---

## 5. Service Layer

### 5.1 Service Interface

**File:** `AlertService.java` (in `service` package)

```java
package com.dbs.capi.sg.encryption.utilities.service;

import com.dbs.capi.sg.encryption.utilities.beans.AlertRequest;
import com.dbs.capi.sg.encryption.utilities.entity.AlertEntity;

public interface AlertService {
    AlertEntity createAlert(AlertRequest alertRequest);
}
```

### 5.2 Service Implementation

**File:** `AlertServiceImpl.java` (in `serviceImpl` package)

In this implementation, we compute the reminder date by subtracting the reminderDays (in days) from the expiryDate. We then set the createdAt timestamp and persist the record.

```java
package com.dbs.capi.sg.encryption.utilities.serviceImpl;

import com.dbs.capi.sg.encryption.utilities.beans.AlertRequest;
import com.dbs.capi.sg.encryption.utilities.entity.AlertEntity;
import com.dbs.capi.sg.encryption.utilities.repository.AlertRepository;
import com.dbs.capi.sg.encryption.utilities.service.AlertService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

@Service
public class AlertServiceImpl implements AlertService {

    private final AlertRepository alertRepository;

    @Autowired
    public AlertServiceImpl(AlertRepository alertRepository) {
        this.alertRepository = alertRepository;
    }

    @Override
    public AlertEntity createAlert(AlertRequest alertRequest) {
        AlertEntity entity = new AlertEntity();
        entity.setProjectName(alertRequest.getProjectName());
        entity.setProjectDesc(alertRequest.getProjectDesc());
        entity.setOwnerName(alertRequest.getOwnerName());
        entity.setExpiryDate(alertRequest.getExpiryDate());

        // Compute reminderDate: expiryDate minus reminderDays days
        Instant reminderDate = alertRequest.getExpiryDate().minus(alertRequest.getReminderDays(), ChronoUnit.DAYS);
        entity.setReminderDate(reminderDate);

        // Set the list of notification emails
        entity.setNotificationEmails(alertRequest.getNotificationEmails());

        // Set the created timestamp
        entity.setCreatedAt(Instant.now());

        return alertRepository.save(entity);
    }
}
```

---

## 6. REST Controller

**File:** `AlertController.java` (in `controller` package)

This controller defines the `/createAlert` endpoint. It receives the JSON payload, validates it, and then calls the service layer.

```java
package com.dbs.capi.sg.encryption.utilities.controller;

import com.dbs.capi.sg.encryption.utilities.beans.AlertRequest;
import com.dbs.capi.sg.encryption.utilities.entity.AlertEntity;
import com.dbs.capi.sg.encryption.utilities.service.AlertService;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/alerts")
@Validated
public class AlertController {

    private final AlertService alertService;

    @Autowired
    public AlertController(AlertService alertService) {
        this.alertService = alertService;
    }

    // POST endpoint: /api/alerts/createAlert
    @PostMapping("/createAlert")
    public ResponseEntity<AlertEntity> createAlert(@Valid @RequestBody AlertRequest alertRequest) {
        AlertEntity savedAlert = alertService.createAlert(alertRequest);
        return ResponseEntity.ok(savedAlert);
    }
}
```

---

## 7. Configuration Reminder

Ensure your `application.properties` (or `application.yml`) is correctly set up for your local MariaDB. For example:

```properties
spring.datasource.url=jdbc:mariadb://localhost:3306/yourdbname
spring.datasource.username=yourusername
spring.datasource.password=yourpassword

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

---

### Summary

1. **DTO (AlertRequest.java):** Maps the incoming JSON and applies validations (including a custom class-level validator for reminderDays).
2. **Custom Validator:** Checks that the number of reminder days is no more than the days between the current time and the expiry date.
3. **Entity (AlertEntity.java):** Represents the database table with all required fields.
4. **Repository (AlertRepository.java):** Provides basic CRUD operations.
5. **Service Layer:** Computes the reminderDate and saves the alert record.
6. **Controller (AlertController.java):** Exposes the `/createAlert` endpoint to accept POST requests from the frontend.

This detailed example follows widely accepted Spring Boot practices and should give you a solid foundation for your enterprise-level project. Feel free to ask if you need further clarification or additional details!

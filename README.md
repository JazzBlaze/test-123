Below is an updated version of the previous solution that combines the LOB and App Code validations into one custom validator (@ValidLobAndAppCode) and adds a cacheable repository method to quickly retrieve the LobMap record.

---

## 1. Updated AlertRequest DTO

Instead of having separate validators for LOB and App Code, we now annotate the DTO at the class level with our new @ValidLobAndAppCode validator. (Other field validations remain the same.)

**File:** `AlertRequest.java` (in your dto or beans package)

```java
package com.dbs.capi.sg.encryption.utilities.dto;

import com.dbs.capi.sg.encryption.utilities.validation.alertValidation.ValidDbsEmail;
import com.dbs.capi.sg.encryption.utilities.validation.alertValidation.ValidDbsListEmail;
import com.dbs.capi.sg.encryption.utilities.validation.alertValidation.ValidReminderDays;
import com.dbs.capi.sg.encryption.utilities.validation.alertValidation.ValidLobAndAppCode;
import jakarta.validation.constraints.*;
import lombok.Getter;
import lombok.Setter;

import java.time.Instant;
import java.util.List;

@Getter
@Setter
@ValidReminderDays // validates reminderDays vs expiryDate
@ValidLobAndAppCode // combined validation for lob and appCode
public class AlertRequest {

    @NotBlank(message = "Project Name is required")
    @Size(max = 100, message = "Project name must not exceed 100 characters")
    private String projectName;

    @NotBlank(message = "Project Description is required")
    @Size(max = 300, message = "Project Description must not exceed 300 characters")
    private String projectDesc;

    @NotBlank(message = "Owner Name is required")
    @Size(max = 100, message = "Owner name must not exceed 100 characters")
    private String ownerName;

    @NotBlank(message = "Line of Business (LOB) is required")
    private String lob;

    @NotBlank(message = "App Code is required")
    private String appCode;

    @NotNull(message = "Expiry Date is required")
    @Future(message = "Expiry date must be in the future")
    private Instant expiryDate;

    @NotNull(message = "Reminder days is required")
    @Min(value = 1, message = "Reminder days must be at least 1")
    private int reminderDays;

    @NotBlank(message = "Primary Contact is required")
    @ValidDbsEmail
    private String primaryContact;

    @NotEmpty(message = "At least one notification email is required")
    @ValidDbsListEmail
    private List<String> notificationEmails;
}
```

---

## 2. Combined Custom Validator for LOB and App Code

### 2.1 Create the Annotation

**File:** `ValidLobAndAppCode.java` (in your validation package)

```java
package com.dbs.capi.sg.encryption.utilities.validation.alertValidation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = ValidLobAndAppCodeValidator.class)
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidLobAndAppCode {
    String message() default "Invalid LOB and App Code combination";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

### 2.2 Create the Validator Implementation

**File:** `ValidLobAndAppCodeValidator.java` (in your validation package)

```java
package com.dbs.capi.sg.encryption.utilities.validation.alertValidation;

import com.dbs.capi.sg.encryption.utilities.dto.AlertRequest;
import com.dbs.capi.sg.encryption.utilities.entity.LobMap;
import com.dbs.capi.sg.encryption.utilities.repository.LobMapRepository;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class ValidLobAndAppCodeValidator implements ConstraintValidator<ValidLobAndAppCode, AlertRequest> {

    @Autowired
    private LobMapRepository lobMapRepository;

    @Override
    public boolean isValid(AlertRequest alertRequest, ConstraintValidatorContext context) {
        if (alertRequest == null) {
            return true; // handled by other validations
        }
        String lob = alertRequest.getLob();
        String appCode = alertRequest.getAppCode();

        // Let @NotBlank handle missing values.
        if (lob == null || lob.trim().isEmpty() || appCode == null || appCode.trim().isEmpty()) {
            return true;
        }

        // Use the repository's cacheable method to find the LobMap.
        LobMap lobMap = lobMapRepository.findByLobAndAppCode(lob, appCode);
        return lobMap != null;
    }
}
```

---

## 3. Update the LobMap Repository with a Cacheable Method

In your repository for LobMap, add a method that is cacheable for fast retrieval. This method will be used both by the validator and the service layer.

**File:** `LobMapRepository.java` (in your repository package)

```java
package com.dbs.capi.sg.encryption.utilities.repository;

import com.dbs.capi.sg.encryption.utilities.entity.LobMap;
import com.dbs.capi.sg.encryption.utilities.entity.LobMapId;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface LobMapRepository extends JpaRepository<LobMap, LobMapId> {

    // This method uses caching for faster retrieval.
    @Cacheable(value = "lobMapCache")
    LobMap findByLobAndAppCode(String lob, String appCode);
    
    // You may retain or remove any previously defined methods.
}
```

---

## 4. Using the Cached LobMap in the Service Layer

In your AlertServiceImpl, use the new cacheable method to retrieve the LobMap record when saving an Alert.

**File:** `AlertServiceImpl.java` (in your serviceImpl package)

```java
package com.dbs.capi.sg.encryption.utilities.serviceImpl;

import com.dbs.capi.sg.encryption.utilities.dto.AlertRequest;
import com.dbs.capi.sg.encryption.utilities.entity.Alert;
import com.dbs.capi.sg.encryption.utilities.entity.LobMap;
import com.dbs.capi.sg.encryption.utilities.repository.AlertRepository;
import com.dbs.capi.sg.encryption.utilities.repository.LobMapRepository;
import com.dbs.capi.sg.encryption.utilities.service.AlertService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

@Service
public class AlertServiceImpl implements AlertService {

    private final AlertRepository alertRepository;
    private final LobMapRepository lobMapRepository;

    @Autowired
    public AlertServiceImpl(AlertRepository alertRepository, LobMapRepository lobMapRepository) {
        this.alertRepository = alertRepository;
        this.lobMapRepository = lobMapRepository;
    }

    @Override
    public Alert createAlert(AlertRequest alertRequest) {
        Alert alert = new Alert();
        alert.setProjectName(alertRequest.getProjectName());
        alert.setProjectDesc(alertRequest.getProjectDesc());
        alert.setOwnerName(alertRequest.getOwnerName());

        // Retrieve LobMap via the cacheable repository method.
        LobMap lobMap = lobMapRepository.findByLobAndAppCode(alertRequest.getLob(), alertRequest.getAppCode());
        if (lobMap == null) {
            throw new RuntimeException("Invalid LOB and App Code combination");
        }
        alert.setLobMap(lobMap);

        alert.setExpiryDate(alertRequest.getExpiryDate());
        Instant reminderDate = alertRequest.getExpiryDate().minus(alertRequest.getReminderDays(), ChronoUnit.DAYS);
        alert.setReminderDate(reminderDate);

        alert.setPrimaryContact(alertRequest.getPrimaryContact());
        alert.setNotificationEmails(alertRequest.getNotificationEmails());
        alert.setCreatedAt(Instant.now());

        return alertRepository.save(alert);
    }
}
```

---

## 5. Summary of Changes

- **DTO Update:**  
  - Added new fields (`lob`, `appCode`, `primaryContact`) and retained existing validations.
  - Added a class-level `@ValidLobAndAppCode` annotation to ensure the combination is valid.

- **Custom Validator:**  
  - `@ValidLobAndAppCode` is defined and implemented to validate the (lob, appCode) combination by using a cacheable repository method.

- **Repository Update:**  
  - In `LobMapRepository`, added `findByLobAndAppCode` annotated with `@Cacheable` so that it caches lookups.

- **Service Layer:**  
  - Uses the cacheable repository method for faster retrieval of LobMap when creating an Alert.

This approach simplifies validation by combining the two checks into one and leverages Spring Boot's caching support for better performance across multiple layers. Let me know if you need any further clarifications or adjustments!
--------------------------------------------------------------------------------------------------------------------



appendix
*******************

Below is a complete, step‐by‐step example that meets your updated specifications. It covers the DTO (with your custom validations and new fields), the custom validators for “lob” and “appCode” (which check against your lob_mapping table), the two JPA entities (Alert and LobMap with a composite key), the service layer (which computes the reminder date and saves the records), and the REST controllers (including one to add new lob mappings).

For brevity, we assume you already have your other custom validations (ValidDbsEmail, ValidDbsListEmail, ValidReminderDays) implemented. You can skip those implementations.

---

## 1. DTO: AlertRequest

**File:** `AlertRequest.java` (in your existing `dto` or `beans` folder)

This DTO maps the JSON payload from the frontend:

```java
package com.dbs.capi.sg.encryption.utilities.dto;

import com.dbs.capi.sg.encryption.utilities.validation.alertValidation.ValidDbsEmail;
import com.dbs.capi.sg.encryption.utilities.validation.alertValidation.ValidDbsListEmail;
import com.dbs.capi.sg.encryption.utilities.validation.alertValidation.ValidReminderDays;
import com.dbs.capi.sg.encryption.utilities.validation.alertValidation.ValidLob;
import com.dbs.capi.sg.encryption.utilities.validation.alertValidation.ValidAppCode;
import jakarta.validation.constraints.*;
import lombok.Getter;
import lombok.Setter;

import java.time.Instant;
import java.util.List;

@Getter
@Setter
@ValidReminderDays // custom validation to ensure reminderDays is within valid bounds relative to expiryDate
public class AlertRequest {

    @NotBlank(message = "Project Name is required")
    @Size(max = 100, message = "Project name must not exceed 100 characters")
    private String projectName;

    @NotBlank(message = "Project Description is required")
    @Size(max = 300, message = "Project Description must not exceed 300 characters")
    private String projectDesc;

    @NotBlank(message = "Owner Name is required")
    @Size(max = 100, message = "Owner name must not exceed 100 characters")
    private String ownerName;

    @NotBlank(message = "Line of Business (LOB) is required")
    @ValidLob // custom validator that checks if the provided lob exists in the lob_mapping table
    private String lob;

    @NotBlank(message = "App Code is required")
    @ValidAppCode // custom validator that checks if the given appCode is valid for the provided lob (using lob_mapping)
    private String appCode;

    @NotNull(message = "Expiry Date is required")
    @Future(message = "Expiry date must be in the future")
    private Instant expiryDate;

    @NotNull(message = "Reminder days is required")
    @Min(value = 1, message = "Reminder days must be at least 1")
    private int reminderDays;

    @NotBlank(message = "Primary Contact is required")
    @ValidDbsEmail
    private String primaryContact;

    @NotEmpty(message = "At least one notification email is required")
    @ValidDbsListEmail
    private List<String> notificationEmails;
}
```

---

## 2. Custom Validators for LOB and App Code

### 2.1 Validator for LOB

**Annotation:** `ValidLob.java`

```java
package com.dbs.capi.sg.encryption.utilities.validation.alertValidation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = ValidLobValidator.class)
@Target({ ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidLob {
    String message() default "Invalid Line of Business (LOB)";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

**Validator Implementation:** `ValidLobValidator.java`

```java
package com.dbs.capi.sg.encryption.utilities.validation.alertValidation;

import com.dbs.capi.sg.encryption.utilities.repository.LobMapRepository;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class ValidLobValidator implements ConstraintValidator<ValidLob, String> {

    @Autowired
    private LobMapRepository lobMapRepository;

    @Override
    public boolean isValid(String lob, ConstraintValidatorContext context) {
        if(lob == null || lob.trim().isEmpty()) {
            return true; // @NotBlank will catch this case.
        }
        // Check if any LobMap record exists with the given lob.
        return lobMapRepository.existsByLobMapIdLob(lob);
    }
}
```

### 2.2 Validator for App Code

**Annotation:** `ValidAppCode.java`

```java
package com.dbs.capi.sg.encryption.utilities.validation.alertValidation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = ValidAppCodeValidator.class)
@Target({ ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidAppCode {
    String message() default "Invalid App Code for the given LOB";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

**Validator Implementation:** `ValidAppCodeValidator.java`

```java
package com.dbs.capi.sg.encryption.utilities.validation.alertValidation;

import com.dbs.capi.sg.encryption.utilities.dto.AlertRequest;
import com.dbs.capi.sg.encryption.utilities.repository.LobMapRepository;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class ValidAppCodeValidator implements ConstraintValidator<ValidAppCode, String> {

    @Autowired
    private LobMapRepository lobMapRepository;

    @Override
    public boolean isValid(String appCode, ConstraintValidatorContext context) {
        if(appCode == null || appCode.trim().isEmpty()) {
            return true; // Handled by @NotBlank.
        }
        // Retrieve the root bean to access the lob field.
        Object rootBean = context.getRootBean();
        if (rootBean instanceof AlertRequest) {
            AlertRequest alertRequest = (AlertRequest) rootBean;
            String lob = alertRequest.getLob();
            if (lob == null || lob.trim().isEmpty()) {
                // Without a valid lob, appCode cannot be verified.
                return false;
            }
            // Validate that the combination (lob, appCode) exists.
            return lobMapRepository.existsByLobMapIdLobAndLobMapIdAppCode(lob, appCode);
        }
        return false;
    }
}
```

---

## 3. Backend Entities

### 3.1 Alert Entity

**File:** `Alert.java` (in the `entity` package)

This entity maps to the “alerts” table. It uses a many-to-one relationship to reference LobMap.

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
public class Alert {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long alertId;

    @Column(name = "project_name", nullable = false, length = 100)
    private String projectName;

    @Column(name = "project_desc", nullable = false, length = 300)
    private String projectDesc;

    @Column(name = "owner_name", nullable = false, length = 100)
    private String ownerName;

    // Many-to-One relationship using composite key (lob, appCode)
    @ManyToOne(optional = false)
    @JoinColumns({
        @JoinColumn(name = "lob", referencedColumnName = "lob"),
        @JoinColumn(name = "app_code", referencedColumnName = "appCode")
    })
    private LobMap lobMap;

    @Column(name = "expiry_date", nullable = false)
    private Instant expiryDate;

    @Column(name = "reminder_date", nullable = false)
    private Instant reminderDate;

    @Column(name = "primary_contact", nullable = false)
    private String primaryContact;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @ElementCollection
    @CollectionTable(name = "alert_notification_emails", joinColumns = @JoinColumn(name = "alert_id"))
    @Column(name = "notification_email", nullable = false)
    private List<String> notificationEmails;
}
```

### 3.2 LobMap Entity with Composite Key

We first define an embeddable key class.

**File:** `LobMapId.java` (in the `entity` package)

```java
package com.dbs.capi.sg.encryption.utilities.entity;

import jakarta.persistence.Embeddable;
import lombok.Getter;
import lombok.Setter;

import java.io.Serializable;
import java.util.Objects;

@Getter
@Setter
@Embeddable
public class LobMapId implements Serializable {
    // Optionally include a generated id if required by your design:
    private Long id;
    private String lob;
    private String appCode;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof LobMapId)) return false;
        LobMapId that = (LobMapId) o;
        return Objects.equals(id, that.id) &&
               Objects.equals(lob, that.lob) &&
               Objects.equals(appCode, that.appCode);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, lob, appCode);
    }
}
```

Now define the LobMap entity.

**File:** `LobMap.java` (in the `entity` package)

```java
package com.dbs.capi.sg.encryption.utilities.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
@Entity
@Table(name = "lob_mapping")
public class LobMap {

    @EmbeddedId
    private LobMapId lobMapId;

    // Additional fields can be added here if needed.
}
```

---

## 4. Repositories

### 4.1 Alert Repository

**File:** `AlertRepository.java` (in the `repository` package)

```java
package com.dbs.capi.sg.encryption.utilities.repository;

import com.dbs.capi.sg.encryption.utilities.entity.Alert;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface AlertRepository extends JpaRepository<Alert, Long> {
}
```

### 4.2 LobMap Repository

**File:** `LobMapRepository.java` (in the `repository` package)

```java
package com.dbs.capi.sg.encryption.utilities.repository;

import com.dbs.capi.sg.encryption.utilities.entity.LobMap;
import com.dbs.capi.sg.encryption.utilities.entity.LobMapId;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface LobMapRepository extends JpaRepository<LobMap, LobMapId> {
    // Used in ValidLobValidator
    boolean existsByLobMapIdLob(String lob);

    // Used in ValidAppCodeValidator
    boolean existsByLobMapIdLobAndLobMapIdAppCode(String lob, String appCode);
}
```

---

## 5. Service Layer

### 5.1 Alert Service

#### Interface: `AlertService.java`

```java
package com.dbs.capi.sg.encryption.utilities.service;

import com.dbs.capi.sg.encryption.utilities.dto.AlertRequest;
import com.dbs.capi.sg.encryption.utilities.entity.Alert;

public interface AlertService {
    Alert createAlert(AlertRequest alertRequest);
}
```

#### Implementation: `AlertServiceImpl.java`

```java
package com.dbs.capi.sg.encryption.utilities.serviceImpl;

import com.dbs.capi.sg.encryption.utilities.dto.AlertRequest;
import com.dbs.capi.sg.encryption.utilities.entity.Alert;
import com.dbs.capi.sg.encryption.utilities.entity.LobMap;
import com.dbs.capi.sg.encryption.utilities.entity.LobMapId;
import com.dbs.capi.sg.encryption.utilities.repository.AlertRepository;
import com.dbs.capi.sg.encryption.utilities.repository.LobMapRepository;
import com.dbs.capi.sg.encryption.utilities.service.AlertService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.time.temporal.ChronoUnit;

@Service
public class AlertServiceImpl implements AlertService {

    private final AlertRepository alertRepository;
    private final LobMapRepository lobMapRepository;

    @Autowired
    public AlertServiceImpl(AlertRepository alertRepository, LobMapRepository lobMapRepository) {
        this.alertRepository = alertRepository;
        this.lobMapRepository = lobMapRepository;
    }

    @Override
    public Alert createAlert(AlertRequest alertRequest) {
        Alert alert = new Alert();
        alert.setProjectName(alertRequest.getProjectName());
        alert.setProjectDesc(alertRequest.getProjectDesc());
        alert.setOwnerName(alertRequest.getOwnerName());

        // Retrieve LobMap based on lob and appCode.
        LobMapId lobMapId = new LobMapId();
        lobMapId.setLob(alertRequest.getLob());
        lobMapId.setAppCode(alertRequest.getAppCode());
        // For new alerts, the record must exist as validated by our custom validators.
        LobMap lobMap = lobMapRepository.findById(lobMapId)
                .orElseThrow(() -> new RuntimeException("Invalid LOB and App Code combination"));
        alert.setLobMap(lobMap);

        alert.setExpiryDate(alertRequest.getExpiryDate());
        // Compute reminderDate as expiryDate minus reminderDays.
        Instant reminderDate = alertRequest.getExpiryDate().minus(alertRequest.getReminderDays(), ChronoUnit.DAYS);
        alert.setReminderDate(reminderDate);

        alert.setPrimaryContact(alertRequest.getPrimaryContact());
        alert.setNotificationEmails(alertRequest.getNotificationEmails());
        alert.setCreatedAt(Instant.now());

        return alertRepository.save(alert);
    }
}
```

### 5.2 LobMap Service

We also add an endpoint to allow users to add new lob mappings.

#### DTO for LobMap Creation: `LobMapRequest.java`

**File:** `LobMapRequest.java` (in the `dto` folder)

```java
package com.dbs.capi.sg.encryption.utilities.dto;

import jakarta.validation.constraints.NotBlank;
import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class LobMapRequest {

    @NotBlank(message = "LOB is required")
    private String lob;

    @NotBlank(message = "App Code is required")
    private String appCode;
}
```

#### Service Interface: `LobMapService.java`

```java
package com.dbs.capi.sg.encryption.utilities.service;

import com.dbs.capi.sg.encryption.utilities.dto.LobMapRequest;
import com.dbs.capi.sg.encryption.utilities.entity.LobMap;

public interface LobMapService {
    LobMap createLobMap(LobMapRequest lobMapRequest);
}
```

#### Service Implementation: `LobMapServiceImpl.java`

We also use Spring Cache here so that lob mappings are cached (and the cache is evicted on new entry).

```java
package com.dbs.capi.sg.encryption.utilities.serviceImpl;

import com.dbs.capi.sg.encryption.utilities.dto.LobMapRequest;
import com.dbs.capi.sg.encryption.utilities.entity.LobMap;
import com.dbs.capi.sg.encryption.utilities.entity.LobMapId;
import com.dbs.capi.sg.encryption.utilities.repository.LobMapRepository;
import com.dbs.capi.sg.encryption.utilities.service.LobMapService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.stereotype.Service;

@Service
public class LobMapServiceImpl implements LobMapService {

    private final LobMapRepository lobMapRepository;

    @Autowired
    public LobMapServiceImpl(LobMapRepository lobMapRepository) {
        this.lobMapRepository = lobMapRepository;
    }

    @Override
    @CacheEvict(value = "lobMapCache", allEntries = true) // clear cache on update
    public LobMap createLobMap(LobMapRequest lobMapRequest) {
        LobMap lobMap = new LobMap();
        LobMapId lobMapId = new LobMapId();
        lobMapId.setLob(lobMapRequest.getLob());
        lobMapId.setAppCode(lobMapRequest.getAppCode());
        lobMap.setLobMapId(lobMapId);
        return lobMapRepository.save(lobMap);
    }
}
```

---

## 6. REST Controllers

### 6.1 Alert Controller

**File:** `AlertController.java` (in the `controller` package)

```java
package com.dbs.capi.sg.encryption.utilities.controller;

import com.dbs.capi.sg.encryption.utilities.dto.AlertRequest;
import com.dbs.capi.sg.encryption.utilities.entity.Alert;
import com.dbs.capi.sg.encryption.utilities.service.AlertService;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/alerts")
public class AlertController {

    private final AlertService alertService;

    @Autowired
    public AlertController(AlertService alertService) {
        this.alertService = alertService;
    }

    @PostMapping("/createAlert")
    public ResponseEntity<Alert> createAlert(@Valid @RequestBody AlertRequest alertRequest) {
        Alert savedAlert = alertService.createAlert(alertRequest);
        return ResponseEntity.ok(savedAlert);
    }
}
```

### 6.2 LobMap Controller

**File:** `LobMapController.java` (in the `controller` package)

```java
package com.dbs.capi.sg.encryption.utilities.controller;

import com.dbs.capi.sg.encryption.utilities.dto.LobMapRequest;
import com.dbs.capi.sg.encryption.utilities.entity.LobMap;
import com.dbs.capi.sg.encryption.utilities.service.LobMapService;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/lobmap")
public class LobMapController {

    private final LobMapService lobMapService;

    @Autowired
    public LobMapController(LobMapService lobMapService) {
        this.lobMapService = lobMapService;
    }

    @PostMapping("/create")
    public ResponseEntity<LobMap> createLobMap(@Valid @RequestBody LobMapRequest lobMapRequest) {
        LobMap createdLobMap = lobMapService.createLobMap(lobMapRequest);
        return ResponseEntity.ok(createdLobMap);
    }
}
```

---

## 7. Main Application & Caching

Make sure caching is enabled in your main application class.

**File:** `CapiSgEncryptionUtilitiesApplication.java`

```java
package com.dbs.capi.sg.encryption.utilities;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableCaching
public class CapiSgEncryptionUtilitiesApplication {
    public static void main(String[] args) {
        SpringApplication.run(CapiSgEncryptionUtilitiesApplication.class, args);
    }
}
```

---

## 8. Final Considerations

- **Custom Validations:**  
  You already have `ValidDbsEmail`, `ValidDbsListEmail`, and `ValidReminderDays` implemented. We added custom annotations/validators for `ValidLob` and `ValidAppCode` here.
  
- **Composite Key & Caching:**  
  The LobMap entity uses an embedded key (LobMapId) and the LobMapService uses caching (via `@CacheEvict` in this example). You can further annotate retrieval methods (if any) with `@Cacheable`.

- **Service Layer:**  
  The AlertService computes the `reminderDate` (as `expiryDate` minus `reminderDays`) and saves the record. The LobMapService provides a way to add new LOB mappings.

- **Endpoints:**  
  - POST `/api/alerts/createAlert` accepts the AlertRequest JSON.  
  - POST `/api/lobmap/create` accepts a LobMapRequest to add a new LOB mapping.

- **Did We Miss Anything?**  
  This design covers your updated JSON payload, DTO validations (including custom validations for lob and appCode), entity definitions (with composite key for LobMap), service layer logic, and both endpoints. If you have additional business rules or security requirements, you can integrate those in later iterations.

This complete example should give you a solid, enterprise-ready starting point based on our previous conversation. Let me know if you need any further clarification or additional features!

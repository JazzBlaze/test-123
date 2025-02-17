Below is an example of a detailed Software Requirements Specification (SRS) for the new certificate management tool. In this document, I’ve broken down the project into its major components, described the architecture and data flow, and provided code snippets and design details for both the frontend and backend. You can customize names (for example, “Certificate Guardian” instead of “Certificate Manager”) as needed. 

---

# Software Requirements Specification (SRS)

## 1. Introduction

### 1.1 Purpose
The purpose of this SRS is to detail the requirements and design for a new tool (tentatively named **Certificate Guardian**) to manage and monitor digital certificates. This tool will be integrated into the existing dashboard application (React + TypeScript frontend, Spring Boot backend, MariaDB) and will enable both internal teams and third-party vendors to:
- Upload certificates (via file or text)
- Automatically extract and store the certificate’s expiry date
- Configure custom reminder intervals (default 90 days before expiry)
- Specify recipients for expiry notifications
- View a list of all certificate records

### 1.2 Scope
The Certificate Guardian tool will:
- Provide a form-based UI for creating new certificate records.
- Validate and parse certificates (whether uploaded or pasted) to extract metadata (notably the expiry date).
- Store certificate data along with metadata, user details, and scheduling information in MariaDB.
- Expose API endpoints to create and retrieve certificate records.
- Integrate with an internal email service to notify configured users on the reminder date, using an automation job (via Jenkins) to trigger the email workflow.

### 1.3 Definitions and Abbreviations
- **Cert**: Short for certificate.
- **UI**: User Interface.
- **API**: Application Programming Interface.
- **Jenkins**: An automation server used to run CI/CD pipelines.
- **DB**: Database.
- **Internal Org Client**: The internal service client used for sending emails.

---

## 2. System Overview

The Certificate Guardian tool is part of an existing dashboard. Users who have authenticated can:
- **Create New** certificate entries by filling out a form.
- **View** a table listing all stored certificate records.

On the backend, the system will:
- Validate and parse the provided certificate data.
- Extract key information (expiry date).
- Store all submitted information along with computed metadata.
- Schedule email notifications to be sent at the defined “reminder” date.
- Use Jenkins pipelines to automate periodic checking and triggering of the email notifications.

---

## 3. Architecture

### 3.1 High-Level Architecture Diagram

```plaintext
+-------------------+         HTTPS         +---------------------+
|   Frontend (UI)   | <-------------------> |   Backend (Spring)  |
| (React/TS/Styled) |                       |  API & Processing   |
+-------------------+                       +----------+----------+
                                                   |
                                                   | JDBC
                                                   v
                                            +--------------+
                                            |  MariaDB     |
                                            +--------------+
                                                   |
                                                   | Triggers via
                                                   | Jenkins Pipeline
                                                   v
                                        +----------------------+
                                        | Email Service Client |
                                        +----------------------+
```

### 3.2 Component Details

- **Frontend:**  
  - A new dashboard module providing two main actions:
    - **Create New:** Form for certificate data input.
    - **View:** Tabular display of certificate records.
  
- **Backend:**  
  - REST API endpoints to accept certificate submissions and fetch records.
  - Certificate parsing service (leveraging built-in Java/Spring classes) to extract expiry information.
  - Database persistence layer (using Spring Data JPA or similar) for storing certificate records.

- **Database (MariaDB):**  
  - A table (e.g., `certificates`) with fields for storing metadata, certificate content (as text or BLOB), expiry date, reminder date, and audit fields.

- **Automation (Jenkins):**  
  - A Jenkins pipeline/job to periodically query the DB for records due for reminders and trigger the internal email service via an API call.

---

## 4. Functional Requirements

### 4.1 Frontend

#### 4.1.1 Certificate Submission Form
- **Fields:**
  - **Project Name:** Text input.
  - **Owner Name:** Text input (e.g., third-party vendor name).
  - **Certificate Input:**  
    - **File Upload:** Drag and drop or browse.
    - **Fallback Text Input:** Text area to paste certificate content.
  - **When to Remind:**  
    - Default: 90 days before expiry.
    - Option to override with a custom number of days.
  - **Notification Emails:**  
    - Input field for one or multiple email addresses.

- **Validation:**
  - Mandatory fields must be filled.
  - Validate file type if a file is uploaded.
  - Validate certificate content (format checking).

- **Behavior:**
  - On submission, the form sends a POST request to the backend API.
  - Display error messages or success confirmations.

*Example (React/TypeScript snippet):*

```tsx
// CertificateForm.tsx
import React, { useState } from 'react';

const CertificateForm: React.FC = () => {
  const [projectName, setProjectName] = useState('');
  const [ownerName, setOwnerName] = useState('');
  const [certContent, setCertContent] = useState('');
  const [reminderDays, setReminderDays] = useState(90);
  const [emails, setEmails] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const payload = {
      projectName,
      ownerName,
      certificate: certContent,
      reminderDays,
      emails: emails.split(',').map(email => email.trim())
    };
    // API call using fetch/axios
    try {
      const response = await fetch('/api/certificates', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
      if (response.ok) {
        // Handle success (e.g., show a notification)
      }
    } catch (err) {
      // Handle error
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        placeholder="Project Name"
        value={projectName}
        onChange={e => setProjectName(e.target.value)}
        required
      />
      <input
        type="text"
        placeholder="Owner Name"
        value={ownerName}
        onChange={e => setOwnerName(e.target.value)}
        required
      />
      <textarea
        placeholder="Paste certificate content here or use file upload below..."
        value={certContent}
        onChange={e => setCertContent(e.target.value)}
        required
      />
      <input
        type="number"
        placeholder="Days before expiry to remind"
        value={reminderDays}
        onChange={e => setReminderDays(parseInt(e.target.value))}
      />
      <input
        type="text"
        placeholder="Notification Emails (comma separated)"
        value={emails}
        onChange={e => setEmails(e.target.value)}
      />
      <button type="submit">Submit</button>
    </form>
  );
};

export default CertificateForm;
```

#### 4.1.2 Certificate Records View
- **Display:**
  - Tabular list of certificate records.
  - Columns: Project Name, Owner, Expiry Date, Reminder Date, Notification Emails, Created At, and an action to view/download the certificate content.

- **Behavior:**
  - On load, fetch the certificate records from the backend.
  - Allow sorting and filtering if needed.

---

### 4.2 Backend

#### 4.2.1 API Endpoints

- **POST /api/certificates:**  
  - Accepts certificate data from the frontend.
  - Validates input.
  - Processes the certificate (validation and expiry extraction).
  - Persists the record in the database.
  
- **GET /api/certificates:**  
  - Returns all certificate records in JSON format for display in the frontend.

#### 4.2.2 Certificate Processing
- **Validation & Parsing:**
  - Determine if the certificate is provided as text.
  - Use Java’s `CertificateFactory` and `X509Certificate` for parsing.
  - Extract expiry date and validate format.

*Example Java snippet for certificate parsing:*

```java
import java.io.ByteArrayInputStream;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.util.Date;

public class CertificateUtil {

    public static Date extractExpiryDate(String certContent) throws Exception {
        CertificateFactory cf = CertificateFactory.getInstance("X.509");
        ByteArrayInputStream bis = new ByteArrayInputStream(certContent.getBytes());
        X509Certificate cert = (X509Certificate) cf.generateCertificate(bis);
        return cert.getNotAfter();
    }
}
```

- **Persistence:**
  - Use Spring Data JPA to persist records.
  - Define an entity (e.g., `CertificateRecord`) with fields:
    - `id` (primary key, auto-generated)
    - `projectName` (String)
    - `ownerName` (String)
    - `certificateContent` (TEXT or BLOB)
    - `fileType` (String, optional if a file is uploaded)
    - `expiryDate` (Date)
    - `reminderDate` (Date – calculated based on `expiryDate` and `reminderDays`)
    - `notificationEmails` (JSON or a comma-separated String)
    - `userId` (String)
    - `createdAt` (Timestamp)

*Example entity definition:*

```java
import javax.persistence.*;
import java.util.Date;

@Entity
@Table(name = "certificates")
public class CertificateRecord {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String projectName;
    private String ownerName;

    @Lob
    private String certificateContent;

    private String fileType;
    private Date expiryDate;
    private Date reminderDate;

    private String notificationEmails;
    private String userId;

    @Temporal(TemporalType.TIMESTAMP)
    private Date createdAt;

    // Getters and setters omitted for brevity
}
```

- **Service Layer:**
  - A service to handle business logic:
    - Validate and parse certificate.
    - Compute `reminderDate` (e.g., `expiryDate - reminderDays`).
    - Save the record via a repository.

#### 4.2.3 Email Notification Triggering

- **Job Scheduling:**
  - After each certificate record insertion, a Jenkins job/pipeline is triggered.
  - Alternatively, a scheduled Spring Boot task can periodically check the DB for records with a reminder date equal to the current date.

- **Jenkins Pipeline Integration:**
  - A Jenkins pipeline can be configured to:
    - Query the database (or hit a dedicated endpoint) for due certificates.
    - Invoke the internal email service client to send notifications.
    
*Example Jenkins pipeline snippet (Declarative Pipeline):*

```groovy
pipeline {
    agent any
    triggers {
        cron('H 0 * * *') // Run daily at a random minute past midnight
    }
    stages {
        stage('Check Expiring Certificates') {
            steps {
                script {
                    // Assuming you have a script or API endpoint to fetch expiring certificates.
                    def response = sh(script: "curl -s http://backend-service/api/certificates/due", returnStdout: true)
                    echo "Due Certificates: ${response}"
                    
                    // Call internal email client for each due certificate
                    // This could be a loop through parsed JSON and triggering email sending.
                }
            }
        }
    }
}
```

- **Internal Email Service Client:**
  - The backend will include a client (e.g., using Spring’s `RestTemplate` or `WebClient`) to call the internal email service API.
  - The payload would include certificate details and the list of email addresses.

---

## 5. Non-Functional Requirements

- **Security:**  
  - Integration with existing authentication.
  - Secure API endpoints (e.g., using JWT or session-based auth).
  
- **Performance:**  
  - The certificate parsing and DB write operations are expected to be fast.
  - Scheduled email notifications should be processed without impacting frontend responsiveness.
  
- **Scalability:**  
  - The design should support moderate certificate volumes.
  - The DB schema and API are designed to scale as usage increases.
  
- **Reliability & Maintainability:**  
  - Code should be well-documented and modular.
  - Use unit and integration tests (e.g., for certificate parsing logic).

---

## 6. Data Flow and Process Flow

1. **Certificate Submission:**
   - User fills out the form on the frontend.
   - Data is sent as a POST request to `/api/certificates`.

2. **Backend Processing:**
   - Controller receives the request and passes data to the service layer.
   - The service validates the certificate input.
   - Certificate parsing utility extracts the expiry date.
   - The service calculates the reminder date (`expiryDate - reminderDays`).
   - Data is persisted to MariaDB.

3. **Record Retrieval:**
   - A GET request to `/api/certificates` returns all records for the frontend table.

4. **Email Notification:**
   - A Jenkins job (or a Spring Boot scheduled task) periodically checks for certificates with a reminder date matching the current date.
   - For each due certificate, the internal email service is invoked to send notifications to the specified email addresses.

---

## 7. Testing and Deployment

### 7.1 Testing
- **Unit Tests:**  
  - Test certificate parsing logic (valid and invalid certificates).
  - Test calculation of reminder dates.
  
- **Integration Tests:**  
  - API endpoint tests (using MockMvc for Spring Boot).
  - DB integration tests (using an in-memory database for testing).

- **UI Tests:**  
  - Form validation and submission.
  - Data fetching for the certificate table.

### 7.2 Deployment
- **Backend:**  
  - Deployed as part of the Spring Boot service.
  - Managed via Jenkins pipelines.
  
- **Frontend:**  
  - Integrated into the existing dashboard repo.
  - Deployed using the existing React build pipeline.

- **Jenkins:**  
  - A dedicated pipeline/job for triggering the email notifications based on due certificate records.

---

## 8. Appendices

### 8.1 Future Enhancements
- File preview/download for uploaded certificates.
- Additional certificate validation (e.g., checking issuer information).
- Enhanced UI features such as filtering and sorting in the certificate table.

### 8.2 References
- Java Documentation on `CertificateFactory` and `X509Certificate`.
- Spring Boot and Spring Data JPA documentation.
- Jenkins Pipeline documentation.

---

This SRS provides a comprehensive guide to implementing the Certificate Guardian tool, detailing the frontend form, backend processing, database design, and Jenkins-based automation. Adjust and expand upon each section as your team iterates on the design and requirements.

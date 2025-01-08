# **Fuel-Conversion-Service**

## **1. Project Overview**
This document outlines a scalable, serverless AWS solution designed to standardize fuel data from suppliers like Shell and OKQ8. The system automates the conversion of diverse file formats into a unified XML structure, providing secure APIs for file uploads, status tracking, and seamless integration with Coreview.

---

## **2. Architecture**
### **2.1 Overview**
The Fuel-Conversion-Service uses a serverless, event-driven architecture built on AWS. Files are uploaded to S3, where events trigger Lambda functions to process and convert data into a standardized XML format. DynamoDB tracks file metadata and processing states, while API Gateway provides secure endpoints for file uploads, status tracking, and integration with Coreview Play and Coreview Prod environments.
![Initial draft of architecture](./architecture.svg)

---

### **2.2 Components**
- **S3 Bucket**:  
  The S3 bucket serves as the primary storage solution for the system. It is used to store both raw and processed fuel files securely. Raw files are uploaded by users, and processed files are generated after conversion. The bucket supports event notifications, triggering further processing as new files are uploaded.

- **DynamoDB**:  
  DynamoDB acts as the metadata storage layer, keeping track of each file's details, such as upload timestamps, processing states, and relevant metadata. It enables efficient querying and status tracking for files as they move through the processing pipeline.

- **Lambda Functions**:  
  AWS Lambda functions serve as the core processing and API logic for the system. Below is a breakdown of the Lambda endpoints included:

  - **`generate-presigned-url` Lambda**:  
    Generates secure pre-signed URLs for file uploads to S3. These URLs allow users to upload files directly to the S3 bucket without needing additional authentication.

  - **`file-status` Lambda**:  
    Retrieves the current processing status and metadata of a specific file by querying DynamoDB.

  - **`list-files` Lambda**:  
    Provides a list of all uploaded files, including their statuses and metadata. This Lambda queries DynamoDB for bulk file details.

  - **`conversion` Lambda**:  
    Handles the processing and conversion of raw fuel data files into a standardized XML format. This Lambda is triggered by S3 events when a new file is uploaded. Initially designed as a single function with a switch-case structure, it can later be modularized into supplier-specific handlers.

  - **`send-to-coreview-play` Lambda**:  
    Pushes processed and validated files to the Coreview Play environment for testing and validation.

  - **`send-to-coreview-prod` Lambda**:  
    Sends validated files from Coreview Play to the Coreview Prod environment for integration into the production system.

- **Other Services**:  
  Additional AWS services such as **API Gateway** are used to expose secure REST endpoints for file upload, status tracking, and integration with downstream systems. The architecture also relies on **IAM roles** for secure access control and **CloudWatch** for monitoring, logging, and ensuring observability of system components.

---

### **2.3 Data Flow**

Below is a step-by-step description of how data moves through the system:

1. **File Ingestion:**
   - Users initiate the process by uploading fuel data files via the **UI**.
   - A **pre-signed URL** is generated using the `/generate-presigned-url` API endpoint. This URL allows users to securely upload their files to the **S3 Bucket** without requiring additional authentication.
   - Uploaded files are stored in the S3 bucket as raw files, ready for processing.

2. **Metadata Storage:**
   - Upon successful file upload, metadata (such as the file name, upload timestamp, and initial state) is recorded in **DynamoDB**.
   - DynamoDB tracks the processing state of each file, ensuring traceability throughout the pipeline.

3. **File Processing:**
   - When a new file is uploaded, an event notification triggers the **Conversion Lambda**.
   - The Lambda function processes the raw file, converting it into a standardized XML format based on the supplier-specific requirements.
   - Processed files are saved back to the S3 bucket in a designated folder for processed outputs.
   - DynamoDB is updated with the file's new state (e.g., `Processed`) and any additional metadata.

4. **File Status Tracking:**
   - Users can query the status of their uploaded files using the `/file-status` API endpoint. This provides details such as the file ID, current processing state (e.g., `Uploaded`, `Processed` , `Sent-to-coreview-play` or `sent-to-coreview-prod`), and a history of state transitions with timestamps.

5. **Integration with Coreview:**
   - Processed files are sent to the **Coreview Play** environment for validation using the `/send-to-coreview-play` API.
   - After successful validation, the file is pushed to the **Coreview Prod** environment via the `/send-to-coreview-prod` API.
   - The integration ensures that downstream systems receive fully standardized and validated fuel data.

6. **Output and Reporting:**
   - Users can access a list of all uploaded files and their statuses using the `/list-files` API endpoint.
   - Processed files are ready for further use or reporting within the Coreview systems.

---

## **3. Supplier Details**
The **Fuel-Conversion-Service** handles data from a variety of suppliers, each with unique formats, fuel types, and market-specific requirements. The table below outlines the key suppliers, their respective markets, and the details of the conversion process, including priorities, formats, and difficulty levels.

**Table 1: Supplier and Fuel Conversion Details**  

| Priority | Supplier    | Market | Type        | Specified Type   | Conversion Difficulty | Incoming Format | Outgoing Format      |
|----------|-------------|--------|-------------|------------------|-----------------------|-----------------|----------------------|
| 1        | Preem       | SE     | FUEL        | Fossil           | High                 | TXT             | File read - XML      |
| 1        | OKQ8        | SE     | FUEL        | Fossil/Electric  | High                 | CSV             | File read - XML      |
| 1        | Circle K    | SE     | FUEL        | Fossil/Electric  | High                 | TXT             | File read - XML      |
| 1        | Shell/ST1   | SE     | FUEL        | Fossil           | High                 | TXT             | File read - XML      |
| 1        | EON         | SE     | FUEL        | Electric         | Low (built)          | API             | File read - XML      |
| 1        | DCS         | SE     | FUEL        | Electric         | Low (built)          | API             | File read - XML      |
| 2        | Circle K    | DK     | FUEL        | Fossil           | High                 | DAT             | File read - XML      |
| 2        | Shell/ST1   | DK     | FUEL        | Fossil           | High                 | TXT             | File read - XML      |
| 2        | UNO X       | DK     | FUEL        | Fossil           | High                 | CSV             | File read - XML      |
| 2        | Q8          | DK     | FUEL        | Fossil           | High                 | .file           | File read - XML      |
| 2        | Clever      | DK     | FUEL        | Electric         | High                 | XLSX            | File read - XML?     |
| 2        | EON         | DK     | FUEL        | Electric         | High                 | XLSX            | File read - XML?     |
| 2        | Nordlys     | DK     | FUEL        | Electric         | High                 | XLSX            | File read - XML?     |
| 3        | Vegamot     | NO     | Bomtoll     | Tolls            | Low                  | XLSX            | InExchange/API?      |
| 3        | Fjellinjen  | NO     | Bomtoll     | Tolls            | Low                  | XLSX            | InExchange/API?      |
| 3        | Ferde       | NO     | Bomtoll     | Tolls            | Low                  | XLSX            | InExchange/API?      |

---

## **4. API Specifications**

### **4.1 Overview**
The API endpoints serve as the primary interface for users to interact with the **Fuel-Conversion-Service**. They enable secure file uploads, track file processing statuses, and facilitate the integration of processed files with downstream systems such as Coreview.

### **4.2 Endpoints**

#### **4.2.1 `generate-presigned-url`**
- **Purpose**: Generates a pre-signed URL for secure file upload to the S3 bucket.
- **Method**: `POST`
- **Endpoint URL**: `/generate-presigned-url`
- **Parameters**: 
  - `fileName` *(string, required)*: Name of the file to be uploaded.
  - `fileType` *(string, required)*: MIME type of the file.
- **Response**:
  - `presignedUrl` *(string)*: URL for uploading the file.
  - `s3Key` *(string)*: S3 object key where the file will be stored.

---

#### **4.2.2 `file-status`**
- **Purpose**: Retrieves the current status and metadata of a specific file.
- **Method**: `GET`
- **Endpoint URL**: `/file-status/{fileId}`
- **Parameters**: 
  - `fileId` *(string, required)*: Unique identifier for the file.
- **Response**:
  - `fileId` *(string)*: Identifier for the file.
  - `status` *(string)*: Current processing status (e.g., `Uploaded`, `Processed`, `Sent-to-coreview-play` or `Sent-to-coreview-prod`).
  - `history` *(array)*: List of state transitions with timestamps.

---

#### **4.2.3 `send-to-coreview-play`**
- **Purpose**: Triggers the sending of a processed file to the Coreview Play environment for validation.
- **Method**: `POST`
- **Endpoint URL**: `/send-to-coreview-play`
- **Parameters**: 
  - `fileId` *(string, required)*: Identifier for the processed file.
- **Response**:
  - `status` *(string)*: Indicates success or failure of the operation.
  - `message` *(string)*: Additional details about the operation.

---

#### **4.2.4 `send-to-coreview-prod`**
- **Purpose**: Sends a validated file to the Coreview Prod environment after successful submission to Coreview Play.
- **Method**: `POST`
- **Endpoint URL**: `/send-to-coreview-prod`
- **Parameters**: 
  - `fileId` *(string, required)*: Identifier for the file to be sent.
- **Response**:
  - `status` *(string)*: Indicates success or failure of the operation.
  - `message` *(string)*: Additional details about the operation.

---

#### **4.2.5 `list-files`**
- **Purpose**: Retrieves a list of all files uploaded by the user along with their statuses.
- **Method**: `GET`
- **Endpoint URL**: `/list-files`
- **Response**:
  - `files` *(array)*: List of uploaded files with metadata and statuses.

---

### **5. Timeline**

### **5.1 Phase 1: Planning and Infrastructure Setup**  
**Duration**: 2 weeks  

| Task No. | Task                     | Details                                                                                          | Estimation  |
|----------|--------------------------|--------------------------------------------------------------------------------------------------|-------------|
| 1.1      | Architecture Design      | Finalize architecture for serverless implementation, including AWS services and data flow.       | 1 week      |
| 1.2      | CI/CD Pipeline Setup     | Develop CI/CD pipelines for infrastructure and code deployment to AWS.                           | 1 week      |

---

### **5.2 Phase 2: Core Development**  
**Duration**: 11 weeks  

| Task No. | Task                                       | Details                                                                                                           | Estimation  |
|----------|--------------------------------------------|-------------------------------------------------------------------------------------------------------------------|-------------|
| 2.1      | Design DynamoDB schema                     | Design DynamoDB database schema for tracking file metadata and implement in terraform.                            | 1 week      |
| 2.2      | Design S3 file structure                   | Decide on and design file structure for all the different suppliers.                                              | 2 days      |
| 2.3      | Develop File Conversion Lambda             | Develop Lambda for converting supplier files into standardized XML format. First supplier integration.            | 3 weeks     |
| 2.4      | Develop `/file-status` endpoint            | Develop API to retrieve current file processing statuses and metadata from DynamoDB.                              | 1 week      |
| 2.5      | Develop `/generate-presigned-url` endpoint | Develop API to generate secure pre-signed URLs for file uploads to S3.                                            | 1 week      |
| 2.6      | Develop `/send-to-coreview-play` endpoint  | Develop API to send processed files to Coreview Play for validation.                                              | 2 weeks     |
| 2.7      | Develop `/send-to-coreview-prod` endpoint  | Develop API to send validated files to Coreview Prod.                                                             | 1 week      |
| 2.8      | Develop `/list-files` endpoint             | Develop API to list all uploaded files and their statuses, querying DynamoDB.                                     | 1 week      |
| 2.9      | Security Configuration	                    | Set up IAM roles with the principle of least privilege.                                                           | 1 week      |

---

### **5.3 Phase 3: Supplier-Specific Extensions**  
**Duration**: 3 weeks  

| Task No. | Task                       | Details                                                                                          | Estimation  |
|----------|----------------------------|--------------------------------------------------------------------------------------------------|-------------|
| 3.1      | Write library code         | Refactor and write library for code shared between supplier integrations.                        | 1 week      |
| 3.2      | Add supplier integration   | Add support for one additional supplier. Time will vary depending on difficulty level.           | 2 weeks     |

---

### **5.4 Phase 4: Testing and Optimization**  
**Duration**: 6 weeks  

| Task No. | Task                       | Details                                                                                          | Estimation  |
|----------|----------------------------|--------------------------------------------------------------------------------------------------|-------------|
| 4.1      | Unit Testing               | Develop unit tests for all Lambdas, including edge cases.                                        | 2 weeks     |
| 4.2      | Integration Testing        | Test end-to-end functionality across all services and workflows.                                 | 4 weeks     |

---

### **5.5 Phase 5: Deployment and Handover**  
**Duration**: 1 week  

| Task No. | Task Name                  | Details                                                                                          | Estimation  |
|----------|----------------------------|--------------------------------------------------------------------------------------------------|-------------|
| 5.1      | Final Deployment           | Deploy to production environment and handover.                                                   | 1 week      |

---

### **5.6 Overall Timeline**  
**23 Weeks (Approx. 5.5 Months)**

---

### **Key Deliverables**

1. **API Endpoints**:
   - `/generate-presigned-url`: Enable secure file uploads.
   - `/file-status`: Provide processing status and metadata for individual files.
   - `/list-files`: Retrieve a list of all uploaded files and their statuses.
   - `/send-to-coreview-play`: Push processed files for validation in the Coreview Play environment.
   - `/send-to-coreview-prod`: Finalize integration by sending validated files to Coreview Prod.

2. **Processed Outputs**:
   - Standardized XML files stored in S3, supporting at least two high-priority suppliers (e.g., OKQ8 or Shell/ST1).

3. **Reports and Monitoring**:
   - Processing status and error logging via CloudWatch.
   - DynamoDB state tracking for file metadata and processing history.

4. **Documentation**:
   - Detailed API specifications and architecture diagrams.
   - Supplier-specific integration guides for the first supported supplier.

5. **Validated Integrations**:
   - Functional connections to both Coreview Play and Coreview Prod for processing and finalizing data.

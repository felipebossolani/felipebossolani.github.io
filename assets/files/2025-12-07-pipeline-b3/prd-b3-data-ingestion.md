# Product Requirements Document (PRD)
## B3 Data Ingestion Pipeline with Cloudflare R2 Integration

**Version:** 1.0  
**Date:** December 7, 2024  
**Project:** Brazilian Financial Market Data Container  
**Component:** B3 Instruments Data Pipeline

---

## 1. Executive Summary

### 1.1 Objective
Migrate the B3 instruments data download pipeline from local file storage to Cloudflare R2 object storage, with event-driven processing triggered automatically upon file upload.

### 1.2 Scope
- Adapt existing download script to upload to Cloudflare R2
- Implement local R2 emulator for development/testing (Docker-based)
- Configure event notification system for automatic processing triggers
- Implement standardized file naming convention
- Create robust task breakdown with incremental commits for easy rollback

### 1.3 Success Criteria
- ✅ Files successfully uploaded to R2 after download
- ✅ Local R2 emulator working for development
- ✅ Event-driven processing triggered on file upload
- ✅ Standardized file naming (yyyy-mm-dd.csv)
- ✅ Successful test run for December 5, 2025 data
- ✅ All sensitive credentials in .env (not committed)

---

## 2. Technical Architecture

### 2.1 Storage Structure
```
financial-data/                    # R2 Bucket
└── b3/
    └── instruments/
        ├── 2024-12-05.csv
        ├── 2024-12-06.csv
        └── 2024-12-07.csv
```

### 2.2 File Naming Convention
**Pattern:** `yyyy-mm-dd.csv`

**Examples:**
- `2024-12-05.csv`
- `2025-01-15.csv`

**Rules:**
- ISO 8601 date format (YYYY-MM-DD)
- Always lowercase extension
- Overwrite if file already exists (idempotent)

### 2.3 Data Flow
```
┌─────────────────┐
│  B3 Download    │
│  (Daily CSV)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Upload to R2   │
│  (S3 API)       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  R2 Event       │
│  Notification   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  CF Queue       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  CF Worker      │
│  (Webhook)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  VPS Processor  │
│  (PostgreSQL)   │
└─────────────────┘
```

---

## 3. Implementation Requirements

### 3.1 Development Environment

#### 3.1.1 Local R2 Emulator
**Technology:** MinIO (S3-compatible storage)

**Docker Compose Configuration:**
```yaml
# docker-compose.yml
version: '3.8'
services:
  minio:
    image: minio/minio:latest
    container_name: local-r2-emulator
    ports:
      - "9000:9000"      # API
      - "9001:9001"      # Console
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    command: server /data --console-address ":9001"
    volumes:
      - ./minio-data:/data
```

**Requirements:**
- Must support S3-compatible API
- Should persist data locally during development
- Must allow bucket creation via API
- Should provide web console for debugging

#### 3.1.2 Environment Variables

**Required Variables:**
```bash
# Production R2
R2_ACCOUNT_ID=<cloudflare_account_id>
R2_ACCESS_KEY_ID=<r2_access_key>
R2_SECRET_ACCESS_KEY=<r2_secret_key>
R2_BUCKET_NAME=financial-data
R2_ENDPOINT=https://<account_id>.r2.cloudflarestorage.com

# Local Development (MinIO)
LOCAL_R2_ENDPOINT=http://localhost:9000
LOCAL_R2_ACCESS_KEY=minioadmin
LOCAL_R2_SECRET_KEY=minioadmin123
LOCAL_R2_BUCKET_NAME=financial-data

# Environment Mode
ENVIRONMENT=development  # or 'production'

# B3 Configuration
B3_INSTRUMENTS_URL=<b3_instruments_url>
B3_INSTRUMENTS_PATH=b3/instruments/
```

### 3.2 Code Adaptations

#### 3.2.1 Configuration Module
**File:** `config/r2_config.py`

**Requirements:**
- Support both production R2 and local MinIO
- Auto-select endpoint based on ENVIRONMENT variable
- Provide singleton S3 client
- Handle bucket creation if not exists
- Include connection retry logic

#### 3.2.2 Download Script Adaptation
**File:** `scripts/download_b3_instruments.py`

**Current Behavior:**
- Downloads CSV from B3
- Saves to local filesystem

**Required Changes:**
- Add R2 upload after successful download
- Remove local file after upload (optional: configurable)
- Use standardized file naming (yyyy-mm-dd.csv)
- Handle duplicate files (overwrite strategy)
- Add upload retry logic (3 attempts)
- Log all operations (download, upload, delete)

**Function Signature:**
```python
def download_and_upload_b3_instruments(date: datetime) -> str:
    """
    Downloads B3 instruments file and uploads to R2.
    
    Args:
        date: Target date for download
        
    Returns:
        S3 key of uploaded file
        
    Raises:
        DownloadError: If B3 download fails
        UploadError: If R2 upload fails
    """
```

#### 3.2.3 Storage Manager
**File:** `utils/storage_manager.py`

**Requirements:**
- Abstract S3 operations (upload, download, delete, list)
- Support both local and production environments
- Implement file naming standardization
- Handle overwrite logic
- Include metadata tracking (upload timestamp, file size)

**Key Methods:**
```python
class StorageManager:
    def upload_instrument_file(self, local_path: Path, date: datetime) -> str
    def download_instrument_file(self, date: datetime, local_path: Path) -> None
    def delete_instrument_file(self, date: datetime) -> None
    def list_instrument_files(self, start_date: datetime = None, end_date: datetime = None) -> List[str]
    def file_exists(self, date: datetime) -> bool
```

### 3.3 Event-Driven Processing

#### 3.3.1 Cloudflare Event Notifications
**Configuration Location:** Cloudflare Dashboard > R2 > financial-data > Settings > Event Notifications

**Required Setup:**
1. Create Cloudflare Queue: `b3-instruments-upload-queue`
2. Configure R2 Event Notification:
   - Event type: `object-create`
   - Prefix filter: `b3/instruments/`
   - Destination: `b3-instruments-upload-queue`

#### 3.3.2 Cloudflare Worker
**File:** `workers/r2-event-handler.js`

**Requirements:**
- Consume messages from `b3-instruments-upload-queue`
- Extract file metadata (bucket, key, size, timestamp)
- Call VPS webhook with file information
- Implement retry logic (3 attempts with exponential backoff)
- Handle webhook failures gracefully

**Webhook Payload:**
```json
{
  "event_type": "b3_instrument_uploaded",
  "bucket": "financial-data",
  "key": "b3/instruments/2024-12-05.csv",
  "size": 1234567,
  "timestamp": "2024-12-07T10:30:00Z",
  "etag": "abc123..."
}
```

#### 3.3.3 VPS Webhook Endpoint
**File:** `api/webhooks/r2_events.py`

**Requirements:**
- Accept POST requests from Cloudflare Worker
- Validate webhook signature (optional: implement later)
- Queue processing job
- Return 200 OK immediately (don't block)
- Log all webhook events

**Endpoint:** `POST /webhooks/r2/instrument-uploaded`

### 3.4 Testing Requirements

#### 3.4.1 Local Development Test
**Test Date:** December 5, 2025

**Test Steps:**
1. Start MinIO container
2. Create `financial-data` bucket
3. Run download script for 2025-12-05
4. Verify file uploaded to MinIO: `b3/instruments/2024-12-05.csv`
5. Verify file naming convention
6. Verify file can be downloaded back
7. Test overwrite scenario (re-run same date)

**Expected Results:**
- ✅ File exists in MinIO at correct path
- ✅ File name matches yyyy-mm-dd.csv format
- ✅ File size > 0 bytes
- ✅ CSV encoding is cp1252
- ✅ Overwrite works without errors

#### 3.4.2 Production Test (Manual)
**Prerequisites:**
- Cloudflare R2 credentials configured
- Event notification setup (optional for first test)

**Test Steps:**
1. Set ENVIRONMENT=production in .env
2. Run download script for recent date
3. Verify file in Cloudflare R2 dashboard
4. Verify file naming and path
5. Test event notification (if configured)

---

## 4. Task Breakdown & Git Strategy

### 4.1 Branch Strategy
**Branch Name:** `feature/b3-r2-integration`

**Branch from:** `main` (or `develop`)

**Merge Strategy:** Squash merge after review

### 4.2 Task List with Commits

#### **Task 1: Project Setup & Configuration**
**Subtasks:**

**1.1 Create feature branch**
```bash
git checkout -b feature/b3-r2-integration
```
**Commit:** `chore: create feature branch for B3 R2 integration`

**1.2 Update .gitignore**
- Add `.env`
- Add `.env.*`
- Add `minio-data/`
- Add `*.log`
- Add `__pycache__/`

**Commit:** `chore: update .gitignore for R2 integration`

**1.3 Create .env.example**
- Template with all required variables
- Placeholder values
- Comments explaining each variable

**Commit:** `docs: add .env.example template for R2 configuration`

**1.4 Create docker-compose.yml**
- MinIO service configuration
- Network setup
- Volume mounts

**Commit:** `feat: add MinIO docker-compose for local R2 emulation`

---

#### **Task 2: Core Configuration Module**

**2.1 Create config/r2_config.py**
- Environment detection logic
- S3 client factory
- Connection configuration

**Commit:** `feat: implement R2 configuration module with env detection`

**2.2 Add bucket initialization**
- Auto-create bucket if not exists
- Handle bucket creation errors
- Add logging

**Commit:** `feat: add automatic bucket creation in R2Config`

**2.3 Add connection testing**
- Health check method
- Connection retry logic
- Timeout handling

**Commit:** `feat: implement R2 connection health check and retry logic`

---

#### **Task 3: Storage Manager Implementation**

**3.1 Create utils/storage_manager.py skeleton**
- Class definition
- Method signatures
- Docstrings

**Commit:** `feat: create StorageManager class skeleton`

**3.2 Implement upload_instrument_file()**
- File naming standardization
- S3 upload with metadata
- Error handling

**Commit:** `feat: implement instrument file upload to R2`

**3.3 Implement download_instrument_file()**
- S3 download logic
- Local path handling
- Error handling

**Commit:** `feat: implement instrument file download from R2`

**3.4 Implement file_exists() and list_instrument_files()**
- File existence check
- Date range filtering
- S3 prefix listing

**Commit:** `feat: add file existence check and listing methods`

**3.5 Implement delete_instrument_file()**
- S3 delete operation
- Confirmation logging
- Error handling

**Commit:** `feat: implement instrument file deletion from R2`

---

#### **Task 4: Download Script Adaptation**

**4.1 Backup existing download script**
```bash
cp scripts/download_b3_instruments.py scripts/download_b3_instruments.py.backup
```
**Commit:** `chore: backup original B3 download script`

**4.2 Refactor download logic**
- Separate download and upload concerns
- Add date parameter
- Improve error handling

**Commit:** `refactor: separate B3 download logic from storage`

**4.3 Integrate R2 upload**
- Call StorageManager after download
- Add file naming standardization
- Handle temp files properly

**Commit:** `feat: integrate R2 upload in B3 download pipeline`

**4.4 Add overwrite handling**
- Check if file exists
- Implement overwrite strategy
- Log overwrite operations

**Commit:** `feat: implement file overwrite handling in download script`

**4.5 Add CLI arguments**
- Date argument (--date)
- Environment override (--env)
- Verbose logging (--verbose)

**Commit:** `feat: add CLI arguments to download script`

---

#### **Task 5: Local Testing Setup**

**5.1 Create test fixtures**
- Mock B3 CSV file
- Test data generator
- Date fixtures

**Commit:** `test: add test fixtures for B3 instruments`

**5.2 Create integration test script**
- MinIO connection test
- Upload/download test
- File naming test

**Commit:** `test: add integration tests for R2 storage operations`

**5.3 Document local testing procedure**
- README section for testing
- MinIO setup instructions
- Troubleshooting guide

**Commit:** `docs: add local testing documentation for R2 integration`

---

#### **Task 6: Event Notification Setup (Optional - Can be separate PR)**

**6.1 Create Cloudflare Worker**
- Queue consumer logic
- Webhook caller
- Error handling

**Commit:** `feat: create Cloudflare Worker for R2 event handling`

**6.2 Create VPS webhook endpoint**
- FastAPI/Flask endpoint
- Request validation
- Job queuing

**Commit:** `feat: implement VPS webhook endpoint for R2 events`

**6.3 Document event notification setup**
- Cloudflare dashboard steps
- Worker deployment guide
- Testing guide

**Commit:** `docs: add event notification setup guide`

---

#### **Task 7: Production Readiness**

**7.1 Add comprehensive logging**
- Structured logging
- Log levels
- Log rotation

**Commit:** `feat: implement comprehensive logging system`

**7.2 Add monitoring/alerting hooks**
- Success/failure metrics
- Error notifications
- Performance tracking

**Commit:** `feat: add monitoring and alerting infrastructure`

**7.3 Create deployment documentation**
- Production setup guide
- Environment configuration
- Rollback procedures

**Commit:** `docs: add production deployment guide`

---

#### **Task 8: Final Testing & Documentation**

**8.1 Run December 5, 2025 test**
- Execute full pipeline locally
- Document results
- Fix any issues

**Commit:** `test: verify pipeline with December 5, 2025 data`

**8.2 Update main README**
- Architecture diagram
- Setup instructions
- Usage examples

**Commit:** `docs: update README with R2 integration documentation`

**8.3 Create CHANGELOG entry**
- List all changes
- Migration notes
- Breaking changes

**Commit:** `docs: add CHANGELOG entry for R2 integration`

---

### 4.3 Commit Message Convention

**Format:**
```
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `test`: Adding tests
- `refactor`: Code refactoring
- `chore`: Build/config changes

**Examples:**
```
feat(storage): implement R2 upload with retry logic

Added StorageManager.upload_instrument_file() with:
- 3 retry attempts
- Exponential backoff
- Comprehensive error logging

Closes #123
```

---

## 5. Dependencies & Requirements

### 5.1 Python Packages
```txt
boto3>=1.28.0           # AWS S3 SDK (R2 compatible)
python-dotenv>=1.0.0    # Environment variables
```

### 5.2 Infrastructure
- Cloudflare account with R2 enabled
- Docker & Docker Compose (for local development)
- VPS with public IP (for webhook endpoint)

### 5.3 Access Requirements
- Cloudflare R2 API token with Read/Write permissions
- B3 data access credentials
- VPS SSH access for deployment

---

## 6. Risk Assessment & Mitigation

### 6.1 Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| R2 API rate limits | High | Low | Implement retry logic with backoff |
| B3 download failures | High | Medium | Add comprehensive error handling |
| Local emulator differences | Medium | Medium | Use MinIO (high S3 compatibility) |
| Event notification delays | Low | Low | Implement fallback polling mechanism |
| Credential exposure | Critical | Low | Strict .gitignore, .env validation |

### 6.2 Rollback Strategy

**If issues occur in production:**
1. Revert to main branch
2. Redeploy previous version
3. Keep R2 files intact (read-only)
4. Analyze logs for root cause

**Rollback Commands:**
```bash
git checkout main
git branch -D feature/b3-r2-integration
# Redeploy previous version
```

---

## 7. Success Metrics

### 7.1 Technical Metrics
- ✅ 100% upload success rate
- ✅ < 5 second upload time for typical CSV
- ✅ Zero credential exposure incidents
- ✅ 100% test coverage for storage operations

### 7.2 Operational Metrics
- ✅ Successful automated processing via events
- ✅ < 1 minute end-to-end latency (upload → process)
- ✅ Zero data loss incidents
- ✅ Clear audit trail in logs

---

## 8. Acceptance Criteria

**This feature is considered complete when:**

1. ✅ All tasks (1-8) are completed with proper commits
2. ✅ Local MinIO testing passes for December 5, 2025
3. ✅ Production R2 upload verified manually
4. ✅ File naming follows yyyy-mm-dd.csv convention
5. ✅ Overwrite logic works correctly
6. ✅ All credentials in .env (not in code)
7. ✅ .gitignore properly configured
8. ✅ Documentation complete and accurate
9. ✅ Code review approved
10. ✅ Feature branch merged to main

---

## 9. Future Enhancements (Out of Scope)

**Phase 2 Considerations:**
- Automatic file retention policies (delete after N days)
- Data validation before upload
- Parallel processing of multiple files
- Cost optimization analysis
- Backup/disaster recovery strategy
- Multi-region R2 replication

---

## 10. Appendix

### 10.1 Reference Documentation
- [Cloudflare R2 Documentation](https://developers.cloudflare.com/r2/)
- [MinIO Documentation](https://min.io/docs/)
- [Boto3 S3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html)
- [B3 Data Documentation](http://www.b3.com.br/data/)

# Design

## System Architecture

```
┌─────────────────┐
│  Video Input    │
│ (Pre-recorded)  │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│           ML Processing Pipeline                │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Frame   │→ │  YOLO    │→ │ Risk Score   │  │
│  │ Extract  │  │ Detection│  │ Calculation  │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
└────────┬────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│              Backend API (Node.js)              │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Upload  │  │  Event   │  │   Alert      │  │
│  │ Endpoint │  │  Logger  │  │   Manager    │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
└────┬────────────────┬────────────────┬──────────┘
     │                │                │
     ▼                ▼                ▼
┌─────────┐    ┌──────────┐    ┌──────────┐
│  AWS S3 │    │PostgreSQL│    │ AWS SNS  │
│ (Frames)│    │  (Logs)  │    │ (Alerts) │
└─────────┘    └──────────┘    └──────────┘
                     │
                     ▼
              ┌─────────────┐
              │   React     │
              │  Dashboard  │
              │  (Leaflet)  │
              └─────────────┘
```

## Component Design

### 1. ML Processing Pipeline (Python)

#### Frame Extractor
- Input: video file path, extraction rate (fps)
- Output: sequence of frames with timestamps
- Library: OpenCV (cv2.VideoCapture)
- Current scope: pre-recorded video files
- Future scope: RTSP/WebRTC streams from live cameras

#### People Detection Module
- Architecture: CNN-based single-stage object detection (YOLOv8) for real-time person detection
- Model variants: YOLOv8n (nano) for speed, or YOLOv8m (medium) for accuracy
- Input: frame image (numpy array, H×W×3)
- Processing stages:
  1. CNN backbone: extracts hierarchical features from input frame using convolutional layers
  2. Detection head: predicts bounding boxes and class probabilities for detected objects
  3. Post-processing: applies confidence thresholding (>0.5) and non-maximum suppression
- Output: bounding boxes with confidence scores for class 'person'
- Crowd count: computed by counting detected bounding boxes belonging to class 'person' per frame
- Density estimation: count mapped to zone area to calculate people per square meter
- Evaluation: currently tested on pre-recorded video datasets
- Future: same architecture supports real-time camera feeds without modification

#### Zone Mapper
- Input: bounding box coordinates, zone polygon definitions
- Output: count per zone
- Logic: check if bbox center point falls within zone polygon

#### Risk Calculator
- Inputs:
  - Current density (count / zone_area)
  - Previous density (from last N frames)
  - Manual incident flags (from API)
- Formula:
  ```
  surge_rate = (current_density - prev_density) / time_delta
  base_score = density / capacity_limit
  surge_penalty = max(0, surge_rate * weight)
  incident_penalty = 0.3 if incident_flag else 0
  risk_score = min(1.0, base_score + surge_penalty + incident_penalty)
  ```
- Thresholds:
  - Low: risk_score < 0.4
  - Medium: 0.4 ≤ risk_score < 0.6
  - High: 0.6 ≤ risk_score < 0.8
  - Critical: risk_score ≥ 0.8

### 2. Backend API (Node.js + Express)

#### Endpoints

**POST /api/upload**
- Accept video file (multipart/form-data)
- Store in temp directory
- Trigger ML pipeline (spawn Python process)
- Return job_id for tracking

**GET /api/status/:job_id**
- Query processing status
- Return current frame count and progress

**GET /api/zones**
- Return list of configured zones with metadata
- Include current risk level and count

**POST /api/zones/:zone_id/incident**
- Manually flag incident for a zone
- Update risk calculation immediately

**GET /api/events**
- Query event logs with filters (time range, zone, risk level)
- Paginated results

**GET /api/alerts**
- Retrieve alert history
- Filter by zone or time range

#### Database Schema (PostgreSQL)

**zones**
- id (PK)
- name (varchar)
- polygon (geometry/json)
- area_sqm (float)
- capacity_limit (int)
- created_at (timestamp)

**events**
- id (PK)
- zone_id (FK)
- timestamp (timestamp)
- people_count (int)
- density (float)
- risk_score (float)
- risk_level (enum: low/medium/high/critical)
- frame_url (varchar, S3 path)

**alerts**
- id (PK)
- zone_id (FK)
- timestamp (timestamp)
- alert_type (varchar)
- message (text)
- acknowledged (boolean)

**incidents**
- id (PK)
- zone_id (FK)
- reported_at (timestamp)
- description (text)
- resolved (boolean)

### 3. Storage (AWS S3)

#### Bucket Structure
```
crowdshield-frames/
  ├── job_<id>/
  │   ├── frame_0001.jpg
  │   ├── frame_0002.jpg
  │   └── metadata.json
  └── evidence/
      └── alert_<timestamp>_zone_<id>.jpg
```

#### Lifecycle Policy
- Transition to Glacier after 30 days
- Delete after 90 days (configurable)

### 4. Alert System (AWS SNS)

#### Topics
- critical-alerts: immediate notification
- high-risk-alerts: batched every 5 minutes
- daily-summary: end-of-day report

#### Subscription Types
- Email for operators
- SMS for emergency contacts (future)
- Webhook for dashboard real-time updates

### 5. Frontend Dashboard (React)

#### Components

**MapView**
- Leaflet map with OpenStreetMap tiles
- Zone overlays with color coding:
  - Green: Low risk
  - Yellow: Medium risk
  - Orange: High risk
  - Red: Critical risk
- Click zone to view details

**ZoneDetails Panel**
- Current count and density
- Risk score gauge
- Trend chart (last 10 minutes)
- Incident flag button

**AlertList**
- Real-time alert feed
- Acknowledge button
- Link to evidence frame

**HistoricalPlayback**
- Timeline scrubber
- Play/pause controls
- Speed adjustment

#### State Management
- React Context for global state
- Polling API every 3 seconds for updates
- WebSocket upgrade (future enhancement)

## Data Flow

### Video Processing Flow
1. Operator uploads video via dashboard
2. Backend stores file and creates job record
3. Python pipeline spawned with job_id
4. For each extracted frame:
   - Run CNN-based object detection (YOLOv8) to detect persons
   - Count detected bounding boxes for class 'person'
   - Map detections to zones
   - Calculate risk scores
   - Store frame to S3
   - Insert event record to database
   - If risk ≥ High, publish SNS alert
5. Job marked complete, cleanup temp files

### Real-Time Monitoring Flow
1. Dashboard polls /api/zones every 3 seconds
2. Backend queries latest events per zone
3. Return current status with risk levels
4. Dashboard updates map colors and charts
5. New alerts trigger browser notification

## Security Design

### Authentication
- API key in request header: `X-API-Key`
- JWT tokens for future multi-user support
- Bcrypt for password hashing (if user accounts added)

### AWS IAM Policy (Least Privilege)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::crowdshield-frames/*"
    },
    {
      "Effect": "Allow",
      "Action": ["sns:Publish"],
      "Resource": "arn:aws:sns:*:*:crowdshield-*"
    }
  ]
}
```

### Environment Variables
- DATABASE_URL: PostgreSQL connection string
- AWS_ACCESS_KEY_ID: IAM user key
- AWS_SECRET_ACCESS_KEY: IAM secret
- AWS_REGION: deployment region
- S3_BUCKET_NAME: frame storage bucket
- SNS_TOPIC_ARN: alert topic
- API_KEY: dashboard authentication
- PORT: server port (default 3000)

## Deployment Architecture (MVP)

### Local Development
- Python virtual environment for ML pipeline
- Node.js server on localhost:3000
- PostgreSQL in Docker container
- React dev server on localhost:5173
- AWS services via SDK with local credentials

### Cloud Deployment (Future)
- EC2 instance for backend + ML pipeline
- RDS PostgreSQL for database
- S3 for frame storage
- CloudFront for dashboard CDN
- Lambda for serverless processing (optional)

## Performance Optimization

### ML Pipeline
- Batch frame processing (process 10 frames at once)
- CNN inference optimization: model quantization for faster execution
- GPU acceleration if available (CUDA for convolutional operations)
- Skip frames if processing falls behind (drop to 0.5 fps)
- Same CNN architecture supports both pre-recorded and live video input

### Backend
- Database connection pooling (pg-pool)
- Redis cache for zone status (future)
- Async processing with job queue (Bull/BullMQ)

### Frontend
- Lazy load historical data
- Debounce map interactions
- Compress images before display
- Service worker for offline capability (future)

## Testing Strategy

### Model Evaluation
- Test dataset: 100 labeled frames from public crowd datasets
- Metrics: MAE, RMSE, processing time
- Compare against manual counts

### API Testing
- Unit tests with Jest
- Integration tests with Supertest
- Mock AWS services with aws-sdk-mock

### End-to-End Testing
- Selenium for dashboard workflows
- Test video samples with known crowd counts
- Verify alert delivery

## Monitoring & Logging

### Application Logs
- Winston logger for Node.js
- Python logging module for ML pipeline
- Log levels: ERROR, WARN, INFO, DEBUG

### Metrics to Track
- Processing latency per frame
- API response times
- Database query performance
- S3 upload success rate
- Alert delivery rate

### Error Handling
- Retry logic for AWS API calls (exponential backoff)
- Graceful degradation if ML pipeline fails
- User-friendly error messages in dashboard

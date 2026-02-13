# Requirements

## Functional Requirements

### Core Features (MVP)
- Accept pre-recorded video/image input from CCTV or drone footage
- Extract frames at configurable intervals (e.g., 1-2 fps)
- Detect and count people per defined zone using YOLO-based model
- Calculate crowd density (people per square meter) for each zone
- Compute risk score based on:
  - Current density level
  - Rate of density change (surge detection)
  - Manual incident flags (e.g., blocked exit)
- Classify zones as Low/Medium/High/Critical risk
- Store processed frames and metadata (timestamp, count, risk level)
- Display real-time dashboard with:
  - Map view with zone overlays
  - Color-coded risk status per zone
  - Trend graphs (density over time)
- Send alerts via AWS SNS when risk threshold exceeded
- Log all events with evidence frames for audit

### Phase 1: Model Pipeline
- Implement people detection using YOLOv8 or similar
- Evaluate counting accuracy on labeled test dataset
- Measure MAE and RMSE against ground truth
- Optimize for speed vs accuracy tradeoff

### Phase 2: Backend & Storage
- REST API for video upload and processing
- PostgreSQL database for event logs and zone metadata
- AWS S3 integration for frame storage
- Basic authentication (API key or JWT)

### Phase 3: Dashboard & Alerts
- React-based operator dashboard
- Leaflet map integration with zone boundaries
- Real-time status updates (simulated polling)
- SNS alert configuration (email/SMS)
- Historical playback of events

### Phase 4: Future Scope
- Live camera feed integration via RTSP/WebRTC
- Multi-camera synchronization
- Predictive analytics using time-series models
- Mobile app for field operators

## Non-Functional Requirements

### Performance
- Frame processing latency: <3 seconds per frame (MVP target)
- Dashboard update frequency: every 2-5 seconds
- Support concurrent processing of multiple video sources

### Accuracy
- People counting MAE: <10% on test dataset
- False alert rate: <15% (precision target)
- Missed critical event rate: <5% (recall target)

### Security
- No hardcoded credentials in codebase
- Environment variables for all secrets (.env)
- Least-privilege IAM policies for AWS resources
- API authentication required for all endpoints
- HTTPS for production deployment

### Scalability
- Handle 5-10 concurrent video streams (MVP)
- Database design supports 100K+ event records
- S3 storage with lifecycle policies for cost management

### Usability
- Dashboard accessible via web browser
- Minimal training required for operators
- Clear visual indicators for risk levels
- Export logs in CSV format

## Data Requirements

### Input Data
- Video formats: MP4, AVI, MOV
- Image formats: JPEG, PNG
- Minimum resolution: 720p
- Frame rate: 15-30 fps (source video)

### Zone Configuration
- Define zones as polygons on map coordinates
- Assign capacity limits per zone
- Configure risk thresholds (density, surge rate)

### Output Data
- Event logs: timestamp, zone_id, count, density, risk_level
- Alert records: timestamp, zone_id, alert_type, message
- Stored frames: S3 URLs with metadata tags

## System Constraints

### MVP Limitations
- Pre-recorded video only (no live streaming)
- Manual zone definition (no auto-detection)
- Single-user dashboard (no multi-tenancy)
- Basic alert rules (threshold-based only)

### Technical Dependencies
- Python 3.9+ for ML pipeline
- Node.js 18+ for backend
- PostgreSQL 14+ or Supabase
- AWS account with S3 and SNS access
- Modern web browser for dashboard

## Evaluation Metrics

### Model Performance
- Mean Absolute Error (MAE) for people count
- Root Mean Square Error (RMSE) for people count
- Processing time per frame (milliseconds)

### Alert Quality
- True positives: correctly identified high-risk events
- False positives: incorrect alerts
- False negatives: missed critical events
- Precision = TP / (TP + FP)
- Recall = TP / (TP + FN)

### System Performance
- End-to-end latency (upload to alert)
- Dashboard response time
- Storage costs per hour of video

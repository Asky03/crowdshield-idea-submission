# CrowdShield

AI-based Crowd Risk Monitoring and Early Warning System

## Overview

CrowdShield analyzes crowd density from video footage to detect high-risk situations and alert operators before incidents occur. The MVP processes pre-recorded videos using YOLO-based people detection and provides a real-time dashboard with zone-based risk assessment.

## Problem Statement

Large public gatherings in India (festivals, rallies, college events) face crowd management challenges. Manual CCTV monitoring is reactive and prone to delays. Stampedes and crowd surges can escalate quickly without early warning systems.

## Solution

- Automated people counting per zone using computer vision
- Risk scoring based on density, surge rate, and incident flags
- Real-time dashboard with map visualization
- Automated alerts via AWS SNS when thresholds exceeded
- Evidence logging for post-event analysis

## Tech Stack

- ML: Python, OpenCV, YOLOv8
- Backend: Node.js, Express, PostgreSQL
- Frontend: React, Leaflet (OpenStreetMap)
- Cloud: AWS S3, AWS SNS
- Deployment: Local (MVP), EC2 + RDS (future)

## Repository Structure

```
crowdshield/
├── requirements.md      # Functional and non-functional requirements
├── design.md           # System architecture and component design
├── README.md           # This file
├── .gitignore          # Git ignore patterns
└── .env.example        # Environment variable template
```

## MVP Scope

Phase 1: ML pipeline with accuracy evaluation
Phase 2: Backend API, database, storage integration
Phase 3: Dashboard, alerts, event logging
Phase 4: Live camera feed support (future)

## Evaluation Metrics

- Counting accuracy: MAE and RMSE on labeled test data
- Alert precision and recall
- Processing latency: <3 seconds per frame target

## Security

- No credentials in code (use .env files)
- Least-privilege AWS IAM policies
- API key authentication for dashboard
- HTTPS for production deployment

## Getting Started

This is an idea submission repository for the AWS AI for Bharat Hackathon. Implementation details and setup instructions will be added during development phases.

## License

To be determined

## Contact

Project team contact information to be added

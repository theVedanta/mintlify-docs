# API Documentation for Project Sentinel

## Overview

Project Sentinel is a command & control backend designed for resource management of Avengers. It provides various endpoints for managing resources, processing intel reports, forecasting future resource levels, and offers real-time event updates via server-sent events (SSE).

## Table of Contents

- [1. Base URL](#1-base-url)
- [2. Middleware](#2-middleware)
- [3. Health Checks](#3-health-checks)
- [4. Server-Sent Events](#4-server-sent-events)
- [5. Routers](#5-routers)
  - [5.1 Admin Router](#51-admin-router)
  - [5.2 Forecast Router](#52-forecast-router)
  - [5.3 Reports Router](#53-reports-router)
  - [5.4 Resources Router](#54-resources-router)
  - [5.5 Retell Webhook Router](#55-retell-webhook-router)
- [6. Data Models](#6-data-models)

## 1. Base URL

```
http://localhost:8000
```

## 2. Middleware

### CORS Middleware

CORS (Cross-Origin Resource Sharing) is enabled for allowing cross-origin requests. The configuration allows all methods and headers from specified origins defined in the application settings.

## 3. Health Checks

### GET /

Returns the health status of the service.

**Response:**

```json
{
    "status": "ok",
    "service": "Project Sentinel API",
    "sse_clients": <number_of_clients>
}
```

### GET /health

Check the health status:
**Response:**

```json
{
  "status": "healthy"
}
```

## 4. Server-Sent Events

### GET /stream

Connect to receive real-time events about reports being processed.

**Event Types:**

- `connected` : A confirmation of client connection.
- `report.received` : A report has been received for processing.
- `report.processed` : A report processing has completed.
- `report.failed` : A report processing has failed.

### Connection Example

```javascript
const evtSource = new EventSource("http://localhost:8000/stream");
evtSource.onmessage = function (event) {
  console.log(event.data);
};
```

## 5. Routers

### 5.1 Admin Router

**Base route:** `/admin`

1. **POST /admin/reset-and-seed**
   Resets the database by truncating tables and seeding from CSV and JSON files.

   **Response Example:**

   ```json
   {
       "status": "ok",
       "resource_logs_inserted": <number>,
       "intel_reports_inserted": <number>
   }
   ```

### 5.2 Forecast Router

**Base route:** `/forecast`

1. **GET /forecast**
   Returns a forecast based on recent stock readings.

   **Query Parameters:**
   - `sector` (required): Sector ID (e.g., `Wakanda`)
   - `resource_type` (required): Resource type (e.g., `Vibranium (kg)`)
   - `window` (optional): Number of recent readings to fetch (default `200`).
   - `tail` (optional): Number of recent readings to use in regression (default `48`).

   **Response Example:**

   ```json
   {
       "sector": "Wakanda",
       "resource_type": "Vibranium (kg)",
       "current_stock": <current_stock_value>,
       "recommended_exhaustion_date": "YYYY-MM-DDTHH:MM:SS",
       "full_series_forecast": { ... },
       "post_snap_forecast": { ... }
   }
   ```

2. **GET /forecast/summary**
   Provides a summary of urgency across all sector-resource combinations.

   **Response Example:**

   ```json
   {
       "count": <total_count>,
       "summaries": [
           {
               "sector": <sector_id>,
               "resource_type": <resource_type>,
               "urgency": <urgency_level>,
               "predicted_exhaustion_date": "YYYY-MM-DDTHH:MM:SS"
           },
           ...
       ]
   }
   ```

### 5.3 Reports Router

**Base route:** `/reports`

1. **GET /reports**
   Returns a list of all processed intel reports.

   **Query Parameters:**
   - `status` (optional): Filter reports by status (`pending`, `processed`, `failed`).
   - `priority` (optional): Filter reports by priority.

   **Response Example:**

   ```json
   [
      {
          "id": <id>,
          "report_id": <report_id>,
          "hero_alias": <hero_alias>,
          "raw_text": <report_text>,
          "timestamp": "YYYY-MM-DDTHH:MM:SS",
          "status": <status>,
          "extracted_sector": <sector>,
          "extracted_resource": <resource>
      },
      ...
   ]
   ```

2. **GET /reports/{report_id}**
   Returns a single report by its UUID.

3. **POST /reports/**
   Submits a new report for processing.

   **Request Body Example:**

   ```json
   {
     "report_id": "<unique_report_id>",
     "hero_alias": "<alias>",
     "secure_contact": "<contact>",
     "raw_text": "<report_text>",
     "timestamp": "YYYY-MM-DDTHH:MM:SS",
     "priority": "<priority>"
   }
   ```

   **Response Example:**

   ```json
   {
       "status": "processed",
       "report_id": "<report_id>",
       "redacted_text": "<redacted_text>",
       "extracted": { ... }
   }
   ```

### 5.4 Resources Router

**Base route:** `/resources`

1. **GET /**
   Returns historical resource stock levels.

   **Query Parameters:**
   - `sector` (optional): Filter by sector ID.
   - `resource_type` (optional): Filter by resource type.
   - `limit` (optional): The maximum number of logs to return (default `1000`, max `10000`).

   **Response Example:**

   ```json
   {
       "sector": <sector_id>,
       "resource_type": <resource_type>,
       "count": <number_of_logs>,
       "data": [ { ... }, ... ]
   }
   ```

2. **POST /**
   Creates a new resource reading.

   **Request Body Example:**

   ```json
   {
       "timestamp": "YYYY-MM-DDTHH:MM:SS",
       "sector_id": "<sector_id>",
       "resource_type": "<resource_type>",
       "stock_level": <stock_level>,
       "usage_rate_hourly": <usage_rate>,
       "snap_event_detected": <boolean>
   }
   ```

### 5.5 Retell Webhook Router

**Base route:** `/webhooks/retell`

1. **POST /**
   Receives incoming webhook events for analyzed calls.

   Accepts payloads from the Retell AI, and triggers reports in the system.

   **Response Codes:**
   - 204: Successfully processed without actions.
   - 400: Invalid JSON payload.

   **Expected JSON Payload:**

   ```json
   {
       "event": "call_analyzed",
       "call": { ... }
   }
   ```

## 6. Data Models

### IntelReport

```python
class IntelReport(Base):
    __tablename__ = 'intel_reports'
    id = Column(Integer, primary_key=True, index=True)
    report_id = Column(String, unique=True, index=True)
    hero_alias = Column(String)
    secure_contact = Column(String)
    raw_text = Column(Text)
    redacted_text = Column(Text, nullable=True)
    timestamp = Column(DateTime)
    priority = Column(String)
    status = Column(String)
    extracted_sector = Column(String, nullable=True)
    extracted_resource = Column(String, nullable=True)
    extracted_urgency = Column(String, nullable=True)
    extracted_summary = Column(Text, nullable=True)
    extracted_action = Column(Text, nullable=True)
```

### ResourceLog

```python
class ResourceLog(Base):
    __tablename__ = 'resource_logs'
    id = Column(Integer, primary_key=True, index=True)
    timestamp = Column(DateTime)
    sector_id = Column(String)
    resource_type = Column(String)
    stock_level = Column(Float)
    usage_rate_hourly = Column(Float)
    snap_event_detected = Column(Boolean)
```

This concludes the basic API documentation for Project Sentinel. Each router within the API serves a specific purpose related to the management and forecasting of resources, allowing for seamless real-time updates and interactions with incoming data from various sources.

---

_Documentation generated by_ **[AutoCodeDocs.ai](https://autocodedocs.ai)**

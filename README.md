# Product Requirement Document: .NET API for Quarter-Hour Energy Time Series (CQRS/Event Sourcing, All UTC, Timestamped Values)

## Problem Statement

Organizations in the energy sector require a robust, auditable API to store and manage quarter-hourly (15-minute interval) time series data, measured in megawatts (MW), for various assets, sites, or points of measurement. Current solutions often lack support for event-based updates, category tagging, explicit timestamping of each value, and the ability to reconstruct historical states of the data as of any point in time, creating compliance risks and operational inefficiencies.

## Goals and Objectives

**Goals:**
- Deliver a reliable .NET API for handling quarter-hourly energy time series data in MW using CQRS and Event Sourcing patterns.
- Support efficient ingestion, partial updates as immutable events, and querying of time series state as of any historical point.

**Objectives:**
- Store every update as a new event, each with an arrival timestamp, containing only the data that just arrived.
- Each value in an update is represented as an object with a decimal `value` and a `dateTimeUtc` timestamp (ISO 8601, UTC).
- Enable queries that reconstruct the time series as it would have appeared at any specified queryTime by replaying all relevant events.
- Enforce data integrity, category tagging, and clear unit handling (MW).
- All time-related fields and operations use UTC exclusively.
- Provide comprehensive documentation and integration samples.

## Use Cases

- Ingest a full or partial day’s quarter-hour readings for a category, with each value explicitly timestamped in UTC and stored as an immutable event.
- Retrieve all 96 quarter-hour values (each with its own `dateTimeUtc`) for a given category, and timespan, as of a specific queryTime (for audit or reporting), by replaying events up to that time.
- Update only a subset of quarter-hour values for a day; each update is a new event and does not overwrite previous data.
- Handle explicit nulls in updates to indicate "no change" for those intervals.
- Aggregate quarter-hour data (e.g., sum, average) as of a given queryTime for reporting or operational decisions.
- Distinguish between multiple categories (e.g., "solar", "wind", "consumption") for the same identifier.

## Key Features

- **Event Sourcing for Updates**
  - Every update is stored as a new, immutable event with a system-generated `arrivalTimestampUtc` (ISO 8601, UTC).
  - No data is overwritten; all changes are append-only.

- **Partial and Bulk Updates with Timestamped Values**
  - Each event contains a list of value objects:
    - `value`: decimal (MW) or null (null means "do not change" for this interval)
    - `dateTimeUtc`: ISO 8601 UTC timestamp for the specific quarter-hour interval
  - Only specified intervals are updated; others remain unchanged in the reconstructed state.

- **Category Support**
  - Every time series is associated with a required `category` string.
  - All endpoints require category as a parameter.

- **Query by queryTime**
  - Optional `queryTime` parameter on queries (UTC).
  - API reconstructs the time series as it would have been after applying only events with `arrivalTimestampUtc` <= `queryTime`.
  - If `queryTime` is omitted, the latest state is returned.
  - a query never returns null. null values are returned a 0

- **Default and Null Handling**
  - If no data exists for a day, all 96 values default to 0.00 MW with the appropriate `dateTimeUtc`.
  - Missing values in the reconstructed state are returned as 0.00 MW in the response.

- **RESTful API Design**
  - Standardized endpoints for create (event) and  query operations

- **Comprehensive Documentation**
  - Clear API documentation and sample code api calls
  - http files with sample data


## Assumptions

- All quarter-hour values are decimals in MW; precision is at least two decimal places.
- Each value object contains a `value` (decimal or null) and `dateTimeUtc` (ISO 8601, UTC).
- Each day consists of exactly 96 quarter-hour intervals (00:00 to 23:45 UTC). 1 day has 92 and one 100 quarter-hour intervals
- category names are unique and consistent.
- All time fields are UTC (ISO 8601 format with "Z" suffix).
- Time zones and daylight saving are handled upstream or via metadata.
- Storage backend can efficiently handle high-frequency, append-only event data.

## Example API Behaviors

- **Update Request Example (Event):**
  ```json
  {
    "category": "consumption",
    "values": [
      { "value": 1.25, "dateTimeUtc": "2025-07-09T06:00:00Z" },
      { "value": null, "dateTimeUtc": "2025-07-09T06:15:00Z" },
      { "value": 1.30, "dateTimeUtc": "2025-07-09T06:30:00Z" }
      // ... up to 96 entries for the day
    ]
  }
  ```
  - Stored as a new event with system-generated `arrivalTimestampUtc` (UTC).
  - Nulls in the `value` property mean the existing value for that interval is not changed when reconstructing state.
  - if a entry of an quarter hour is not present it counts as null
  - values are never negative

- **Query Request Example:**
  ```json
  {
    "category": "consumption",
    "date": "2025-07-09",
    "queryTime": "2025-07-09T10:30:00Z"
  }
  ```
  - API reconstructs the state by replaying only events with `arrivalTimestampUtc` <= `queryTime` (all times UTC).

- **Query Response Example:**
  ```json
  {
    "category": "consumption",
    "date": "2025-07-09",
    "values": [
      { "value": 1.25, "dateTimeUtc": "2025-07-09T06:00:00Z" },
      { "value": 0.00, "dateTimeUtc": "2025-07-09T06:15:00Z" },
      { "value": 1.30, "dateTimeUtc": "2025-07-09T06:30:00Z" }
      // ... up to 96 entries for the day
    ]
  }
  ```
  - Each value object includes both the MW value and its UTC timestamp.

All time-related fields and operations in this API are strictly UTC, and each value is explicitly timestamped, ensuring consistency, traceability, and compliance across all integrations and use cases.


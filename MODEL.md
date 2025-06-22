# Age of Kindness - Data & API Model

This document outlines the core data models, architectural principles, and API endpoints for the Age of Kindness backend platform. It serves as the single source of truth for the system's structure.

## Core Theories & Philosophy

1.  **The Possibility of Delivery:** Every message submitted to the platform is a candidate for delivery. Its journey is a process, not a simple transaction.

2.  **The Vault (Time-Locked Privacy):** A message can contain "sealed" content, inaccessible until a specific `deliveryTime` is reached.

3.  **The Multipart Message:** A message is a structured object composed of multiple parts (unsealed for routing, sealed for private content).

4.  **Unified Identity:** A message has **one single, unique ID** (`messageId`) for its entire lifecycle, from submission to expiration. This ID is used as the document key in both the vetting log and the public messages collection, providing a direct, traceable link.

## Data Collections (Firestore)

The system is built around three core collections.

### 1. `messages_vetting_log`

The intake queue and permanent, private audit log for every submission.

-   **Purpose:** To reliably ingest all user submissions, track their journey through the AI vetting pipeline, and provide a permanent record for administrative review.
-   **Document ID:** The unique `messageId` generated upon submission.

**Schema (`messages_vetting_log`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `messageId` | `string` | The single, unique identifier for this message's entire lifecycle. |
| `submissionTime`| `Timestamp` | The moment the user hit send. |
| `payload` | `map` | The raw, unvetted payload from the client. |
| `vettingStatus`| `string` | `"queued"`, `"in_progress"`, `"approved"`, `"rejected"`. |
| `vettingDetails`| `map` | (Phase 1.1) Raw AI scores and rejection reasons. |
| `processedTime` | `Timestamp`| When the vetting process completed. |

---

### 2. `messages`

The public collection of approved, visible messages.

-   **Purpose:** To store all messages that have passed vetting and are currently "in-flight" or "at rest." This is the primary collection queried by the Unity client.
-   **Document ID:** The **same `messageId`** from the `messages_vetting_log`. This creates a direct 1:1 link.

**Schema (`messages`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `messageId` | `string` | The single, unique identifier for this message. |
| `status` | `string` | `"in_transit"`, `"delivered"`. |
| `broadcastTime` | `Timestamp` | When the message passed vetting and became active. |
| `deliveryTime` | `Timestamp` | The target moment of delivery completion. |
| `deliveryLifespan` | `number` | The journey's duration in seconds. |
| `postDeliveryLifespan`| `number` | How long the message is viewable post-delivery. |
| `expirationTime` | `Timestamp` | `deliveryTime + postDeliveryLifespan`. Used for TTL. |
| `origin` | `map` | Geo-data for the message origin. |
| `destination` | `map` | Geo-data for the message destination. |
| `originGeohash` | `string` | For efficient geospatial queries. |
| `destinationGeohash` | `string` | For efficient geospatial queries. |
| `route` | `map` | A structured object describing the journey's `legs`. |
| `content` | `map` | The structured, multipart message content itself. |

#### `content` Object Schema

| Field | Type | Description |
| :--- | :--- | :--- |
| `unsealed` | `map` | Publicly visible content, available immediately. |
| `sealed` | `map` | (Nullable) Private content, inaccessible until `deliveryTime`. |
| `vetting` | `map` | The public "Certificate of Kindness" from AI moderation. |

---

### 3. `location_stats`

*(Schema and purpose unchanged.)*

---

## API Endpoints (Cloud Functions)

### `/message` [POST]

The primary ingestion endpoint for all new messages.

-   **Action:**
    1.  Generates a new, unique `messageId`.
    2.  Accepts a raw message payload from a user.
    3.  Creates a new document in the `messages_vetting_log` collection using the `messageId` as the document key.
-   **Response:** `202 Accepted` with the `messageId`. This allows the client to potentially track the submission status later, if needed.

---

### `/revalidate` [POST] (Future)

-   **Theory:** An administrative endpoint to re-validate a message.
-   **Request Body:** `{ "messageId": "..." }`

---

### `/interact` [POST] (Future)

-   **Theory:** A generic endpoint for users to interact with a delivered message.
-   **Request Body:** `{ "messageId": "...", "interactionType": "like" }`

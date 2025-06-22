# Age of Kindness - Data & API Model

This document outlines the core data models, architectural principles, and API endpoints for the Age of Kindness backend platform. It serves as the single source of truth for the system's structure.

## Core Theories & Philosophy

1.  **The Possibility of Delivery:** Every message submission is a candidate for delivery, guided through a multi-stage lifecycle.

2.  **The Vault (Time-Locked Privacy):** Messages can contain "sealed" content, guaranteed private until a future `deliveryTime`.

3.  **The Multipart Message:** A message is a structured object with `unsealed` (public) and `sealed` (private) parts.

4.  **Unified Identity:** A message has one single, unique `messageId` for its entire lifecycle, used as the document key in all related collections.

5.  **Staged Vetting:** Content moderation is a multi-stage process. A message's `unsealed` portion is vetted at submission time, while the `sealed` portion is vetted just before its final delivery.

## Data Collections (Firestore)

### 1. `messages_vetting_log`

The permanent, private audit log for every submission.

-   **Document ID:** The unique `messageId`.
-   **Purpose:** To reliably ingest all user submissions, track their journey through the AI vetting pipeline, and provide a permanent record for administrative review.

**Schema (`messages_vetting_log`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `messageId` | `string` | The single, unique identifier for this message's lifecycle. |
| `submissionTime`| `Timestamp` | The moment the user submitted the message. |
| `payload` | `map` | The raw, unvetted payload from the client. |
| `status` | `string` | `"queued"`, `"in_progress"`, `"route_approved"`, `"pre_delivery_vetting_required"`, `"delivered_approved"`, `"rejected"`, `"dormant"` |
| `vettingHistory`| `array` | (Phase 1.1) An array of vetting attempt objects. |
| `processedTime` | `Timestamp`| When the *last* processing step was completed. |

---

### 2. `messages`

The public collection of approved, visible messages.

-   **Document ID:** The **same `messageId`** from the log, creating a direct 1:1 link.
-   **Purpose:** To store all messages that have passed initial vetting and are currently visible to users. This is the primary collection queried by the Unity client.

**Schema (`messages`):**

| Field | Type | Description |
| :--- | :--- | :--- |
| `messageId` | `string` | The single, unique identifier for this message. |
| `status` | `string` | `"in_transit"` (unsealed part visible), `"delivered"` (fully visible). |
| `broadcastTime` | `Timestamp` | When the message passed *initial* vetting. |
| `deliveryTime` | `Timestamp` | The target moment of final delivery/unsealing. |
| `deliveryLifespan` | `number` | The journey's duration in seconds. |
| `postDeliveryLifespan`| `number` | How long the message is viewable post-delivery. |
| `expirationTime` | `Timestamp` | Used for Firestore's automatic TTL deletion. |
| `origin` | `map` | Geo-data for the message origin. |
| `destination` | `map` | Geo-data for the message destination. |
| `originGeohash` | `string` | For efficient geospatial queries. |
| `destinationGeohash` | `string` | For efficient geospatial queries. |
| `route` | `map` | A structured object describing the journey's `legs`. |
| `content` | `map` | The structured, multipart message content itself. |

#### `content` Object Schema

| Field | Type | Description |
| :--- | :--- | :--- |
| `unsealed` | `map` | Publicly visible part, vetted at submission time. |
| `sealed` | `map` | (Nullable) Private part, vetted at `deliveryTime`. |
| `vetting` | `map` | The public "Certificate of Kindness" from the *initial* vetting. |

---

### 3. `location_stats`

An aggregated collection for efficient "hotzone" mapping.
*(Schema and purpose defined previously. Updated via backend triggers, not direct API calls.)*

---

## API Endpoints (Cloud Functions)

All endpoints are hosted as serverless functions and are prefixed with `/api`.

### **`/message`**

#### `POST /api/message`

The primary ingestion endpoint for all new messages. This endpoint is designed for speed and reliability, immediately queueing the message for asynchronous processing.

-   **Action:**
    1.  Generates a new, unique `messageId`.
    2.  Validates the basic structure of the incoming request body.
    3.  Creates a new document in the `messages_vetting_log` collection using the `messageId` as the document key. The initial `status` is set to `"queued"`.
-   **Request Body (`application/json`):**
    ```json
    {
      "origin": { "latitude": 48.85, "longitude": 2.35, "name": "Paris, FR" },
      "destination": { "latitude": 35.68, "longitude": 139.69, "name": "Tokyo, JP" },
      "route": {
        "legs": [{ "agentId": "aok_arc_v1" }]
      },
      "content": {
        "unsealed": {
          "mimeType": "text/plain",
          "data": "A message of kindness"
        },
        "sealed": {
          "mimeType": "text/plain",
          "data": "This is the secret message for the future.",
          "encryption": "none" // In V1, we store it plainly on the backend.
        }
      },
      "requestedDeliveryTimestamp": 1788294400 // Optional: For future-dated messages
    }
    ```
-   **Success Response (`202 Accepted`):**
    ```json
    {
      "status": "queued",
      "messageId": "unique-message-id-12345"
    }
    ```
-   **Error Response (`400 Bad Request`, `500 Internal Server Error`):**
    ```json
    {
      "error": "Invalid request body: 'origin' field is missing."
    }
    ```

---

### **`/revalidate`** (Future Scope)

#### `POST /api/revalidate`

An administrative endpoint to trigger a re-validation of a previously rejected or flagged message.

-   **Theory:** Useful for correcting AI moderation errors or re-evaluating messages against updated policies. This would likely be a protected endpoint requiring admin authentication.
-   **Request Body (`application/json`):**
    ```json
    {
      "messageId": "the-rejected-message-id-67890"
    }
    ```
-   **Success Response (`202 Accepted`):**
    ```json
    {
      "status": "re-queued for vetting",
      "messageId": "the-rejected-message-id-67890"
    }
    ```
-   **Status:** Not implemented in Phase 1.0.

---

### **`/interact`** (Future Scope)

#### `POST /api/interact`

A generic endpoint for users to interact with a delivered message, providing a flexible foundation for future social features without modifying the core `Message` document.

-   **Theory:** Could be used for "liking," reporting, bookmarking, or leaving reactions. Each interaction would create a new document in a separate `interactions` collection, linked by `messageId`.
-   **Request Body (`application/json`):**
    ```json
    {
      "messageId": "public-message-id-abcde",
      "interactionType": "like"
      // "userId": "anonymous-or-auth-user-id" // Optional user identifier
    }
    ```
-   **Success Response (`201 Created`):**
    ```json
    {
      "status": "interaction recorded",
      "interactionId": "unique-interaction-id-fghij"
    }
    ```
-   **Status:** Not implemented in Phase 1.0.

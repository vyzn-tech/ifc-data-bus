# IFC Data Bus Message Format Review & Improvement Proposals

**Review Date:** November 21, 2025  
**Reviewed By:** AI Assistant  
**Version:** 0.2

## Executive Summary

The current message format provides a solid foundation for event-driven IFC data exchange using Commands, Events, and Queries. However, there are opportunities to enhance **consistency**, **observability**, **error handling**, and **developer experience**.

---

## Current Strengths ✅

1. **Clear Message Type Taxonomy** - Well-defined Command/Event/Query pattern
2. **Correlation Support** - `correlationId` enables request-response tracking
3. **Sequence Numbers** - Proper ordering for events (Model.Publish, Model.Change)
4. **Model Versioning** - Strong revision tracking with hash verification
5. **Security Foundation** - Token-based authentication with permissions
6. **IFC5 Integration** - Leverages IFC5 paths and entity references
7. **Temporal Tracking** - ISO 8601 timestamps throughout

---

## Critical Issues 🔴

### 1. Inconsistent Sequence Number Usage

**Problem:** Only some messages have `sequenceNumber` fields.

**Current State:**
- ✅ `Model.Publish` - Has sequence numbers
- ✅ `Model.Change` - Has sequence numbers  
- ✅ `Event.BuildingLcaCalculated` - Has sequence numbers
- ❌ `Command.CalculateBuildingLca` - Missing sequence numbers
- ❌ `Query.*` - Missing sequence numbers
- ❌ `QueryResponse.*` - Missing sequence numbers

**Why This Matters:**
Based on `WHY_SEQUENCE_NUMBERS.md` and `MQTT_AND_SEQUENCE_NUMBERS.md`, sequence numbers are critical for:
- Detecting missing messages
- Ensuring correct processing order
- Event replay capability
- Conflict detection

**Recommendation:**
```diff
+ Add sequence numbers to ALL message types that create state changes
+ Keep sequence numbers scoped per message stream (per modelId)
```

**Example - Enhanced Command:**
```json
{
  "sequenceNumber": 50,
  "previousSequenceNumber": 49,
  "messageType": "de.din.DIN_EN15978_2012_10.Command.CalculateBuildingLca",
  ...
}
```

### 2. Missing MQTT Topic Routing Information

**Problem:** Messages don't specify which MQTT topic they should be published to or where responses should be sent.

**Why This Matters:**
- Consumers need to know where to publish responses
- Enables point-to-point replies for Queries
- Prevents message loops

**Recommendation:**
Add `routing` field to message envelope:

```json
{
  "routing": {
    "publishedOn": "ifc-data-bus/model/bim-project-123/change",
    "replyTo": "ifc-data-bus/reply/SystemD/query-lca-001",
    "ttl": 300
  }
}
```

### 3. Weak Error Handling Structure

**Problem:** 
- Error codes are ad-hoc strings (`"MODEL_NOT_FOUND"`, `"LCA_RESULTS_NOT_FOUND"`)
- No error code registry or documentation
- No severity levels
- Missing user-friendly error messages vs. technical details

**Current Error Format:**
```json
{
  "error": {
    "code": "MODEL_NOT_FOUND",
    "message": "Model 'bim-project-123' revision 2 not found",
    "details": { ... }
  }
}
```

**Recommendation:**
Standardize error structure with severity, categories, and actionable information:

```json
{
  "error": {
    "code": "MODEL_NOT_FOUND",
    "category": "NotFound",
    "severity": "error",
    "message": "Model 'bim-project-123' revision 2 not found",
    "userMessage": "The requested building model version could not be found. Please verify the model ID and revision number.",
    "details": {
      "modelId": "bim-project-123",
      "revision": 2,
      "availableRevisions": [1]
    },
    "retryable": false,
    "helpUrl": "https://docs.ifc-data-bus.org/errors/MODEL_NOT_FOUND"
  }
}
```

---

## Important Issues 🟡

### 4. Missing Distributed Tracing Support

**Problem:** No support for distributed tracing (OpenTelemetry standard).

**Why This Matters:**
- Difficult to debug issues across multiple systems
- Can't trace message flow through the entire pipeline
- No performance monitoring capabilities

**Recommendation:**
Add OpenTelemetry-compatible tracing fields:

```json
{
  "tracing": {
    "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
    "spanId": "00f067aa0ba902b7",
    "parentSpanId": "00f067aa0ba902b6",
    "traceFlags": "01"
  }
}
```

### 5. No Message Size Limits or Pagination

**Problem:** 
- No specification for maximum message size
- No pagination for large result sets
- Could cause issues with MQTT broker limits

**Recommendation:**
Add pagination support for Query responses:

```json
{
  "pagination": {
    "page": 1,
    "pageSize": 100,
    "totalItems": 450,
    "totalPages": 5,
    "hasMore": true,
    "nextCursor": "eyJsYXN0SWQiOiAiMTIzIn0="
  }
}
```

### 6. Inconsistent Status Field Usage

**Problem:**
- `QueryResponse.LcaResults.json` has `"status": "success"`
- `QueryResponse.LcaResults.Error.json` has `"status": "error"` 
- Success responses nest data in `"data": {...}`
- Error responses put error directly in `"error": {...}`

**Current Structure (Inconsistent):**
```json
// Success
{
  "payload": {
    "status": "success",
    "data": { ... }
  }
}

// Error
{
  "payload": {
    "status": "error",
    "error": { ... }
  }
}
```

**Recommendation:**
Use consistent response envelope pattern:

```json
{
  "payload": {
    "status": "success" | "error" | "partial",
    "data": { ... },        // Present only on success/partial
    "error": { ... }        // Present only on error/partial
  }
}
```

### 7. Missing Idempotency Key in Events

**Problem:** 
- `idempotencyKey` only in Commands
- Events can be duplicated by MQTT (QoS 1) or network issues
- No way to detect duplicate event processing

**Why This Matters:**
From the documentation, MQTT QoS 1 can deliver duplicates. Without idempotency keys:
- Same event might be processed multiple times
- Model state could become corrupted
- No way to prevent duplicate side effects

**Recommendation:**
Add `idempotencyKey` to ALL message types:

```json
{
  "idempotencyKey": "event-model-change-2-systemA-20251121-102000",
  "messageType": "Model.Change",
  ...
}
```

**Pattern:** `{messageType}-{modelId}-{sequenceNumber}-{systemId}-{timestamp}`

### 8. No Message Priority Support

**Problem:** All messages treated equally - critical updates can be delayed behind bulk operations.

**Recommendation:**
Add priority field (0-9, where 9 is highest):

```json
{
  "priority": 5,  // 0=bulk, 5=normal, 9=critical
  "messageType": "Model.Change"
}
```

### 9. Missing Temporal Validation Fields

**Problem:** No expiration or validity period for messages.

**Why This Matters:**
- Old queries shouldn't be processed
- Commands might no longer be relevant
- Prevents stale data issues

**Recommendation:**
Add temporal control fields:

```json
{
  "temporal": {
    "expiresAt": "2025-11-21T11:00:00Z",
    "notBefore": "2025-11-21T10:00:00Z",
    "maxProcessingDelay": 300  // seconds
  }
}
```

---

## Minor Issues / Enhancements 🟢

### 10. Security Token Format Unclear

**Problem:** 
- All examples use placeholder SHA-256 hash: `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`
- No specification of what gets hashed
- No salt or algorithm details

**Recommendation:**
Document token format or use JWT:

```json
{
  "security": {
    "authScheme": "Bearer",
    "tokenHash": "sha256:a7b2c3d4...",
    "tokenAlgorithm": "HS256",
    "permissions": ["read:model", "write:lca"],
    "issuedAt": "2025-11-21T10:00:00Z",
    "expiresAt": "2025-11-21T12:00:00Z"
  }
}
```

### 11. Model Hash Encoding Always "hex"

**Problem:** Always specifies `"encoding": "hex"` but no other options shown.

**Recommendation:**
Either make this dynamic (support base64, etc.) or remove redundant field:

```json
{
  "hash": {
    "algorithm": "sha256",
    "value": "e3b0c44298...",
    "encoding": "hex"  // Keep if multiple encodings supported
  }
}
```

Or simplify:
```json
{
  "hash": "sha256:e3b0c44298..."  // Algorithm prefix pattern
}
```

### 12. Missing JSON Schema Definitions

**Problem:** No machine-readable schema for validation.

**Recommendation:**
Create JSON Schema files for each message type:
- Enables automatic validation
- Better IDE support
- Living documentation

### 13. No Batch Message Support

**Problem:** Can't send multiple operations in one message.

**Why This Matters:**
- Multiple small changes = many messages
- Network overhead
- Harder to maintain consistency

**Recommendation:**
Add batch operation support to `Model.Change`:

```json
{
  "messageType": "Model.Change.Batch",
  "payload": {
    "operations": [
      { "op": "update", "path": "...", ... },
      { "op": "add", "path": "...", ... },
      { "op": "delete", "path": "...", ... }
    ],
    "atomicity": "all-or-nothing"  // or "best-effort"
  }
}
```

### 14. IFC Path Format Undocumented

**Problem:** 
- Uses IFC paths like `"3v12fRbTX0hQxL6jLSEJxB"` and `"3Dz9w4Uer6ovnbK9EAPYN"`
- No documentation on:
  - Are these IFC GUIDs? (22-character base64 encoded)
  - IFC5 fragment references?
  - Custom identifiers?

**Recommendation:**
Document path format and add type hints:

```json
{
  "path": "3v12fRbTX0hQxL6jLSEJxB",
  "pathType": "IFC_GUID",
  "pathFormat": "base64-compressed"
}
```

### 15. Missing Response Timeout Specification

**Problem:** Queries don't specify how long to wait for responses.

**Recommendation:**
```json
{
  "messageType": "Model.Query",
  "responseTimeout": 30,  // seconds
  "correlationId": "query-model-001"
}
```

### 16. No Message Envelope Version

**Problem:** 
- `ifcDataBusVersion` is "0.2"
- But message format might evolve independently
- No way to version message structure separately

**Recommendation:**
Add separate message schema version:

```json
{
  "messageSchemaVersion": "1.0",
  "ifcDataBusVersion": "0.2",
  "messageType": "Model.Publish"
}
```

### 17. Missing Acknowledgment Pattern

**Problem:** No way for receiver to acknowledge message receipt.

**Recommendation:**
Define acknowledgment message type:

```json
{
  "messageType": "System.Acknowledgment",
  "acknowledges": {
    "correlationId": "LCA-2025-001",
    "sequenceNumber": 101,
    "status": "received" | "processing" | "completed" | "failed"
  }
}
```

---

## Proposed Improved Message Structure

### Core Message Envelope (v2.0)

```json
{
  "messageSchemaVersion": "2.0",
  "ifcDataBusVersion": "0.2",
  
  "messageType": "Model.Change",
  "messageId": "msg-20251121-103000-abc123",
  "correlationId": "change-001",
  "idempotencyKey": "change-2-bim-project-123-systemA-20251121-103000",
  
  "sequenceNumber": 2,
  "previousSequenceNumber": 1,
  
  "timestamp": "2025-11-21T10:20:00Z",
  "priority": 5,
  
  "sender": {
    "systemId": "SystemA",
    "systemType": "AuthoringTool",
    "userId": "author@example.com",
    "version": "1.2.3"
  },
  
  "routing": {
    "publishedOn": "ifc-data-bus/model/bim-project-123/change",
    "replyTo": "ifc-data-bus/reply/SystemA/change-001",
    "ttl": 3600
  },
  
  "security": {
    "authScheme": "Bearer",
    "tokenHash": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "permissions": ["write:model"],
    "issuedAt": "2025-11-21T10:00:00Z",
    "expiresAt": "2025-11-21T12:00:00Z"
  },
  
  "tracing": {
    "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
    "spanId": "00f067aa0ba902b7",
    "parentSpanId": "00f067aa0ba902b6"
  },
  
  "temporal": {
    "expiresAt": "2025-11-21T11:00:00Z",
    "maxProcessingDelay": 300
  },
  
  "modelRef": {
    "ifcSchema": "IFC5",
    "modelId": "bim-project-123",
    "revision": 2,
    "baseRevision": 1,
    "hash": {
      "algorithm": "sha256",
      "value": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
      "encoding": "hex"
    }
  },
  
  "payload": {
    "changeDescription": "Fenster in Fassade Süd angepasst",
    "operations": [
      {
        "op": "update",
        "path": "3v12fRbTX0hQxL6jLSEJxB",
        "pathType": "IFC_GUID",
        "entityType": "IfcWindow",
        "changedProperties": {
          "OverallHeight": { "old": 1.20, "new": 1.35 },
          "OverallWidth": { "old": 0.90, "new": 1.00 }
        }
      }
    ]
  }
}
```

### Standardized Error Response

```json
{
  "messageType": "Model.QueryResponse",
  "correlationId": "query-model-001",
  
  "payload": {
    "status": "error",
    "error": {
      "code": "MODEL_NOT_FOUND",
      "category": "NotFound",
      "severity": "error",
      "message": "Model 'bim-project-123' revision 2 not found",
      "userMessage": "The requested building model version could not be found. Please verify the model ID and revision number.",
      "details": {
        "modelId": "bim-project-123",
        "revision": 2,
        "availableRevisions": [1]
      },
      "retryable": false,
      "retryAfter": null,
      "helpUrl": "https://docs.ifc-data-bus.org/errors/MODEL_NOT_FOUND",
      "errorId": "err-20251121-110005-xyz789"
    }
  }
}
```

---

## Standardized Error Code Registry

### Error Categories

| Category | HTTP Equivalent | Description |
|----------|----------------|-------------|
| `BadRequest` | 400 | Invalid message format or parameters |
| `Unauthorized` | 401 | Authentication required or failed |
| `Forbidden` | 403 | Insufficient permissions |
| `NotFound` | 404 | Requested resource doesn't exist |
| `Conflict` | 409 | State conflict (e.g., wrong sequence number) |
| `Unprocessable` | 422 | Valid format but business logic prevents processing |
| `TooManyRequests` | 429 | Rate limit exceeded |
| `InternalError` | 500 | System error |
| `ServiceUnavailable` | 503 | Temporary unavailability |
| `Timeout` | 504 | Operation timed out |

### Common Error Codes

```
MODEL_NOT_FOUND
MODEL_REVISION_MISMATCH
MODEL_HASH_MISMATCH
SEQUENCE_GAP_DETECTED
SEQUENCE_NUMBER_CONFLICT
INVALID_IFC_PATH
INVALID_OPERATION
PERMISSION_DENIED
TOKEN_EXPIRED
TOKEN_INVALID
LCA_CALCULATION_FAILED
LCA_RESULTS_NOT_FOUND
INVALID_MESSAGE_FORMAT
SCHEMA_VALIDATION_FAILED
MESSAGE_EXPIRED
IDEMPOTENCY_KEY_CONFLICT
```

---

## Implementation Priority

### Phase 1 (Critical - Do First) 🔴
1. Add idempotency keys to all message types
2. Standardize error handling structure
3. Add MQTT routing information
4. Document IFC path format

### Phase 2 (Important - Do Next) 🟡
5. Add distributed tracing support
6. Implement sequence numbers for Commands/Queries
7. Add pagination support
8. Fix status field inconsistency
9. Add message priority

### Phase 3 (Nice to Have) 🟢
10. Create JSON Schema definitions
11. Add batch operation support
12. Implement acknowledgment pattern
13. Add temporal validation fields
14. Separate message schema versioning

---

## Migration Strategy

### Backward Compatibility

To avoid breaking existing implementations:

1. **Keep existing fields** - Don't remove current fields
2. **Add new optional fields** - New fields should be optional initially
3. **Version negotiation** - Use `messageSchemaVersion` to support multiple versions
4. **Deprecation period** - Mark old patterns as deprecated with 6-month sunset

### Migration Example

```json
{
  "messageSchemaVersion": "2.0",
  "ifcDataBusVersion": "0.2",
  
  // New required fields
  "idempotencyKey": "...",
  "routing": { ... },
  
  // New optional fields (can be omitted by v1.0 clients)
  "tracing": { ... },
  "priority": 5,
  
  // Existing fields (unchanged)
  "messageType": "Model.Change",
  "correlationId": "change-001",
  "timestamp": "2025-11-21T10:20:00Z",
  ...
}
```

---

## Testing Recommendations

### Message Validation Tests

1. **Schema validation** - All messages validate against JSON Schema
2. **Sequence number validation** - Gaps/conflicts detected
3. **Idempotency tests** - Duplicate messages handled correctly
4. **Security tests** - Invalid tokens rejected
5. **Error handling tests** - All error codes have test coverage

### Integration Tests

1. **Cross-system message flow** - Test full Command→Event→Query flow
2. **Out-of-order delivery** - Test sequence number handling
3. **Network failure scenarios** - Test retry/timeout behavior
4. **Large message handling** - Test pagination
5. **Concurrent modification** - Test conflict detection

---

## Documentation Recommendations

### Required Documentation

1. **Message Type Catalog** - Document all message types with examples
2. **Error Code Registry** - Complete list with descriptions and solutions
3. **IFC Path Format Specification** - Document GUID/path format
4. **Security Implementation Guide** - Token generation and validation
5. **Sequence Number Guide** - When and how to use sequence numbers
6. **MQTT Topic Schema** - Topic naming conventions

### Developer Experience

1. **Message Builder Library** - Provide SDKs for common languages
2. **Message Validator** - CLI tool to validate messages against schema
3. **Mock MQTT Broker** - For local testing
4. **Example Messages** - Comprehensive example collection
5. **Postman/Insomnia Collection** - For API exploration

---

## Conclusion

The current message format is well-designed for the hackathon scope, but implementing these improvements will make the IFC Data Bus production-ready with better:

- **Reliability** - Idempotency, error handling, sequence numbers
- **Observability** - Tracing, monitoring, debugging capabilities
- **Developer Experience** - Clear documentation, validation, helpful errors
- **Scalability** - Pagination, priority, batch operations
- **Maintainability** - Versioning, migration strategy, JSON Schema

### Recommended Next Steps

1. Review this document with the hackathon team
2. Prioritize improvements based on demo requirements
3. Implement Phase 1 (Critical) changes
4. Create JSON Schema definitions
5. Update example messages
6. Document error codes

---

**Questions or Feedback?** Please open an issue or discussion on the GitHub repository.


# Improved Message Examples (v2.0)

This folder contains example messages demonstrating the proposed improvements to the IFC Data Bus message format.

## Key Improvements Demonstrated

### 1. Enhanced Message Envelope
All examples now include:
- `messageSchemaVersion`: Separate versioning for message format
- `messageId`: Unique identifier for each message
- `idempotencyKey`: Prevents duplicate processing
- `priority`: Message prioritization (0-9)
- `routing`: MQTT topic information and TTL

### 2. Distributed Tracing
OpenTelemetry-compatible tracing fields:
- `traceId`: Distributed trace identifier
- `spanId`: Current span identifier  
- `parentSpanId`: Parent span for trace hierarchy

### 3. Temporal Control
New `temporal` section:
- `expiresAt`: Message expiration timestamp
- `responseTimeout`: How long to wait for responses
- `maxProcessingDelay`: Maximum acceptable processing delay

### 4. Improved Error Handling
Enhanced error structure with:
- `category`: Error category (NotFound, BadRequest, etc.)
- `severity`: Error severity level
- `userMessage`: User-friendly error description
- `retryable`: Whether the operation can be retried
- `retryAfter`: When to retry (seconds)
- `helpUrl`: Link to documentation
- `errorId`: Unique error identifier
- `suggestedActions`: Actionable steps for resolution

### 5. Richer Response Data
Success responses now include:
- `pagination`: Pagination support for large result sets
- `breakdown`: Detailed analysis breakdowns
- `benchmarks`: Performance benchmarks and ratings
- `warnings`: Non-fatal warnings with context
- `qualityIndicators`: Data quality metrics

### 6. Acknowledgment Pattern
New `System.Acknowledgment` message type:
- Confirms message receipt
- Provides processing status updates
- Shows progress information
- Enables async operation tracking

## File Overview

| File | Description |
|------|-------------|
| `Model.Change.v2.json` | Enhanced model change event with full envelope |
| `de.din.DIN_EN15978_2012_10.Command.CalculateBuildingLca.v2.json` | LCA command with sequence numbers and routing |
| `de.din.DIN_EN15978_2012_10.QueryResponse.LcaResults.Error.v2.json` | Standardized error response with rich error info |
| `de.din.DIN_EN15978_2012_10.QueryResponse.LcaResults.Success.v2.json` | Success response with pagination and detailed LCA data |
| `System.Acknowledgment.v2.json` | New acknowledgment message showing progress |

## Comparison with v1.0

### v1.0 Message Size
```json
{
  "ifcDataBusVersion": "0.2",
  "messageType": "Model.Change",
  "correlationId": "change-001",
  "timestamp": "2025-11-21T10:20:00Z",
  ...
}
```

### v2.0 Message Size
```json
{
  "messageSchemaVersion": "2.0",
  "ifcDataBusVersion": "0.2",
  "messageType": "Model.Change",
  "messageId": "msg-20251121-102000-abc123",
  "correlationId": "change-001",
  "idempotencyKey": "change-2-bim-project-123-systemA-20251121-102000",
  "timestamp": "2025-11-21T10:20:00Z",
  "priority": 5,
  "routing": { ... },
  "tracing": { ... },
  "temporal": { ... },
  ...
}
```

**Size Increase:** ~40% more metadata, but significantly improved reliability and observability.

## Migration Guide

### Making v1.0 Consumers v2.0 Compatible

1. **Ignore Unknown Fields**: v1.0 consumers should ignore new v2.0 fields
2. **Check Schema Version**: Read `messageSchemaVersion` to determine format
3. **Gradual Adoption**: New fields are optional initially

### Example Consumer Code (Python)

```python
def process_message(message):
    # Detect message version
    schema_version = message.get("messageSchemaVersion", "1.0")
    
    if schema_version == "2.0":
        # Handle v2.0 features
        if "idempotencyKey" in message:
            if is_duplicate(message["idempotencyKey"]):
                return  # Skip duplicate
        
        # Extract tracing context
        if "tracing" in message:
            setup_tracing(message["tracing"])
    
    # Process common fields (compatible with both versions)
    message_type = message["messageType"]
    correlation_id = message["correlationId"]
    payload = message["payload"]
    
    # Your business logic here
    ...
```

## Field Reference

### Required Fields (v2.0)
- `messageSchemaVersion`
- `ifcDataBusVersion`
- `messageType`
- `messageId`
- `correlationId`
- `idempotencyKey`
- `timestamp`
- `sender`
- `payload`

### Optional Fields (v2.0)
- `sequenceNumber` (required for events that create state changes)
- `previousSequenceNumber` (required if sequenceNumber present)
- `priority` (defaults to 5)
- `routing` (recommended for MQTT deployments)
- `security` (required for authenticated systems)
- `tracing` (recommended for production systems)
- `temporal` (recommended for time-sensitive operations)
- `modelRef` (required for model-related messages)

## Best Practices

### 1. Idempotency Key Generation
```
Pattern: {messageType}-{entityId}-{sequence}-{systemId}-{timestamp}
Example: change-2-bim-project-123-systemA-20251121-102000
```

### 2. Sequence Number Management
- Use separate sequence counters per model
- Include `previousSequenceNumber` for validation
- Start from 1 for new models

### 3. Priority Levels
- 0-2: Bulk operations, background tasks
- 3-5: Normal operations (default: 5)
- 6-7: High priority, user-initiated
- 8-9: Critical, real-time updates

### 4. TTL (Time-to-Live)
- Commands: 600-3600 seconds
- Events: 3600-86400 seconds
- Queries: 30-300 seconds
- Acknowledgments: 60 seconds

### 5. Tracing Integration
```python
from opentelemetry import trace

def publish_message(message):
    tracer = trace.get_tracer(__name__)
    with tracer.start_as_current_span("publish_message") as span:
        message["tracing"] = {
            "traceId": format(span.context.trace_id, '032x'),
            "spanId": format(span.context.span_id, '016x'),
            "parentSpanId": format(span.parent.span_id, '016x') if span.parent else None
        }
        mqtt_client.publish(topic, json.dumps(message))
```

## Testing

### Validation Tools
```bash
# Validate message against JSON schema
python validate_message.py Model.Change.v2.json

# Check idempotency key uniqueness
python check_idempotency.py messages/*.json

# Verify sequence number chain
python verify_sequence.py messages/*.json
```

### Mock Messages
All examples use realistic but non-functional values:
- Hashes: Random but properly formatted SHA-256
- IFC GUIDs: Valid 22-character base64 format
- Timestamps: Recent dates
- Trace IDs: Valid OpenTelemetry format

## Questions or Feedback?

See `MESSAGE_FORMAT_REVIEW.md` for detailed rationale and discussion of each improvement.


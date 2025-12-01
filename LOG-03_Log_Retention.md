# Cybersecurity Audit - Control LOG-03

## Control Information

- **Control ID**: LOG-03
- **Control Name**: Log Retention
- **Audit Date**: 2025-11-27
- **Client Question**: "How long do you retain logs?"

---

## Executive Summary

⚠️ **PARTIALLY COMPLIANT**: The platform generates comprehensive logs across multiple layers (application, edge functions, security events), and uses Grafana for log aggregation and monitoring. However, the explicit log retention policy is not documented in the codebase. The retention period is managed through Supabase's logging infrastructure and Grafana configuration, which are external to the application code.

1. **Comprehensive Logging** - Logs are generated at application level, edge functions, and security events
2. **Grafana Integration** - Logs are aggregated and monitored through Grafana
3. **Multiple Log Sources** - Application logs, edge function logs, security logs, and database audit logs
4. **Retention Policy** - Managed by Supabase infrastructure and Grafana configuration (not in codebase)
5. **Log Classification** - Different types of logs (security, application, debug) are generated but classification for retention purposes is not explicitly documented

---

## 1. Logging Infrastructure

The platform uses **Supabase** as its backend infrastructure, which provides built-in logging capabilities that can be exported to external monitoring tools like Grafana.

### 1.1. Supabase Logging

**Provider**: Supabase (managed PostgreSQL and Edge Functions)

- **Database Logs**: PostgreSQL logs including query logs, connection logs, and error logs
- **Edge Function Logs**: Execution logs from Deno-based edge functions
- **API Logs**: Request/response logs from Supabase API endpoints
- **Auth Logs**: Authentication and authorization event logs
- **Storage Logs**: File upload/download logs from Supabase Storage

**Evidence**:
According to Supabase documentation:
- Supabase provides comprehensive logging for all services (database, API, auth, storage, edge functions)
- Logs can be exported to external monitoring tools including Grafana
- Log retention is configurable through Supabase dashboard or API

### 1.2. Grafana Integration

The platform uses **Grafana** for log aggregation, visualization, and monitoring:

- **Log Aggregation**: Centralized collection of logs from multiple sources
- **Visualization**: Dashboards for log analysis and monitoring
- **Retention Management**: Log retention policies configured in Grafana
- **Query Capabilities**: Advanced log querying and filtering

**Evidence**:
The client has confirmed that Grafana is used for log management. Grafana typically supports configurable retention policies based on:
- Time-based retention (e.g., 30 days, 90 days, 1 year)
- Size-based retention (e.g., maximum storage size)
- Log type classification (different retention for different log types)

---

## 2. Application-Level Logging

The application generates logs at multiple levels for debugging, monitoring, and security purposes.

### 2.1. Edge Function Logging

Edge functions use `console.log()`, `console.error()`, and `console.warn()` for logging, which are captured by Supabase's logging infrastructure.

**Evidence**:
```typescript
// supabase/functions/crypto-service/index.ts
Deno.serve(async (req) => {
  try {
    // ... authentication and processing ...
    
    if (!masterKeyHex) {
      console.error("MASTER_ENCRYPTION_KEY is not set");
      throw new Error("Server configuration error");
    }
    
    // ... encryption/decryption logic ...
    
  } catch (error) {
    console.error("Crypto service error:", error);
    return new Response(JSON.stringify({ error: error.message }), {
      status: 400,
      headers: { ...corsHeaders, "Content-Type": "application/json" },
    });
  }
});
```

**Edge Functions with Logging**:
- `crypto-service` - Logs encryption/decryption operations and errors
- `stripe-webhook` - Logs webhook events, payment processing, and errors
- `generate-user-keys` - Logs key generation operations
- `generate-company-keys` - Logs company key generation
- `send-notification-email` - Logs email sending operations
- `send-rfx-review-email` - Logs review email operations
- `cleanup-temp-files` - Logs file cleanup operations

### 2.2. Client-Side Logging

The application generates logs in the browser console for debugging and monitoring:

**Evidence**:
```typescript
// src/services/antiScrapingService.ts
async logSuspiciousActivity(activity: {
  type: string;
  details: string;
  userAgent?: string;
  ip?: string;
  timestamp?: number;
}): Promise<void> {
  try {
    // En un entorno real, esto se guardaría en la base de datos
    console.warn('Suspicious activity detected:', activity);
    
    console.log('Security log entry:', {
      type: activity.type,
      details: activity.details,
      user_agent: activity.userAgent || navigator.userAgent,
      ip_address: activity.ip || this.getClientIP(),
      timestamp: activity.timestamp || Date.now(),
    });
  } catch (error) {
    console.error('Error logging suspicious activity:', error);
  }
}
```

**Note**: Client-side console logs are ephemeral and not persisted unless captured by a logging service or browser developer tools. These logs are primarily for development and debugging purposes.

### 2.3. RFX Key Distribution Logging

The RFX key distribution system generates structured logs with identifiable prefixes:

**Evidence**:
```markdown
// docs/RFX_KEY_DISTRIBUTION_IMPLEMENTATION.md
## Logs y Monitoreo

El sistema genera logs detallados con prefijos identificables:

### Logs de Envío (RFXSendingPage)
- `🔐 [RFX Sending]` - Inicio de distribución a developers
- `✅ [RFX Sending]` - Distribución exitosa a developers
- `⚠️ [RFX Sending]` - Advertencias durante distribución
- `❌ [RFX Sending]` - Errores durante distribución

### Logs de Aprobación (RFXManagement)
- `🔐 [RFX Management]` - Inicio de distribución a suppliers
- `✅ [RFX Management]` - Distribución exitosa a suppliers
- `⚠️ [RFX Management]` - Advertencias durante distribución
- `❌ [RFX Management]` - Errores durante distribución

### Logs de Distribución (rfxKeyDistribution)
- `🔑 [RFX Key Distribution]` - Inicio de distribución (general)
- `👥 [RFX Key Distribution]` - Conteo de usuarios/developers
- `🔐 [RFX Key Distribution]` - Operaciones de cifrado
- `✅ [RFX Key Distribution]` - Operaciones exitosas por usuario
- `⚠️ [RFX Key Distribution]` - Advertencias (usuarios sin claves)
- `❌ [RFX Key Distribution]` - Errores fatales
```

---

## 3. Log Types and Classification

The platform generates various types of logs, but explicit classification for retention purposes is not documented in the codebase.

### 3.1. Security Logs

**Types**:
- Authentication events (login, logout, session management)
- Authorization failures
- Suspicious activity detection
- Security policy violations
- Encryption/decryption operations

**Evidence**:
```typescript
// src/services/antiScrapingService.ts
async logSuspiciousActivity(activity: {
  type: string;
  details: string;
  userAgent?: string;
  ip?: string;
  timestamp?: number;
}): Promise<void> {
  console.warn('Suspicious activity detected:', activity);
  console.log('Security log entry:', {
    type: activity.type,
    details: activity.details,
    user_agent: activity.userAgent || navigator.userAgent,
    ip_address: activity.ip || this.getClientIP(),
    timestamp: activity.timestamp || Date.now(),
  });
}
```

### 3.2. Application Logs

**Types**:
- Application errors and exceptions
- Business logic events
- User actions
- Data operations
- API requests and responses

**Evidence**:
```typescript
// supabase/functions/stripe-webhook/index.ts
async function handler(req: Request): Promise<Response> {
  // Log all incoming requests for debugging
  console.log("Request received", {
    method: req.method,
    url: req.url,
    headers: Object.fromEntries(req.headers.entries()),
    hasStripeSignature: !!req.headers.get("stripe-signature"),
  });
  
  // ... processing ...
  
  console.log("Webhook event received", { type: event.type, id: event.id });
}
```

### 3.3. Audit Logs

**Types**:
- Database operations (via Supabase RLS and triggers)
- Data access events
- Administrative actions
- Configuration changes

**Note**: Supabase provides audit logging capabilities through PostgreSQL's built-in logging and Supabase-specific audit features. These logs are managed by Supabase infrastructure.

### 3.4. Debug Logs

**Types**:
- Development debugging information
- Detailed operation traces
- Performance metrics
- Diagnostic information

**Evidence**:
```typescript
// supabase/functions/generate-user-keys/index.ts
console.log(`🔑 Generating keys for user ${target_user_id}...`);
console.log(`✅ Keys generated and stored successfully for user ${target_user_id}`);
console.log(`⚠️ User ${target_user_id} acquired keys during generation. Returning existing keys.`);
```

---

## 4. Log Storage and Retention

### 4.1. Supabase Log Storage

Supabase stores logs in its managed infrastructure:

- **Database Logs**: Stored in Supabase's logging infrastructure
- **Edge Function Logs**: Captured and stored by Supabase Edge Runtime
- **API Logs**: Stored in Supabase API gateway logs
- **Storage Logs**: Stored in Supabase Storage service logs

**Retention**: Managed by Supabase's infrastructure. Retention periods can be configured through:
- Supabase Dashboard settings
- Supabase API for log management
- Integration with external log aggregation tools (e.g., Grafana)

### 4.2. Grafana Log Retention

Grafana manages log retention through:

- **Data Source Configuration**: Retention policies configured per data source
- **Storage Backend**: Depends on the underlying storage (e.g., Loki, Elasticsearch, CloudWatch)
- **Retention Policies**: Time-based or size-based retention rules
- **Log Classification**: Different retention periods for different log types (if configured)

**Evidence**:
The client has confirmed Grafana is used for log management. Typical Grafana retention configurations include:
- Short-term logs (debug, info): 7-30 days
- Medium-term logs (warnings, errors): 30-90 days
- Long-term logs (security, audit): 90 days - 1 year or longer

### 4.3. Log Export and Archival

Logs can be exported from Supabase to Grafana for:
- Long-term retention
- Compliance requirements
- Advanced analysis and correlation
- Centralized monitoring

**Note**: The specific retention periods and archival policies are configured in Grafana and Supabase infrastructure, not in the application code.

---

## 5. Log Retention Policy Documentation

### 5.1. Current State

**Finding**: The explicit log retention policy is not documented in the application codebase.

**Evidence**:
- No retention policy documentation found in codebase
- No retention configuration in application code
- No retention-related environment variables or configuration files
- Retention is managed externally (Supabase/Grafana)

### 5.2. Expected Retention Policy Components

A comprehensive log retention policy should include:

1. **Retention Periods by Log Type**:
   - Security logs: Minimum retention period (e.g., 1 year)
   - Audit logs: Compliance-driven retention (e.g., 7 years for financial data)
   - Application logs: Operational retention (e.g., 30-90 days)
   - Debug logs: Short-term retention (e.g., 7-30 days)

2. **Retention Mechanisms**:
   - Active storage period
   - Archival period
   - Deletion schedule

3. **Compliance Requirements**:
   - Regulatory requirements (GDPR, industry-specific)
   - Legal hold requirements
   - Audit trail requirements

4. **Storage and Access**:
   - Primary storage location
   - Archival storage location
   - Access controls for archived logs

---

## 6. Logging Best Practices

### 6.1. Structured Logging

The platform uses structured logging in some areas:

**Evidence**:
```typescript
// supabase/functions/stripe-webhook/index.ts
console.log("Webhook event received", { 
  type: event.type, 
  id: event.id 
});

console.log("checkout.session.completed received", { 
  sessionId: session.id, 
  mode: session.mode, 
  paymentStatus: session.payment_status,
  hasInvoice: !!session.invoice,
  invoiceId: typeof session.invoice === "string" ? session.invoice : session.invoice?.id,
  customerId: session.customer,
  subscriptionId
});
```

### 6.2. Log Levels

The platform uses appropriate log levels:
- `console.log()` - Informational messages
- `console.warn()` - Warnings and suspicious activity
- `console.error()` - Errors and exceptions

### 6.3. Sensitive Data Protection

**Finding**: Logs may contain sensitive information that should be protected or redacted.

**Evidence**:
```typescript
// supabase/functions/stripe-webhook/index.ts
// Log that secret is configured (without exposing the value)
console.log("STRIPE_WEBHOOK_SECRET configured", { 
  hasSecret: !!STRIPE_WEBHOOK_SECRET,
  secretPrefix: STRIPE_WEBHOOK_SECRET.substring(0, 6) + "...",
  secretLength: STRIPE_WEBHOOK_SECRET.length
});
```

**Good Practice**: The code demonstrates awareness of not logging sensitive values directly, showing only prefixes and lengths.

---

## 7. Conclusions

### 7.1. Strengths

✅ **Comprehensive Logging**: The platform generates logs across multiple layers (application, edge functions, security events)

✅ **Grafana Integration**: Centralized log aggregation and monitoring through Grafana provides professional log management capabilities

✅ **Structured Logging**: Some areas use structured logging with identifiable prefixes and consistent formats

✅ **Multiple Log Sources**: Logs are generated from various sources (database, API, edge functions, client-side) providing comprehensive visibility

✅ **Security Logging**: Security events and suspicious activities are logged for monitoring and investigation

✅ **Sensitive Data Protection**: Code demonstrates awareness of not logging sensitive values directly

### 7.2. Recommendations

1. **Document Log Retention Policy**: Create formal documentation specifying retention periods for different log types (security, audit, application, debug) and ensure it aligns with compliance requirements

2. **Implement Log Classification**: Explicitly classify logs by type (security, audit, application, debug) to enable different retention policies in Grafana

3. **Define Retention Periods**: Document specific retention periods:
   - Security logs: Minimum 1 year (or as required by compliance)
   - Audit logs: 7 years (or as required by regulations)
   - Application logs: 30-90 days
   - Debug logs: 7-30 days

4. **Centralize Security Logs**: Implement persistent storage for security logs (currently using console.warn/log) in a dedicated security logs table or service for long-term retention and analysis

5. **Configure Grafana Retention**: Ensure Grafana retention policies are configured and documented, with different retention periods for different log classifications

6. **Implement Log Archival**: For long-term retention requirements, implement log archival to cost-effective storage (e.g., S3, Glacier) with appropriate access controls

7. **Regular Retention Reviews**: Establish a process for regular review of log retention policies to ensure they remain compliant with changing regulations and business requirements

8. **Log Retention Automation**: Automate log retention and archival processes to ensure consistent application of retention policies

---

## 8. Control Compliance

| Criterion | Status | Evidence |
|-----------|--------|----------|
| Log retention policy exists | ⚠️ PARTIAL | Logs are generated and aggregated in Grafana, but explicit retention policy is not documented in codebase |
| Retention policy is documented | ❌ NON-COMPLIANT | No retention policy documentation found in application codebase |
| Retention periods are reasonable | ⚠️ PARTIAL | Retention is managed by Supabase/Grafana infrastructure, but specific periods are not documented |
| Logs are classified for retention | ⚠️ PARTIAL | Different log types are generated, but explicit classification for retention purposes is not documented |
| Retention is automated | ⚠️ PARTIAL | Retention is managed by infrastructure (Supabase/Grafana), but automation is not visible in codebase |

**FINAL VERDICT**: ⚠️ **PARTIALLY COMPLIANT** with control LOG-03. The platform generates comprehensive logs across multiple layers and uses Grafana for log aggregation and monitoring, which provides the infrastructure for reasonable log retention. However, the explicit log retention policy is not documented in the codebase. Retention periods are managed through Supabase and Grafana configuration, which are external to the application code. To achieve full compliance, the platform should document explicit retention policies, classify logs for different retention periods, and ensure retention periods align with compliance and business requirements.

---

## Appendices

### A. Log Sources Summary

| Source | Log Type | Examples | Retention Location |
|--------|----------|-----------|-------------------|
| Edge Functions | Application, Security | `crypto-service`, `stripe-webhook`, `generate-user-keys` | Supabase Edge Runtime → Grafana |
| Database | Audit, Application | Query logs, RLS policy logs | Supabase PostgreSQL → Grafana |
| API | Application, Security | Request/response logs, auth events | Supabase API Gateway → Grafana |
| Storage | Application | Upload/download logs | Supabase Storage → Grafana |
| Client-side | Debug, Security | Console logs, suspicious activity | Browser console (ephemeral) |
| Security Service | Security | Suspicious activity detection | Console (should be persisted) |

### B. Edge Functions with Logging

The following edge functions generate logs:

1. **crypto-service** (`supabase/functions/crypto-service/index.ts`)
   - Logs: Encryption/decryption operations, errors
   - Log Level: Error, Info

2. **stripe-webhook** (`supabase/functions/stripe-webhook/index.ts`)
   - Logs: Webhook events, payment processing, errors
   - Log Level: Log, Error

3. **generate-user-keys** (`supabase/functions/generate-user-keys/index.ts`)
   - Logs: Key generation operations, warnings, errors
   - Log Level: Log, Warn, Error

4. **generate-company-keys** (`supabase/functions/generate-company-keys/index.ts`)
   - Logs: Company key generation, warnings, errors
   - Log Level: Log, Warn, Error

5. **send-notification-email** (`supabase/functions/send-notification-email/index.ts`)
   - Logs: Email sending operations, errors
   - Log Level: Error

6. **send-rfx-review-email** (`supabase/functions/send-rfx-review-email/index.ts`)
   - Logs: Review email operations, warnings, errors
   - Log Level: Warn, Error

7. **cleanup-temp-files** (`supabase/functions/cleanup-temp-files/index.ts`)
   - Logs: File cleanup operations, errors
   - Log Level: Log, Error

### C. Recommended Retention Periods

Based on industry best practices and compliance requirements:

| Log Type | Recommended Retention | Rationale |
|----------|----------------------|-----------|
| Security Logs | 1-3 years | Required for security incident investigation and compliance |
| Audit Logs | 7 years | Common requirement for financial and regulatory compliance |
| Application Error Logs | 90 days | Sufficient for troubleshooting and monitoring |
| Application Info Logs | 30 days | Operational monitoring and debugging |
| Debug Logs | 7-30 days | Short-term debugging and development |
| Database Audit Logs | 1-7 years | Compliance and data access auditing |
| API Access Logs | 90 days | Security monitoring and troubleshooting |

### D. Grafana Configuration Recommendations

For Grafana log retention configuration:

1. **Data Source Setup**:
   - Configure Supabase logs as data source in Grafana
   - Set up appropriate retention policies per data source

2. **Log Classification**:
   - Create labels/tags for log classification (security, audit, application, debug)
   - Configure different retention policies based on classification

3. **Storage Backend**:
   - Use appropriate storage backend (Loki, Elasticsearch, CloudWatch) based on volume and retention requirements
   - Configure storage limits and archival policies

4. **Retention Rules**:
   - Define time-based retention rules (e.g., delete after X days)
   - Define size-based retention rules (e.g., keep last X GB)
   - Configure archival rules for long-term storage

5. **Access Controls**:
   - Implement appropriate access controls for log viewing and management
   - Ensure compliance with data protection regulations

---

**End of Audit Report - Control LOG-03**


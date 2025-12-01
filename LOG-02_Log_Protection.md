# Cybersecurity Audit - Control LOG-02

## Control Information

- **Control ID**: LOG-02
- **Control Name**: Log Protection
- **Audit Date**: 2025-11-27
- **Client Question**: "Do logs contain sensitive data? How are they protected?"

---

## Executive Summary

⚠️ **PARTIAL COMPLIANCE**: The platform implements logging infrastructure with centralized storage in Grafana Loki, but local log files contain sensitive user data without encryption or access controls. Logs transmitted to Grafana Loki are protected via HTTPS, but local log files on the filesystem lack adequate protection mechanisms.

1. **Sensitive data in logs** - User messages, conversation content, and agent responses are logged in plain text in local files
2. **Encryption in transit** - Logs sent to Grafana Loki are protected via HTTPS with API token authentication
3. **No local file encryption** - Local log files (`ws_messages.log`, `ws_agent_messages.log`) are stored unencrypted on the filesystem
4. **No access controls** - Local log files are written with standard file permissions without additional access restrictions
5. **No data sanitization** - No filtering or sanitization of sensitive data before logging
6. **No retention policy** - No visible log retention or rotation policy for local log files

---

## 1. Logging Infrastructure Overview

The platform implements a multi-layered logging system that captures application events, user interactions, and system information through both local file logging and centralized cloud-based logging via Grafana Loki.

### 1.1. Logging Components

The platform uses three main logging mechanisms:

1. **Grafana Loki** - Centralized cloud-based log aggregation service
2. **Local log files** - File-based logging for WebSocket messages and agent interactions
3. **Print interceptor** - Automatic capture of all stdout/stderr output

**Evidence**:
```python
// agent/tools/grafana_loki_client.py
class GrafanaLokiClient:
    """
    Cliente para enviar logs directamente a Grafana Loki via HTTP API
    """
    
    def __init__(self):
        # Configuración de Grafana Cloud Loki
        self.loki_url = os.getenv("LOKI_URL", "https://logs-prod-012.grafana.net")
        self.loki_user = os.getenv("LOKI_USER", "1293489")
        self.loki_password = os.getenv("LOKI_PASSWORD")  # Token de API
        
        # Configurar autenticación básica
        self.auth = (self.loki_user, self.loki_password)
```

### 1.2. Local Log Files

The platform maintains two local log files that capture application activity:

- **`ws_messages.log`** - WebSocket message logging
- **`ws_agent_messages.log`** - Agent interaction logging

**Evidence**:
```python
// api/ws_handler.py
def log_agent_message(self, msg):
    with open("ws_agent_messages.log", "a", encoding="utf-8") as f:
        f.write(f"[{datetime.now().isoformat()}] {msg}\n")

// api/rfx_specs_handler.py
def _log_ws_message(msg: str):
    with open("ws_messages.log", "a", encoding="utf-8") as f:
        f.write(f"[{datetime.now().isoformat()}] {msg}\n")
```

These log files are:
- Written in append mode (`"a"`) without explicit file permissions
- Stored in the application root directory
- Excluded from version control (`.gitignore`, `.dockerignore`)
- Not encrypted at rest

---

## 2. Sensitive Data in Logs

### 2.1. User Messages and Conversation Content

User input messages are logged in plain text without sanitization or filtering. This includes all user-provided content sent through WebSocket connections.

**Evidence**:
```python
// api/ws_handler.py
def _enqueue(self, message):
    msg_str = str(message)
    try:
        log_str = json.dumps(message, ensure_ascii=False, default=str)
    except Exception:
        log_str = str(message)
    self.log_agent_message(log_str)  # Logs complete message content
```

The logged data includes:
- **User input text** - Complete user messages in plain text
- **Agent responses** - Full agent-generated responses
- **Tool inputs and outputs** - Complete tool call data including parameters and results
- **Conversation metadata** - Conversation IDs, user IDs, and session information
- **System information** - Memory usage, CPU metrics, connection counts

**Evidence**:
```python
// api/webserver.py
def log_system_usage():
    """
    Envía un log a Loki con información sobre el uso de memoria y CPU
    """
    message = f"System usage: RSS={system_data['rss_mb']}MB, VMS={system_data['vms_mb']}MB, ProcessMem={system_data['memory_percent']}%, ProcessCPU={system_data['process_cpu_percent']}%, SystemMem={system_data['system_memory_percent']}%, SystemCPU={system_data['system_cpu_percent']}%, ActiveConnections={system_data['active_connections']}, ChildrenMemory={system_data['children_memory_mb']}MB, ChildrenCount={system_data['children_count']}, TotalMemory={system_data['total_memory_mb']}MB"
    
    loki_client.send_log(
        message=message,
        level="info",
        labels=labels
    )
```

### 2.2. No Data Sanitization

The platform does not implement any filtering, sanitization, or masking of sensitive data before logging. All data is logged as-is without any redaction mechanisms.

**Evidence**:
```python
// agent/tools/print_interceptor.py
def write(self, text):
    """Intercepta writes a stdout/stderr"""
    # Escribir al stdout original
    self.original_stdout.write(text)
    self.original_stdout.flush()
    
    # Enviar a Loki si no es un mensaje de Loki y no contiene intermediate_step
    if not text.startswith("[LOKI]"):
        # Filtrar mensajes que contengan intermediate_step
        if "intermediate_step" not in text:
            self._queue_log(text.strip(), "print")  # No sanitization applied
```

The only filtering applied is:
- Exclusion of messages starting with `[LOKI]` (to prevent log loops)
- Exclusion of messages containing `"intermediate_step"` (for performance reasons)

No filtering is applied for:
- Passwords or authentication tokens
- API keys or secrets
- Personal identifiable information (PII)
- Financial data
- Other sensitive user data

### 2.3. Credentials and Secrets

The platform correctly avoids logging API keys and passwords. These are stored in environment variables and are not included in log messages.

**Evidence**:
```python
// agent/tools/grafana_loki_client.py
self.loki_password = os.getenv("LOKI_PASSWORD")  # Token de API - not logged
```

However, the absence of explicit sanitization means that if sensitive data were accidentally included in user messages or error messages, it would be logged without protection.

---

## 3. Log Protection Mechanisms

### 3.1. Grafana Loki Protection

Logs transmitted to Grafana Loki are protected through multiple mechanisms:

**Encryption in Transit**:
- All communication with Grafana Loki uses HTTPS
- API token authentication via Basic Auth
- Organization-scoped access via `X-Scope-OrgID` header

**Evidence**:
```python
// agent/tools/grafana_loki_client.py
push_url = f"{self.loki_url}/loki/api/v1/push"

response = requests.post(
    push_url,
    headers=self.headers,
    auth=self.auth,  # Basic authentication with API token
    json=payload,
    timeout=10
)
```

**Access Control**:
- API token required for authentication
- Organization ID (`fqsource`) provides tenant isolation
- Access controlled by Grafana Cloud's authentication system

### 3.2. Local File Protection

Local log files lack adequate protection mechanisms:

**No Encryption at Rest**:
- Log files are stored in plain text on the filesystem
- No encryption applied to log file contents
- Files are readable by any process with filesystem access

**No Access Controls**:
- Files are opened with standard Python `open()` function
- No explicit file permission settings
- No access control lists (ACLs) or additional restrictions
- Files inherit default filesystem permissions

**Evidence**:
```python
// api/ws_handler.py
def log_agent_message(self, msg):
    with open("ws_agent_messages.log", "a", encoding="utf-8") as f:
        f.write(f"[{datetime.now().isoformat()}] {msg}\n")
    # No file permission settings
    # No encryption
    # No access control
```

**File Exclusion from Version Control**:
Log files are correctly excluded from version control systems:

**Evidence**:
```gitignore
// .gitignore
ws_messages.log
ws_agent_messages.log

// .dockerignore
ws_messages.log
ws_agent_messages.log
```

However, this only prevents accidental commits; it does not protect files on the filesystem.

### 3.3. Print Interceptor Protection

The print interceptor system sends logs to Grafana Loki but also writes to stdout, which may be captured by process managers or redirected to files.

**Evidence**:
```python
// agent/tools/print_interceptor.py
def write(self, text):
    """Intercepta writes a stdout/stderr"""
    # Escribir al stdout original
    self.original_stdout.write(text)  # Still writes to stdout
    self.original_stdout.flush()
    
    # Enviar a Loki
    if not text.startswith("[LOKI]"):
        if "intermediate_step" not in text:
            self._queue_log(text.strip(), "print")
```

This means sensitive data may be:
- Written to stdout/stderr streams
- Captured by process managers (systemd, supervisor, etc.)
- Redirected to log files by the operating system
- Visible in terminal output during development

---

## 4. Log Retention and Lifecycle Management

### 4.1. Retention Policy

No explicit log retention policy is implemented for local log files. Logs are appended indefinitely without rotation or automatic deletion.

**Evidence**:
- Log files are opened in append mode (`"a"`)
- No log rotation mechanism
- No size limits
- No time-based retention
- No automatic cleanup

This creates risks:
- **Disk space exhaustion** - Logs can grow indefinitely
- **Performance degradation** - Large log files impact read/write performance
- **Compliance violations** - May violate data retention requirements
- **Security exposure** - Older sensitive data remains accessible longer than necessary

### 4.2. Grafana Loki Retention

Grafana Cloud Loki may have retention policies configured at the service level, but these are not visible in the application code. The platform relies on Grafana Cloud's default retention settings.

---

## 5. Access Control and Authorization

### 5.1. Local File Access

Local log files have no application-level access controls. Access is determined solely by filesystem permissions, which typically allow:
- Read access to the user running the application
- Read access to users in the same group
- Potentially read access to other users (depending on umask settings)

**No Application-Level Controls**:
- No authentication required to read log files
- No authorization checks
- No audit trail of log file access
- No role-based access control (RBAC)

### 5.2. Grafana Loki Access

Access to Grafana Loki logs is controlled by:
- API token authentication
- Grafana Cloud's access control system
- Organization-scoped access via `X-Scope-OrgID`

However, the API token is stored in environment variables and could be compromised if:
- Environment variables are exposed
- Configuration files are accessible
- Secrets are logged or displayed

---

## 6. Compliance and Risk Assessment

### 6.1. Sensitive Data Exposure Risks

The current logging implementation creates several risks:

1. **Local File Exposure**:
   - Sensitive user data stored unencrypted on filesystem
   - Accessible to any process with filesystem read permissions
   - No audit trail of access
   - Risk of accidental exposure through backups or file sharing

2. **No Data Minimization**:
   - All user input is logged without filtering
   - No distinction between sensitive and non-sensitive data
   - Complete conversation history stored in plain text

3. **Long-Term Retention**:
   - No automatic deletion of old logs
   - Sensitive data may be retained longer than necessary
   - Compliance with data protection regulations (GDPR, CCPA) may be at risk

4. **Development Environment Exposure**:
   - Logs visible in terminal output during development
   - Debug information may expose sensitive data
   - No separation between development and production logging

### 6.2. Protection Mechanisms in Place

The platform does implement some protection:

1. **Encryption in Transit**:
   - HTTPS for all Grafana Loki communication
   - API token authentication
   - Organization-scoped access

2. **Version Control Exclusion**:
   - Log files excluded from git and docker
   - Prevents accidental commits of sensitive data

3. **Centralized Logging**:
   - Grafana Loki provides centralized management
   - May have additional security controls at service level

---

## 7. Conclusions

### 7.1. Strengths

✅ **Centralized logging infrastructure**: Grafana Loki provides scalable, managed log storage with HTTPS encryption in transit

✅ **Version control exclusion**: Log files are properly excluded from git and docker to prevent accidental commits

✅ **No credential logging**: API keys and passwords are not logged, stored only in environment variables

✅ **Structured logging**: Logs include metadata and labels for better organization and querying

✅ **Organization isolation**: Grafana Loki uses organization-scoped access for multi-tenant isolation

### 7.2. Recommendations

1. **Implement log file encryption**: Encrypt local log files at rest using filesystem encryption or application-level encryption before writing to disk

2. **Add access controls**: Implement application-level access controls for log files, including:
   - File permission restrictions (chmod 600 or more restrictive)
   - Access logging and audit trails
   - Role-based access control for log viewing

3. **Implement data sanitization**: Add filtering and sanitization mechanisms to:
   - Redact or mask sensitive data patterns (passwords, API keys, PII)
   - Filter sensitive fields before logging
   - Implement configurable sanitization rules

4. **Add log rotation and retention**: Implement log rotation and retention policies:
   - Size-based rotation (e.g., rotate at 100MB)
   - Time-based retention (e.g., delete logs older than 90 days)
   - Automatic cleanup of old log files

5. **Separate sensitive data logging**: Consider separate logging streams for sensitive vs. non-sensitive data:
   - Sensitive data logs with enhanced protection
   - Non-sensitive operational logs with standard protection
   - Different retention policies per log type

6. **Implement log access auditing**: Add audit logging for:
   - Who accessed log files
   - When logs were accessed
   - What logs were viewed
   - Export or download activities

7. **Review Grafana Loki retention**: Verify and document Grafana Cloud Loki retention policies and ensure they align with compliance requirements

8. **Add development/production separation**: Implement different logging configurations for development and production:
   - Reduced logging in production
   - Enhanced sanitization in production
   - No sensitive data logging in development

---

## 8. Control Compliance

| Criterion | Status | Evidence |
|-----------|--------|----------|
| Sensitive data identification | ⚠️ PARTIAL | User messages and conversation content are logged, but no explicit sensitive data classification |
| Encryption in transit | ✅ COMPLIANT | HTTPS used for Grafana Loki communication with API token authentication |
| Encryption at rest | ❌ NON-COMPLIANT | Local log files stored unencrypted on filesystem |
| Access controls | ❌ NON-COMPLIANT | No application-level access controls; only filesystem permissions |
| Data sanitization | ❌ NON-COMPLIANT | No filtering or sanitization of sensitive data before logging |
| Log retention policy | ❌ NON-COMPLIANT | No retention or rotation policy for local log files |
| Version control exclusion | ✅ COMPLIANT | Log files excluded from .gitignore and .dockerignore |
| Credential protection | ✅ COMPLIANT | API keys and passwords not logged, stored in environment variables |

**FINAL VERDICT**: ⚠️ **PARTIALLY COMPLIANT** with control LOG-02. The platform implements encryption in transit for centralized logs and excludes log files from version control, but local log files contain sensitive user data without encryption at rest, access controls, or data sanitization. Log retention policies are not implemented, creating long-term exposure risks.

---

## Appendices

### A. Log File Locations

Local log files are stored in the application root directory:
- `ws_messages.log` - WebSocket message logging
- `ws_agent_messages.log` - Agent interaction logging

### B. Grafana Loki Configuration

Grafana Loki is configured with:
- **URL**: `https://logs-prod-012.grafana.net`
- **Organization ID**: `fqsource`
- **Authentication**: Basic Auth with API token
- **Endpoint**: `/loki/api/v1/push`

### C. Sensitive Data Types in Logs

The following types of sensitive data may be present in logs:
- User-provided text messages
- Conversation content and context
- User IDs and conversation IDs
- System resource information (memory, CPU usage)
- Tool call parameters and outputs
- Agent-generated responses

### D. Recommended Log Sanitization Patterns

Recommended patterns to sanitize before logging:
- Credit card numbers (regex: `\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}`)
- Social Security Numbers
- Email addresses (consider masking)
- API keys (pattern: `[A-Za-z0-9]{32,}`)
- Passwords (any field named "password" or "passwd")
- JWT tokens (pattern: `eyJ[A-Za-z0-9-_=]+\.eyJ[A-Za-z0-9-_=]+\.?[A-Za-z0-9-_.+/=]*`)

---

**End of Audit Report - Control LOG-02**


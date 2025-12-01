# Cybersecurity Audit - Control IA-01

## Control Information

- **Control ID**: IA-01
- **Control Name**: Data Usage for Training
- **Audit Date**: 2025-11-27
- **Client Question**: Do you use client data to train AI models?

---

## Executive Summary

⚠️ **PARTIAL COMPLIANCE**: The platform uses third-party AI services (OpenAI models) through external WebSocket services, but there is no explicit evidence of configuration to prevent client data from being used for model training. While the platform implements strong encryption for data at rest and in transit, data sent to AI services is transmitted in plaintext (as required for processing), and there is no documented configuration of OpenAI's data usage controls (such as disabling training data usage).

**Key Findings**:
1. **AI Model Usage** - Platform uses OpenAI models (GPT-5 variants) exclusively through external services
2. **Data Transmission** - Client data is sent to third-party WebSocket services (Railway-hosted) which then interface with OpenAI
3. **No Direct Control** - The platform does not directly configure OpenAI API settings for data usage restrictions
4. **Encryption Gap** - While sensitive data is encrypted at rest, it is decrypted before transmission to AI services
5. **Missing Configuration** - No evidence of OpenAI API configuration to disable training data usage

---

## 1. AI Model Usage Overview

### 1.1. Third-Party AI Services

The platform exclusively uses **OpenAI models** through external WebSocket services hosted on Railway infrastructure. The client application does not make direct API calls to OpenAI.

**AI Models Used**:
- **GPT-5** - Primary model for general AI operations
- **GPT-5-mini** - Used for technical and company information nodes
- **GPT-5-nano** - Used for company evaluation operations

**Evidence**:
```typescript
// src/hooks/useCompanyAutoFill.ts
const requestMessage = {
  type: 'REQUEST_COMPANY_COMPLETION',
  api_version: 'v1',
  request_id: requestId,
  auth: { supabase_jwt: session.access_token || 'test' },
  payload: {
    free_text: request.freeText || '',
    source_urls: request.urls || [],
    document_urls: request.existingPdfUrls || [],
    hints: {
      language: 'es',
      reasoning_effort: 'medium',
      verbosity: 'medium',
      model_variant: 'gpt-5',
    },
  },
} as const;
```

```typescript
// src/hooks/useProductAutoFill.tsx
hints: {
  language: 'es',
  reasoning_effort: 'medium',
  verbosity: 'medium',
  model_variant: 'gpt-5'
}
```

### 1.2. Model Configuration in Database

The platform stores AI model configuration in the database schema, allowing administrators to configure which models are used for different operations.

**Evidence**:
```sql
-- supabase/migrations/20251121132118_remote_schema_baseline.sql
"technical_info_node_model" "text" DEFAULT 'gpt-5-mini'::"text",
"technical_decision_node_model" "text" DEFAULT 'gpt-5-mini'::"text",
"company_info_node_model" "text" DEFAULT 'gpt-5-mini'::"text",
"company_decision_node_model" "text" DEFAULT 'gpt-5-mini'::"text",
"evaluation_node_model" "text" DEFAULT 'gpt-5-mini'::"text",
"company_evaluation_model" "text" DEFAULT 'gpt-5-nano'::"text",
```

---

## 2. Data Flow to AI Services

### 2.1. WebSocket Services Architecture

The platform uses external WebSocket services hosted on Railway to interface with OpenAI:

1. **Chat Service**: `wss://web-production-8e58.up.railway.app/ws`
2. **Auto-fill Service (Products)**: `wss://productscrapermembers-production.up.railway.app/ws`
3. **Embedding Generation Service**: `wss://web-production-9f433.up.railway.app/ws`

**Evidence**:
```typescript
// src/services/chatService.ts
let websocketUrl = 'wss://web-production-8e58.up.railway.app/ws';
```

```typescript
// src/hooks/useCompanyAutoFill.ts
const wsUrl = 'wss://productscrapermembers-production.up.railway.app/ws';
```

```typescript
// src/hooks/useEmbeddingGeneration.ts
const ws = new WebSocket('wss://web-production-9f433.up.railway.app/ws');
```

### 2.2. Data Transmission Process

When client data is sent to AI services:

1. **Client Application** - Sends user data (text, URLs, documents) via WebSocket
2. **External Service** - Receives data and processes it (presumably calling OpenAI APIs)
3. **OpenAI API** - Processes the data and returns responses
4. **Response Flow** - Results flow back through the WebSocket service to the client

**Evidence**:
```typescript
// src/services/chatService.ts
export async function sendChat(messages: ChatMessage[], conversationId?: string) {
  // ... connection setup ...
  
  // Send regular text message
  const msg = { type: 'message', message: lastUserMessage.content };
  socket.send(JSON.stringify(msg));
  
  // Or send multimodal message
  const multimodalMsg: MultimodalWebSocketMessage = {
    type: 'multimodal_message',
    content: {
      text: typeof lastUserMessage.content === 'string' ? lastUserMessage.content : '',
      images: lastUserMessage.images || [],
      documents: lastUserMessage.documents || []
    },
    metadata: {
      timestamp: new Date().toISOString(),
    }
  };
  socket.send(JSON.stringify(multimodalMsg));
}
```

### 2.3. Encryption and Data Protection

**Important**: While the platform implements strong encryption (AES-256-GCM) for sensitive RFX data at rest, data sent to AI services is **transmitted in plaintext** as required for processing.

**Evidence**:
```markdown
# docs/RFX_AGENT_ENCRYPTION_GUIDE.md
## Communication with Agent
The information sent to the RFX Agent via WebSocket (WSS) is sent decrypted (plain text), 
as the WSS channel provides the necessary security and the Agent requires readable data 
for processing.
```

This means that:
- ✅ Data is encrypted in the database
- ✅ Data is encrypted during transmission (WSS/TLS)
- ⚠️ Data is decrypted before being sent to AI services for processing
- ⚠️ The external WebSocket services receive plaintext data

---

## 3. OpenAI API Configuration

### 3.1. Direct API Access

The platform does **not** make direct calls to OpenAI APIs from the client application. The `openai` package (v4.39.1) is included in dependencies but is not used in client-side code.

**Evidence**:
```json
// package.json
"openai": "^4.39.1",
```

**Analysis**: The `openai` package is likely used in the backend WebSocket services (Railway-hosted), but the client application has no direct control over OpenAI API configuration.

### 3.2. Missing Data Usage Controls

**Critical Finding**: There is **no evidence** in the codebase of:
- OpenAI API configuration to disable training data usage
- Explicit opt-out of data retention for training purposes
- Documentation of OpenAI's data usage policies
- Configuration flags such as `training_data_opt_out` or similar

**OpenAI API Capabilities**: OpenAI's API supports configuration options to prevent data from being used for training:
- **Data Usage Controls**: Can be configured via API parameters
- **Enterprise Agreements**: May include explicit data usage restrictions
- **API Settings**: Can be configured at the organization or project level

**Current State**: The platform relies on external services (Railway-hosted) to interface with OpenAI, and there is no documented evidence that these services are configured to prevent training data usage.

---

## 4. Data Security Considerations

### 4.1. Data Classification

The platform handles various types of data that may be sent to AI services:

1. **Company Information** - Company profiles, descriptions, requirements
2. **Product Information** - Product specifications, technical details
3. **RFX Data** - Request for proposals, technical requirements, company requirements
4. **User Conversations** - Chat messages, queries, and interactions
5. **Documents** - PDFs, images, and other files uploaded by users

### 4.2. Encryption at Rest

Sensitive RFX data is encrypted using AES-256-GCM before storage:

**Evidence**:
```markdown
# .cursor/rules/cybersecurity.mdc
## 8.2. Cifrado de Datos en RFX
* **Campos de Texto:** Los campos sensibles (`description`, `technical_requirements`, 
  `company_requirements`) en la tabla `rfx_specs` y su histórico `rfx_specs_commits` 
  se almacenan cifrados con la clave del RFX.
```

### 4.3. Data Transmission Security

Data is transmitted over secure WebSocket connections (WSS), providing:
- ✅ **TLS Encryption** - All data in transit is encrypted
- ✅ **Authentication** - JWT tokens are used for service authentication
- ⚠️ **Plaintext Processing** - Data is decrypted before AI processing

---

## 5. Third-Party Service Dependencies

### 5.1. Railway-Hosted Services

The platform depends on external WebSocket services hosted on Railway infrastructure. These services:
- Interface with OpenAI APIs
- Process client data
- Return AI-generated responses

**Risk**: The platform has **no direct control** over:
- How these services configure OpenAI API calls
- Whether data usage restrictions are enabled
- Data retention policies of the external services
- Compliance with data protection requirements

### 5.2. Service Authentication

The platform authenticates with external services using Supabase JWT tokens:

**Evidence**:
```typescript
// src/hooks/useCompanyAutoFill.ts
auth: { supabase_jwt: session.access_token || 'test' }
```

This provides authentication but does not control data usage policies.

---

## 6. Compliance Assessment

### 6.1. OpenAI Data Usage Policies

**OpenAI's Standard Policy**: By default, OpenAI may use API data to improve their models, unless explicitly configured otherwise.

**Enterprise Options**: OpenAI offers:
- **Data Usage Controls** - Can disable training data usage
- **Zero Data Retention** - For enterprise customers
- **Custom Agreements** - May include specific data usage restrictions

**Current Status**: ⚠️ **UNKNOWN** - No evidence of configuration to prevent training data usage.

### 6.2. Platform's Data Protection Measures

| Measure | Status | Evidence |
|-----|-----|----|
| Encryption at Rest | ✅ IMPLEMENTED | AES-256-GCM for sensitive RFX data |
| Encryption in Transit | ✅ IMPLEMENTED | WSS/TLS for all communications |
| Data Usage Controls | ❌ NOT EVIDENT | No configuration found |
| Training Data Opt-Out | ❌ NOT EVIDENT | No configuration found |
| Third-Party Agreements | ❌ NOT EVIDENT | No documentation found |

---

## 7. Conclusions

### 7.1. Strengths

✅ **Strong Encryption**: The platform implements robust encryption (AES-256-GCM) for sensitive data at rest
✅ **Secure Transmission**: All data is transmitted over encrypted channels (WSS/TLS)
✅ **Authentication**: Proper authentication mechanisms (JWT) are used for service access
✅ **Data Classification**: The platform has clear data classification and encryption policies
✅ **Transparent Architecture**: The use of external services is clearly documented in the codebase

### 7.2. Recommendations

1. **Configure OpenAI Data Usage Controls**: 
   - Verify that external WebSocket services are configured to disable training data usage
   - Implement explicit API parameters (e.g., `training_data_opt_out`) if available
   - Document the configuration in the codebase or deployment documentation

2. **Establish Service Agreements**:
   - Review agreements with Railway-hosted services to ensure they comply with data usage restrictions
   - Verify that these services are configured to prevent OpenAI from using client data for training
   - Consider enterprise agreements with OpenAI if direct API access is established

3. **Document Data Usage Policies**:
   - Create explicit documentation about data usage policies for AI services
   - Include information about how client data is handled by third-party services
   - Document any agreements or configurations that prevent training data usage

4. **Implement Monitoring**:
   - Add logging/monitoring to track data sent to AI services
   - Implement audit trails for AI service interactions
   - Monitor for any changes in data usage policies

5. **Consider Direct API Access**:
   - Evaluate moving to direct OpenAI API access (with proper configuration) instead of external services
   - This would provide direct control over data usage settings
   - Would enable explicit configuration of training data opt-out

6. **Review Third-Party Service Security**:
   - Conduct security reviews of Railway-hosted services
   - Verify their compliance with data protection requirements
   - Ensure they have proper data usage controls in place

---

## 8. Control Compliance

| Criterion | Status | Evidence |
|-----|-----|----|
| Explicit confirmation that client data is not used to train models | ⚠️ PARTIAL | No explicit configuration found, but platform uses third-party services |
| AI provider statement | ❌ NOT EVIDENT | No documentation of OpenAI data usage agreements |
| Internal AI policy | ⚠️ PARTIAL | Encryption policies exist, but no specific AI data usage policy |

**FINAL VERDICT**: ⚠️ **PARTIALLY COMPLIANT** with control IA-01. The platform uses OpenAI models exclusively through third-party services, implements strong encryption for data at rest, and uses secure transmission channels. However, there is no explicit evidence of configuration to prevent client data from being used for AI model training. The platform should verify and document that external services are configured to disable training data usage, or establish direct control over OpenAI API settings with explicit opt-out configuration.

---

## Appendices

### A. OpenAI Data Usage Policy Reference

OpenAI's API documentation states that:
- By default, data sent to the API may be used for training
- Enterprise customers can opt out of training data usage
- Data usage controls can be configured via API parameters or account settings
- Zero data retention options are available for enterprise customers

**Reference**: OpenAI API Documentation - Data Usage Policies

### B. Third-Party Service Architecture

The platform uses the following external services for AI operations:

1. **Chat Service** (`wss://web-production-8e58.up.railway.app/ws`)
   - Handles conversational AI interactions
   - Processes RFX-related queries
   - Manages multimodal inputs (text, images, documents)

2. **Auto-fill Service** (`wss://productscrapermembers-production.up.railway.app/ws`)
   - Processes company and product information
   - Extracts data from URLs and documents
   - Generates structured data from unstructured inputs

3. **Embedding Service** (`wss://web-production-9f433.up.railway.app/ws`)
   - Generates embeddings for companies and products
   - Used for search and matching operations

### C. Data Flow Diagram

```
Client Application
    ↓ (Encrypted WSS)
External WebSocket Service (Railway)
    ↓ (API Call - Configuration Unknown)
OpenAI API
    ↓ (Response)
External WebSocket Service
    ↓ (Encrypted WSS)
Client Application
```

**Note**: The configuration of OpenAI API calls (including data usage settings) is controlled by the external WebSocket service, not the client application.

---

**End of Audit Report - Control IA-01**


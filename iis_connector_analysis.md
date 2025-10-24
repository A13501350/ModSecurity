# ModSecurity IIS Connector Analysis

## Overview

This document provides a comprehensive analysis of the IIS connector code implementation for ModSecurity, which provides integration between IIS's native module system and the ModSecurity web application firewall.

## Architecture and Module Structure

The IIS connector for ModSecurity implements the IIS Native Module pattern with the following key components:

### Main Components:
1. **CMyHttpModule** - The core module class inheriting from `CHttpModule`
2. **CMyHttpModuleFactory** - Factory class responsible for creating module instances
3. **REQUEST_STORED_CONTEXT** - Stores per-request ModSecurity context
4. **MODSECURITY_STORED_CONTEXT** - Stores per-configuration context

### Key Architecture Elements:
- **Entry Point**: `RegisterModule()` in `main.cpp` - called when the DLL is loaded
- **Event Registration**: Registers for `RQ_BEGIN_REQUEST`, `RQ_SEND_RESPONSE`, and `RQ_END_REQUEST` events
- **Global Context**: `g_pModuleContext` and `g_pHttpServer` for module-wide access

The architecture follows the IIS Native Module pattern, processing requests at specific lifecycle points.

## Module Lifecycle

### Module Initialization (CMyHttpModule Constructor):
- Registers event log source for ModSecurity
- Initializes system page size information
- Sets up logging hook to Windows Event Viewer
- Configures callback functions for request/response body handling
- Initializes ModSecurity engine with `modsecInit()`
- Processes configuration with `modsecStartConfig()` and `modsecFinalizeConfig()`
- Starts process-level initialization with `modsecInitProcess()`

### Module Factory Creation:
- `CMyHttpModuleFactory` creates a single instance of `CMyHttpModule` for IIS
- IIS reuses the same module instance for all requests (thread-safe design)

### Module Cleanup:
- `~CMyHttpModule()` terminates the module and deregisters the event log source
- Note: Has a temporary workaround where `modsecTerminate()` is commented out due to APR pool cleanup issues

## Key Request Processing Methods

### OnBeginRequest Method:
This method is triggered at the beginning of each HTTP request and performs:

1. **Module Configuration Retrieval:**
   - Gets configuration context using `MODSECURITY_STORED_CONTEXT::GetConfig()`
   - Checks if ModSecurity is enabled for this location
   - Processes configuration file if not already loaded

2. **Connection and Request Creation:**
   - Creates ModSecurity connection record with `modsecNewConnection()`
   - Creates request record with `modsecNewRequest()`
   - Sets up IIS-specific request body inspection flags

3. **IIS Context Setup:**
   - Creates `REQUEST_STORED_CONTEXT` to store connection and request references
   - Sets up critical section protection
   - Stores IIS context in request notes

4. **Request Data Mapping:**
   - Maps IIS request data to ModSecurity's Apache-style structures
   - Converts UTF-16 headers to UTF-8
   - Transfers all HTTP headers (known and unknown) to request headers table
   - Sets up URI, method, protocol, and client information
   - Creates request URI structure from IIS raw request data

5. **Security Processing:**
   - Calls `modsecProcessRequest()` to perform initial security rules evaluation
   - If security action is required, sets response status and marks request as handled

### OnSendResponse Method:
This method is triggered when the response is ready to be sent and performs:

1. **Response Processing Check:**
   - Verifies if response body processing is enabled
   - Gets the stored request context

2. **Response Data Collection:**
   - Reads response body data from IIS into a buffer
   - Handles both memory-based and file-based response chunks
   - Processes chunked response data from IIS

3. **Header Transfer:**
   - Transfers all IIS response headers to ModSecurity's response headers table
   - Handles both known (standard) and unknown headers

4. **Security Analysis:**
   - Calls `modsecProcessResponse()` to run response security rules
   - If security action is required, modifies response status and handles the request

5. **Request Cleanup:**
   - Calls `FinishRequest()` to clean up ModSecurity request and connection records

## Context and Request/Response Handling

The IIS connector implements sophisticated context management through:

1. **REQUEST_STORED_CONTEXT Class:**
   - Stores ModSecurity connection and request records
   - Maintains IIS context and provider references
   - Manages response buffer for response body inspection
   - Provides cleanup with `FinishRequest()` method

2. **Request/Response Body Callbacks:**
   - `ReadBodyCallback` - reads request body from IIS
   - `WriteBodyCallback` - writes request body back to IIS 
   - `ReadResponseCallback` - reads response body from internal buffer
   - `WriteResponseCallback` - writes response body back to IIS

3. **Critical Section Protection:**
   - Uses `m_csLock` to ensure thread-safe access to shared resources
   - Properly enters/leaves critical sections around sensitive operations

4. **OnPostEndRequest Method:**
   - Provides final cleanup for request context if needed
   - Called after the main request processing is complete

## Configuration Handling and Security Rule Processing

### Configuration Management:
- The system reads configuration from IIS's configuration system using the configuration path `system.webServer/ModSecurity`
- Configuration options include:
  - `enabled` - boolean flag to enable/disable ModSecurity
  - `configFile` - path to the main ModSecurity configuration file
- Configuration is loaded only once per application and cached in `MODSECURITY_STORED_CONTEXT`
- The `MODSECURITY_STORED_CONTEXT::GetConfig()` method handles configuration loading with thread safety

### Security Rule Processing:
- **Initialization:** `modsecInit()` initializes the ModSecurity engine with a server record
- **Configuration Processing:** `modsecProcessConfig()` loads and processes the ModSecurity rules from the configuration file
- **Connection Processing:** `modsecNewConnection()` and `modsecProcessConnection()` handle connection-level initialization
- **Request Processing:** `modsecNewRequest()` creates a request context and `modsecProcessRequest()` evaluates request rules
- **Response Processing:** `modsecProcessResponse()` evaluates response rules
- **Rule Evaluation:** The engine processes rules in multiple phases (connection, request headers, request body, response headers, response body)
- **Action Handling:** If security rules trigger an action, the module sets the response status and marks the request as handled

### API Integration:
- The connector uses a series of callback functions registered with the core ModSecurity engine
- These include `modsecSetReadBody()`, `modsecSetReadResponse()`, `modsecSetWriteBody()`, and `modsecSetWriteResponse()` for body inspection
- Logging is integrated with Windows Event Viewer through the `Log()` function

## Complete HTTP Request Lifecycle

Here's a comprehensive view of how an HTTP request flows through the IIS connector:

### 1. Module Registration Phase:
- IIS loads the ModSecurity IIS connector DLL
- `RegisterModule()` is called, which creates the module factory and registers for events
- The module registers for `RQ_BEGIN_REQUEST`, `RQ_SEND_RESPONSE`, and `RQ_END_REQUEST` events
- Priority is set to `PRIORITY_ALIAS_FIRST` for `BEGIN_REQUEST` and `PRIORITY_ALIAS_LAST` for `SEND_RESPONSE`

### 2. Request Arrival (OnBeginRequest):
- IIS receives an HTTP request and triggers the `OnBeginRequest` notification
- The module retrieves configuration context for this request's URL path
- If ModSecurity is disabled for this path, the module returns immediately
- If configuration hasn't been loaded yet, it processes the ModSecurity configuration file
- Creates new connection and request records for ModSecurity
- Sets up the request-stored context to track this specific request
- Maps all IIS request data (headers, URI, method, etc.) to Apache-style structures
- Calls `modsecProcessRequest()` to evaluate request against security rules
- If security action is taken, sets response status and marks request as handled

### 3. Request Processing Phase:
- The request continues to other IIS modules and the application
- Application processes the request normally
- Request body inspection is handled through callbacks if needed
- ModSecurity monitors and validates all input during this phase

### 4. Response Generation:
- Application generates response
- IIS prepares to send response to client
- The `OnSendResponse` notification is triggered

### 5. Response Processing (OnSendResponse):
- If response body inspection is enabled, the module captures the response data
- Reads all response chunks (from memory or file) into an internal buffer
- Transfers all response headers to ModSecurity's response headers
- Calls `modsecProcessResponse()` to evaluate response against security rules
- If security action is taken, modifies response status and handles the request
- Updates content-length header if necessary

### 6. Cleanup (OnPostEndRequest):
- Final cleanup occurs during the `RQ_END_REQUEST` phase
- The `OnPostEndRequest` method ensures any remaining resources are cleaned up
- ModSecurity request and connection records are finalized
- Context objects are properly destroyed

### Performance and Thread Safety Considerations:
- IIS uses a single module instance for all requests, so critical sections protect shared resources
- The module maintains its own configuration cache to avoid repeated file reads
- Request context is stored per-request using IIS's module context container system
- Memory management uses both IIS request-scoped memory and APR pools for efficient cleanup

This implementation provides a robust security layer that integrates seamlessly with IIS's native module system while maintaining compatibility with the existing ModSecurity rule engine and API.
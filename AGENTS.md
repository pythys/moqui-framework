# AGENTS

Moqui is a business automation/ERP web framework with XML-based DSLs for
entities, services, and screens. This file guides agents through feedback
loop-driven development.

## Component-Specific Instructions

**IMPORTANT**: Individual components may have their own `AGENTS.md` files that
provide component-specific instructions, conventions, and workflows. When
working within a specific component:

1. **Check for component AGENTS.md**: Look for
   `runtime/component/{component-name}/AGENTS.md`
2. **Component instructions override framework instructions**: If a component's
   AGENTS.md contradicts this file, follow the component-specific guidance
3. **Component instructions supplement framework instructions**: Component
   AGENTS.md files may add additional context, data models, workflows, or
   conventions specific to that component

Example: If working in `runtime/component/mantle-udm/`, check for
`runtime/component/mantle-udm/AGENTS.md` for entity relationship guidance,
naming conventions, or data model patterns specific to the Universal Data Model.

## Quick Start Checklist

Before starting development, verify these prerequisites:

```bash
# 1. OpenSearch running
ps aux | grep opensearch | grep -v grep
# Should show: java ...opensearch.bootstrap.OpenSearch

# 2. Data loaded (check for demo data existence)
ls -lh runtime/db/h2/moqui.mv.db
# Should exist and be > 50MB after loading demo data

# 3. Server ready (run after ./gradlew run)
grep "Started oejs.ServerConnector.*8080" runtime/log/moqui.log | tail -1
# Should show: Started oejs.ServerConnector@...{0.0.0.0:8080}

# 4. Authentication working
curl -u john.doe:moqui http://localhost:8080/rest/s1/mantle/parties
# Should return JSON with partyIdList
```

## Architecture

- **Framework** (`framework/`): Core Java/Groovy APIs, low-level services,
  entities, screens
- **Runtime** (`runtime/`): Configuration, templates, base components (tools,
  webroot)
- **Components** (`runtime/component/`): All components are optional, but most
  projects have at minimum:
  - `mantle-udm`: Universal Data Model - rich entity definitions (check here
    first for entities)
  - `mantle-usl`: Universal Service Library - business logic services (check
    here first for services)

Component structure: `entity/`, `service/`, `screen/`, `template/`, `data/`,
`src/`

## Core APIs

Access everything via ExecutionContext (`ec`):
- `ec.entity` - EntityFacade: database operations
- `ec.service` - ServiceFacade: service calls
- `ec.screen` - ScreenFacade: screen rendering
- `ec.web` - WebFacade: HTTP request/response
- `ec.user` - UserFacade: authentication/authorization
- `ec.message` - MessageFacade: user messages/errors
- `ec.logger` - LoggerFacade: logging
- `ec.l10n` - L10nFacade: localization/internationalization
- `ec.resource` - ResourceFacade: file/resource access
- `ec.cache` - CacheFacade: caching operations
- `ec.transaction` - TransactionFacade: transaction management
- `ec.artifactExecution` - ArtifactExecutionFacade: track artifact execution
- `ec.elastic` - ElasticFacade: OpenSearch operations

## DSLs

All DSLs use XML with XSDs in `framework/xsd/`. The framework-catalog.xml maps
public URLs to local schemas for validation.

### Core DSLs (most frequently used):
- **entity-definition-3.xsd**: Entity definitions (`entity/*.xml`)
- **service-definition-3.xsd**: Service definitions (`service/*.xml`)
- **xml-screen-3.xsd**: Screen definitions (`screen/*.xml`)
- **xml-form-3.xsd**: Form widgets (`form-single`, `form-list`) - embedded in
  screens
- **xml-actions-3.xsd**: Action scripts - embedded in services/screens, or
  standalone script files

### Event-Condition-Action (ECA) Rules:
- **entity-eca-3.xsd**: Entity ECAs - triggers on entity CRUD operations
- **service-eca-3.xsd**: Service ECAs - triggers before/after service calls
  (SECAs)
- **email-eca-3.xsd**: Email ECAs - triggers for email events (EECAs)

### Configuration & Integration:
- **rest-api-3.xsd**: REST API definitions (`rest.xml`)
- **moqui-conf-3.xsd**: Framework configuration (`MoquiConf.xml`)
- **common-types-3.xsd**: Common type definitions (imported by other XSDs)

## Agentic Development Workflow (Feedback Loop-Driven)

**CRITICAL**: Agents must establish continuous feedback loops when developing
Moqui code. The server must be started by the agent, and implementations should
be tested iteratively by calling services/querying entities and monitoring
server output until the code truly works.

### Initial Setup and Server Start

Moqui requires 3 servers to run: database (embedded H2 for development),
OpenSearch (for search/indexing), and Moqui itself.

**First-time setup** (one-time only):
```bash
./gradlew downloadOpenSearch    # Downloads OpenSearch to runtime/opensearch
./gradlew startElasticSearch    # Start OpenSearch in daemon mode (stays running)
```

**Standard development workflow:**
```bash
./gradlew cleanAll              # Stops OpenSearch, deletes all data (does NOT restart OpenSearch automatically)
./gradlew startElasticSearch    # Restart OpenSearch after cleanAll
./gradlew load                  # Build framework and load seed/demo data
```

**Starting the Moqui server:**

Start the server in the background and wait for it to be ready:

```bash
# IMPORTANT: Stop any existing server first to avoid port conflicts
pkill -f "gradlew run"
# Also kill any java process using port 8080 (in case Gradle spawned it)
kill $(ss -tlnp 2>/dev/null | grep :8080 | grep -oP 'pid=\K\d+' | head -1) 2>/dev/null || true
sleep 2

# Start server in background (output goes to /tmp/moqui-server.log)
nohup ./gradlew run > /tmp/moqui-server.log 2>&1 &

# Wait for server to be ready (typically within less than a minute)
for i in {1..60}; do
  if grep -q "Started oejs.ServerConnector.*8080" /tmp/moqui-server.log 2>/dev/null || \
     strings /tmp/moqui-server.log 2>/dev/null | grep -q "Started oejs.ServerConnector.*8080"; then
    echo "Server is ready!"
    break
  fi
  sleep 1
done
```

**Verifying Server is Running**:
```bash
# Test with a simple REST call
curl -u john.doe:moqui http://localhost:8080/rest/s1/mantle/parties
# Or check the log for the startup message (use strings if log is binary)
grep "Started oejs.ServerConnector.*8080" /tmp/moqui-server.log || strings /tmp/moqui-server.log | grep "Started oejs.ServerConnector.*8080"
```

**Stopping the Server**:
```bash
# IMPORTANT: Stop any existing server first to avoid port conflicts
pkill -f "gradlew run"
# Also kill any java process using port 8080 (in case Gradle spawned it)
kill $(ss -tlnp 2>/dev/null | grep :8080 | grep -oP 'pid=\K\d+' | head -1) 2>/dev/null || true
sleep 2
# Verify port is free
ss -tlnp 2>/dev/null | grep :8080 || echo "Port 8080 is free"
```

**Notes:**
- **ALWAYS stop any existing server before starting** to avoid port 8080 conflicts
- `./gradlew run` does NOT auto-start OpenSearch - start it manually first with
  `./gradlew startElasticSearch`
- Server logs available in `/tmp/moqui-server.log` AND in `runtime/log/moqui.log`
- If the server doesn't respond after seeing the log message, wait 5-10 more
  seconds for full initialization

The server starts at http://localhost:8080 with default credentials: `john.doe`
/ `moqui` (super admin user created by demo data)

**Authentication**: Most curl examples require basic authentication with `-u
john.doe:moqui` unless otherwise noted. This user is only available after
loading demo data (`./gradlew load` includes demo data by default).

### Data Management and Loading

**CRITICAL**: Services and entities require test data to function. Data must be
loaded and verified before testing.

**Data Types** (specified in XML root element `<entity-facade-xml type="...">`):
- **seed**: Core data required for system operation (users, security,
  configuration). Load before demo.
- **demo**: Test/demonstration data for development
- **all**: All data types including seed, demo, seed-initial, install, and any
  custom types (default for `./gradlew load`)

**Loading Data**:
```bash
./gradlew load              # Loads all data types (safest - handles dependencies)
./gradlew load -Ptypes=demo # Load only demo (requires other data to be loaded first)
```

**Data File Format** - Use full entity names in component's `data/` directory:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<entity-facade-xml type="demo">
    <mantle.product.Product productId="DEMO-001" productName="Demo Product"/>
    <mantle.product.category.ProductCategory productCategoryId="CAT-001"/>
</entity-facade-xml>
```

**Data Development Workflow**:
1. Create data XML in `component-name/data/*.xml` using full entity names
2. Load data: `./gradlew load -Ptypes=demo` (or `./gradlew load` for all)
3. Verify via REST (`curl -u john.doe:moqui
   http://localhost:8080/rest/s1/mantle/parties`) or EntityDataFind UI
4. Monitor server output for parsing/constraint/foreign key errors
5. Test service with loaded data
6. Iterate: modify data files and reload until working

**Ad-hoc Testing** (non-persistent):
- **DataImport UI**: http://localhost:8080/qapps/tools/Entity/DataImport
  (XML/JSON/CSV text input)
- **EntityDataFind UI**:
  http://localhost:8080/qapps/tools/Entity/DataEdit/EntityDataFind (create/edit
  records)
- **REST API**: Direct entity manipulation requires authorization permissions
  not granted by default to john.doe

**Important**: Data loads are transactional (errors rollback entire file). Use
`type="demo"` for test data, `type="seed"` for required system data.

### Service Development Feedback Loop

**Service Implementation Approaches**:

Services are always defined in `service/*.xml` and can be implemented in
different ways:
- **XML actions** (most common): Service logic in `<actions>` block using XML
  DSL
- **Inline Groovy**: Groovy code in `<script><![CDATA[...]]></script>` blocks
  within XML
- **External Groovy script**: `type="script"
  location="component://component-name/script/path/script.groovy"`
  - Has full ExecutionContext (`ec`) access, transaction management, etc.
  - Example:
    `location="component://mantle-usl/service/mantle/party/PartyServices/findParty.groovy"`
- **External Java/Groovy class**: Referenced from XML, implemented as compiled
  classes
  - Must implement ServiceRunner interface or similar
  - Used for framework extensions, not typical business logic

**Important**: The `src/` directory is for compiled Java/Groovy classes
(infrastructure, facades, runners) that do NOT have automatic ExecutionContext
access. Service scripts go in `service/` or `src/main/resources/` (with
`classpath://` location).

**Service Naming**: Service file `service/mantle/order/OrderServices.xml`
creates namespace `mantle.order.OrderServices`. Services defined as
`<service verb="get" noun="OrderInfo">` are called as
`mantle.order.OrderServices.get#OrderInfo`.

**Three Ways to Access Services**:

1. **ServiceRun UI** (Recommended for development/testing):
   - Test ANY service without modifying REST configuration

   **Browser UI**:
   - Navigate to: http://localhost:8080/qapps/tools/Service/ServiceRun
   - Enter service name: `org.moqui.impl.BasicServices.get#GeoRegionsForDropDown`
   - Fill parameters in the form and submit
   - See results immediately

   **Programmatic (curl)**:
   ```bash
   # Basic pattern - all parameters go in JSON body
   curl -X POST -H "Content-Type: application/json" \
        -u john.doe:moqui \
        -d '{"serviceName": "service.path.verb#Noun", "param1": "value1", "param2": "value2"}' \
        http://localhost:8080/apps/tools/Service/ServiceRun/runJson

   # Example: Get geo regions with parameters
   curl -X POST -H "Content-Type: application/json" \
        -u john.doe:moqui \
        -d '{"serviceName": "org.moqui.impl.BasicServices.get#GeoRegionsForDropDown", "geoId": "USA"}' \
        http://localhost:8080/apps/tools/Service/ServiceRun/runJson

   # Example: Service with no parameters (noop)
   curl -X POST -H "Content-Type: application/json" \
        -u john.doe:moqui \
        -d '{"serviceName": "org.moqui.impl.BasicServices.noop"}' \
        http://localhost:8080/apps/tools/Service/ServiceRun/runJson
   ```

   **Note**: The `runJson` endpoint accepts ANY service name and returns JSON responses - perfect for testing and automation without creating custom REST definitions.

2. **Custom REST API** (For production/integration):
   - Define in `*.rest.xml` files using resource/id structure (see
     rest-api-3.xsd)
   - `<resource name="parties">` → `/rest/{api-name}/parties`
   - `<id name="partyId">` → `/rest/{api-name}/parties/{partyId}` (path
     parameter)
   - `<resource name="contact">` → `/rest/{api-name}/parties/{partyId}/contact`
   - Body parameters from service in-parameters or entity fields
   - Example: `/rest/s1/mantle/parties` (custom REST API defined in
     mantle.rest.xml)
   - **Note**: Services must be explicitly exposed in `*.rest.xml` to be
     callable via REST

3. **Direct Service Call** (In Groovy/Java code):
   - `ec.service.sync().name("mantle.order.OrderServices.get#OrderInfo").parameter("orderId",
     orderId).call()`

**Entity Query Patterns**:
- **In Groovy scripts**:
  `ec.entity.find('mantle.product.Product').condition('productId',
  productId).one()`
- **In XML (actions DSL)**: 
  ```xml
  <entity-find-one entity-name="mantle.product.Product" value-field="product"/>
  <entity-find entity-name="mantle.order.OrderItem" list="orderItemList">
      <econdition field-name="orderId"/>
  </entity-find>
  ```

When implementing or modifying services, follow this iterative workflow:

1. **Implement/modify the service** in `service/*.xml` (with XML actions, inline
   Groovy, or script reference)

2. **Test immediately** (no server restart needed for service changes - changes
   take effect after a few seconds):

   **Option A: ServiceRun UI** (Fastest - no REST config needed):
   - Navigate to: http://localhost:8080/qapps/tools/Service/ServiceRun
   - Enter service name: `org.moqui.impl.BasicServices.get#GeoRegionsForDropDown`
   - Fill parameters in the form and submit
   - View results immediately in the UI

   **Option A (Programmatic)**: Use the `runJson` endpoint (see "Three Ways to Access Services" above for curl examples)

   **Option B: Custom REST API** (if service already exposed in `*.rest.xml`):
   ```bash
   curl -X POST -H "Content-Type: application/json" \
        -u john.doe:moqui \
        -d '{"inputMessage": "test"}' \
        http://localhost:8080/rest/s1/example/testAgent
   ```

   **Note**: Wait 3-5 seconds after saving service changes for cache refresh
   before testing.

3. **Monitor the server output** for errors and execution details:

   ```bash
   # Check server logs for errors (successful calls may not log in production mode)
   tail -30 /tmp/moqui-server.log

   # Or check the runtime log file
   tail -30 runtime/log/moqui.log

   # To observe both service response AND logs in one command:
   curl -X POST -H "Content-Type: application/json" \
        -u john.doe:moqui \
        -d '{"serviceName": "org.moqui.impl.BasicServices.get#GeoRegionsForDropDown", "geoId": "USA"}' \
        http://localhost:8080/apps/tools/Service/ServiceRun/runJson \
        && echo "" && echo "=== Checking logs ===" && tail -20 /tmp/moqui-server.log
   ```

   **What to look for in logs**:
   - Error messages and stack traces (logged with ERROR or WARN level)
   - Parameter validation failures
   - Transaction issues
   - SQL errors for entity operations
   - **Note**: Successful service calls typically don't generate detailed logs in production mode

4. **Examine the response**:
   - Check the JSON response from curl for service output parameters
   - Or view results in the ServiceRun UI form
   - Look for error messages in the response

5. **Fix issues and repeat** steps 1-4 until the service executes correctly

### Entity Development Feedback Loop

When adding or modifying entities:
1. **Define/modify entity** in `entity/*.xml` files
2. **Restart the server** (entity definition changes require restart - see "When to Restart the Server" section above)
3. **Test entity operations**:

   **Option A: EntityDataFind UI** (Recommended - most visual and interactive):
   - Navigate to:
     http://localhost:8080/qapps/tools/Entity/DataEdit/EntityDataFind?selectedEntity=YourEntityName
   - Search, create, edit, or delete records through the UI
   - Immediately see results and validation errors

   **Option B: Custom REST API** (if exposed in `*.rest.xml`):
   ```bash
   # Find/list records via custom REST API (e.g., mantle.rest.xml)
   curl -X GET -u john.doe:moqui \
        http://localhost:8080/rest/s1/mantle/parties
   ```

   **Note**: Entity REST API (/rest/e1/) requires special permissions not
   granted by default to john.doe user

4. **Monitor server output** for:
   - SQL statements being executed
   - Constraint violations
   - Relationship/foreign key errors
   - Field type mismatches

5. **Verify data integrity** by querying the entity and checking relationships

6. **Iterate** until entity structure and operations work correctly

### Screen/UI Development Feedback Loop

When developing screens (XML widget system → FreeMarker macros → HTML):

1. **Define/modify screen** in `screen/*.xml` files

2. **Reload in browser** (usually no restart needed for screen changes in dev
   mode)

3. **Navigate to the screen**: `http://localhost:8080/apps/YourApp/YourScreen`

4. **Test transitions** (form submissions, links) and **monitor server output**
   for rendering errors

5. **Inspect browser output** using developer tools (HTML, network tab, console
   errors)

6. **Fix issues and refresh** browser until UI works correctly

### When to Restart the Server
- **No restart needed**: Service XML/Groovy changes, Screen XML changes, data
  file changes
- **Restart required**: Entity definition changes, SECA/EECA rule changes,
  configuration changes (MoquiConf.xml), dependency changes
**How to Restart the Server**:
```bash
# Stop the running server
pkill -f "gradlew run"
# Also kill any java process using port 8080 (in case Gradle spawned it)
kill $(ss -tlnp 2>/dev/null | grep :8080 | grep -oP 'pid=\K\d+' | head -1) 2>/dev/null || true
sleep 2
# Start it again in background
nohup ./gradlew run > /tmp/moqui-server.log 2>&1 &
# Wait for server to be ready (typically 2-5 seconds after previous startup)
for i in {1..60}; do
  if grep -q "Started oejs.ServerConnector.*8080" /tmp/moqui-server.log 2>/dev/null || \
     strings /tmp/moqui-server.log 2>/dev/null | grep -q "Started oejs.ServerConnector.*8080"; then
    echo "Server is ready!"
    break
  fi
  sleep 1
done
```

### Key Testing Endpoints

**UI Tools** (Recommended for development):
- **Service Runner UI**: http://localhost:8080/qapps/tools/Service/ServiceRun
  - Test ANY service by name without REST configuration
  - Most flexible for iterative development
- **Entity Data Browser**:
  http://localhost:8080/qapps/tools/Entity/DataEdit/EntityDataFind
  - Browse, create, edit, delete entity records visually
  - Works with all entities regardless of permissions

**REST APIs** (For programmatic access):

Four types of REST APIs are available:

0. **ServiceRun JSON API** (`/apps/tools/Service/ServiceRun/runJson`):
   - Test ANY service programmatically without REST configuration
   - Pass service name and parameters in JSON body
   - Returns JSON response from service
   - Example curl:
     ```bash
     curl -X POST -H "Content-Type: application/json" \
          -u john.doe:moqui \
          -d '{"serviceName": "org.moqui.impl.BasicServices.get#GeoRegionsForDropDown", "geoId": "USA"}' \
          http://localhost:8080/apps/tools/Service/ServiceRun/runJson
     ```
   - Most flexible for agentic development and testing
   - Full access with john.doe credentials

1. **Custom REST API** (`/rest/s1/{api-name}/...` or `/rest/s2/...`):
   - Services/entities MUST be explicitly defined in `*.rest.xml` files
   - Provides custom resource paths (e.g., `/rest/s1/mantle/parties`)
   - Example files: `mantle.rest.xml`, `example.rest.xml`
   - Most common for agentic development
   - Full access with john.doe credentials

2. **Entity REST** (`/rest/e1/{entity.name}`):
   - Direct CRUD operations on any entity
   - Requires ENTITY_REST permissions (john.doe does NOT have these)
   - Not recommended for agents - use EntityDataFind UI instead

3. **Master Entity REST** (`/rest/m1/{entity.name}`):
   - Returns entity with related records via relationships
   - Same permissions as Entity REST (restricted)
   - Useful when entity has master definition configured

### Reading Server Output for Errors

The server console output (`./gradlew run` stdout/stderr) shows all logging.
Common error patterns:

- **Service not found**: `Could not find service with name [...]` → Check
  service name spelling and path
- **Entity not found**: `Could not find definition for entity name [...]` →
  Verify entity name
- **Missing required parameter**: `Required parameter [...] not found` → Check
  service in-parameters
- **Field constraint violation**: SQL constraint errors → Review entity field
  definitions and required flags
- **Authentication failure**: `User does not have permission` → Check service
  authenticate attribute and user permissions
- **Null pointer exceptions**: Often indicate missing data or incorrect entity
  relationships

Always read the full stack trace in the server output - Moqui provides detailed
error context including the service call stack and entity operation details.

The same logs are written to `runtime/log/moqui.log` if you need to search or
review past output.

## Troubleshooting

### System State Verification

Run these commands at any time to check system state:

```bash
# Check OpenSearch is running
ps aux | grep opensearch | grep -v grep
# Should show Java process with opensearch.bootstrap.OpenSearch

# Check Moqui server is running and ready
grep "Started oejs.ServerConnector.*8080" runtime/log/moqui.log | tail -1
# Should show: Started oejs.ServerConnector@...{0.0.0.0:8080}

# Check if data is loaded
ls -lh runtime/db/h2/moqui.mv.db
# Should be > 50MB if demo data loaded

# Test authentication
curl -u john.doe:moqui http://localhost:8080/rest/s1/mantle/parties
# Should return JSON with partyIdList
```

### Common Issues

**Server Won't Start**:
```bash
# 1. Check if OpenSearch is running
ps aux | grep opensearch | grep -v grep

# 2. If not running, start it
./gradlew startElasticSearch

# 3. Check if port 8080 is already in use
lsof -i :8080
# If occupied, kill the process or use different port

# 4. Check server logs for errors
tail -50 runtime/log/moqui.log
```

**Service Call Fails** (Decision Tree):
- **404 Error** → Service not exposed in `*.rest.xml`, use ServiceRun UI instead
- **403 Error** → Check service `authenticate` attribute and user permissions
- **500 Error** → Check server logs for stack trace and error details
- **Connection Refused** → Server not running, check with `grep ServerConnector
  runtime/log/moqui.log`

**Service Changes Not Applying**:
```bash
# 1. Wait 5 seconds for service cache to refresh
sleep 5

# 2. Check for syntax errors in server logs
tail -20 runtime/log/moqui.log | grep -i error

# 3. Verify file was saved correctly
grep "your-change-marker" path/to/YourService.xml
```

**Data Load Fails**:
```bash
# 1. Check for errors in gradle output
./gradlew load 2>&1 | grep -i "error\|exception\|constraint"

# 2. Common causes:
#    - Foreign key violations (load seed before demo)
#    - Duplicate primary keys
#    - Invalid entity names (must be fully qualified)

# 3. Load seed first, then demo
./gradlew load -Ptypes=seed
./gradlew load -Ptypes=demo
```

**Entity Changes Not Visible**:
```bash
# Entity changes REQUIRE server restart
# 1. Stop server
pkill -f "gradlew run"

# 2. Start server again in background
nohup ./gradlew run > /tmp/moqui-server.log 2>&1 &

# 3. Wait for: Started oejs.ServerConnector.*8080
for i in {1..60}; do
  if grep -q "Started oejs.ServerConnector.*8080" /tmp/moqui-server.log 2>/dev/null; then
    echo "Server is ready!"
    break
  fi
  sleep 1
done

# 4. Test with EntityDataFind UI
```

**Can't Access Entity via REST**:
- **Entity REST** (`/rest/e1/`) requires special permissions john.doe doesn't
  have
- **Use instead**: EntityDataFind UI or Custom REST API (if exposed in
  `*.rest.xml`)

## Coding standards and criteria

- Prefer existing entities and services (especially in `mantle-udm` and
  `mantle-usl`) before adding new ones.
- Use the DSLs (entity/service/screen/form/actions) where appropriate; avoid
  custom Java/Groovy when a DSL exists.
- Keep changes minimal and consistent with existing Groovy/Java style.
- Avoid editing generated/output directories (e.g., `build/`,
  `framework/build/`, `runtime/log/`, `runtime/db/`).

## Component discovery

Community components are listed in `addons.xml`. Use it to see the ecosystem and
defaults, but keep component-specific details inside each component's own
AGENTS.md.

## Documentation entry points (framework)

- Index: https://moqui.org/docs/framework
- Full page list: https://moqui.org/m/alldocs/framework
- Key references: Tool/Config Overview, Data Model, Services, XML Actions, XML
  Screen/Form, Source Management, Security.

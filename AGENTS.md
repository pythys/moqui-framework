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

# 3. Server ready (run after starting server with java -jar moqui.war)
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

### Server Management

Moqui requires 3 servers: database (embedded H2), OpenSearch, and Moqui itself.

**Initial setup** (one-time):
```bash
./gradlew downloadOpenSearch startElasticSearch load
```

**Server lifecycle commands:**
```bash
# Stop
pkill -9 -f "java.*moqui.war"; sleep 2

# Start
nohup java -jar moqui.war > /tmp/moqui-server.log 2>&1 &
# Wait: grep -q "Started.*8080" /tmp/moqui-server.log

# Restart (required for: entity changes, configuration changes, SECA/EECA changes)
# No restart needed for: service changes, screen changes, data file changes
pkill -9 -f "java.*moqui.war"; sleep 2
nohup java -jar moqui.war > /tmp/moqui-server.log 2>&1 &
```

**Verify running:**
```bash
curl -u john.doe:moqui http://localhost:8080/rest/s1/mantle/parties
```

Server at http://localhost:8080, credentials: `john.doe` / `moqui` (from demo data).

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

**CRITICAL: The Moqui server MUST be stopped before loading data. Data loading will fail or cause database conflicts if the server is running.**

```bash
# Stop server first
pkill -9 -f "java.*moqui.war"; sleep 2

# Then load data
./gradlew load              # Loads all data types (safest - handles dependencies)
./gradlew load -Ptypes=demo # Load only demo (requires other data to be loaded first)

# Restart server after loading
nohup java -jar moqui.war > /tmp/moqui-server.log 2>&1 &
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
2. **Stop the server** (required): `pkill -9 -f "java.*moqui.war"; sleep 2`
3. Load data: `./gradlew load -Ptypes=demo` (or `./gradlew load` for all)
4. **Restart the server**: `nohup java -jar moqui.war > /tmp/moqui-server.log 2>&1 &`
5. Verify via REST (`curl -u john.doe:moqui
   http://localhost:8080/rest/s1/mantle/parties`) or EntityDataFind UI
6. Monitor server output for parsing/constraint/foreign key errors
7. Test service with loaded data
8. Iterate: modify data files and reload until working (repeat from step 2)

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

The server console output (written to `/tmp/moqui-server.log`) shows all logging.
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

### Common Issues

**Port 8080 conflict** (`Failed to bind`, `Address already in use`):
```bash
pkill -9 -f "java.*moqui.war"; sleep 2
```

**OpenSearch not running**:
```bash
ps aux | grep opensearch | grep -v grep  # Verify
./gradlew startElasticSearch             # Start if needed
```

**Service changes not applying**: Wait 5 seconds for cache refresh

**Entity changes not visible**: Restart server (see Server Management section)

**Service call errors**:
- 404 → Use ServiceRun UI (`/qapps/tools/Service/ServiceRun`)
- 403 → Check permissions
- 500 → Check `tail runtime/log/moqui.log`

**Data load fails**: Server must be stopped first.

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

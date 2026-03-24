# AGENTS

Moqui is a business automation/ERP web framework with XML-based DSLs for entities, services, and screens. This file guides agents through feedback loop-driven development.

## Architecture

- **Framework** (`framework/`): Core Java/Groovy APIs, low-level services, entities, screens
- **Runtime** (`runtime/`): Configuration, templates, base components (tools, webroot)
- **Components** (`runtime/component/`): All components are optional, but most projects have at minimum:
  - `mantle-udm`: Universal Data Model - rich entity definitions (check here first for entities)
  - `mantle-usl`: Universal Service Library - business logic services (check here first for services)

Component structure: `entity/`, `service/`, `screen/`, `template/`, `data/`, `src/`

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

All DSLs use XML with XSDs in `framework/xsd/`. The framework-catalog.xml maps public URLs to local schemas for validation.

### Core DSLs (most frequently used):
- **entity-definition-3.xsd**: Entity definitions (`entity/*.xml`)
- **service-definition-3.xsd**: Service definitions (`service/*.xml`)
- **xml-screen-3.xsd**: Screen definitions (`screen/*.xml`)
- **xml-form-3.xsd**: Form widgets (`form-single`, `form-list`) - embedded in screens
- **xml-actions-3.xsd**: Action scripts - embedded in services/screens, or standalone script files

### Event-Condition-Action (ECA) Rules:
- **entity-eca-3.xsd**: Entity ECAs - triggers on entity CRUD operations
- **service-eca-3.xsd**: Service ECAs - triggers before/after service calls (SECAs)
- **email-eca-3.xsd**: Email ECAs - triggers for email events (EECAs)

### Configuration & Integration:
- **rest-api-3.xsd**: REST API definitions (`rest.xml`)
- **moqui-conf-3.xsd**: Framework configuration (`MoquiConf.xml`)
- **common-types-3.xsd**: Common type definitions (imported by other XSDs)

## Agentic Development Workflow (Feedback Loop-Driven)

**CRITICAL**: Agents must establish continuous feedback loops when developing Moqui code. The server must be started by the agent, and implementations should be tested iteratively by calling services/querying entities and monitoring server output until the code truly works.

### Initial Setup and Server Start

Moqui requires 3 servers to run: database (embedded H2 for development), OpenSearch (for search/indexing), and Moqui itself.

**First-time setup** (one-time only):
```bash
./gradlew downloadOpenSearch    # Downloads OpenSearch 3.4.0 to runtime/opensearch
./gradlew startElasticSearch    # Start OpenSearch in daemon mode (stays running)
```

**Standard development workflow:**
```bash
./gradlew cleanAll              # Stops OpenSearch, deletes all data (does NOT restart OpenSearch automatically)
./gradlew startElasticSearch    # Restart OpenSearch after cleanAll
./gradlew load                  # Build framework and load seed/demo data
```

**Starting the Moqui server:**

**IMPORTANT**: Run the Moqui server in a subagent (using the Task tool with general agent) so it runs in the background and can be easily stopped by terminating the subagent task. This keeps server logs isolated and makes process management simple.

```bash
# In subagent - this blocks and keeps running
./gradlew run
```

**Notes:**
- `./gradlew run` does NOT auto-start OpenSearch - start it manually first with `./gradlew startElasticSearch`
- Server logs available in subagent output AND in `runtime/log/moqui.log`

The server starts at http://localhost:8080 with default credentials: `john.doe` / `moqui` (super admin user created by demo data)

**Authentication**: Most curl examples require basic authentication with `-u john.doe:moqui` unless otherwise noted. This user is only available after loading demo data (`./gradlew load` includes demo data by default).

### Data Management and Loading

**CRITICAL**: Services and entities require test data to function. Data must be loaded and verified before testing.

**Data Types** (specified in XML root element `<entity-facade-xml type="...">`):
- **seed**: Core data required for system operation (users, security, configuration). Load before demo.
- **demo**: Test/demonstration data for development
- **all**: All data types including seed, demo, seed-initial, install, and any custom types (default for `./gradlew load`)

**Loading Data**:
```bash
./gradlew load              # Loads all data types (safest - handles dependencies)
./gradlew load -Ptypes=demo # Load only demo (requires seed to be loaded first)
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
3. Verify via REST (`curl -u john.doe:moqui http://localhost:8080/rest/s1/mantle/parties`) or EntityDataFind UI
4. Monitor server output for parsing/constraint/foreign key errors
5. Test service with loaded data
6. Iterate: modify data files and reload until working

**Ad-hoc Testing** (non-persistent):
- **DataImport UI**: http://localhost:8080/qapps/tools/Entity/DataImport (XML/JSON/CSV text input)
- **EntityDataFind UI**: http://localhost:8080/qapps/tools/Entity/DataEdit/EntityDataFind (create/edit records)
- **REST API**: Direct entity manipulation requires authorization permissions not granted by default to john.doe

**Important**: Data loads are transactional (errors rollback entire file). Use `type="demo"` for test data, `type="seed"` for required system data.

### Service Development Feedback Loop

**Service Implementation Approaches**:

Services are always defined in `service/*.xml` and can be implemented in different ways:
- **XML actions** (most common): Service logic in `<actions>` block using XML DSL
- **Inline Groovy**: Groovy code in `<script><![CDATA[...]]></script>` blocks within XML
- **External Groovy script**: `type="script" location="component://component-name/service/path/script.groovy"`
  - Script file lives in `service/` directory alongside XML definitions
  - Has full ExecutionContext (`ec`) access, transaction management, etc.
  - Example: `location="component://mantle-usl/service/mantle/party/PartyServices/findParty.groovy"`
- **External Java/Groovy class**: Referenced from XML, implemented as compiled classes
  - Must implement ServiceRunner interface or similar
  - Used for framework extensions, not typical business logic

**Important**: The `src/` directory is for compiled Java/Groovy classes (infrastructure, facades, runners) that do NOT have automatic ExecutionContext access. Service scripts go in `service/` or `src/main/resources/` (with `classpath://` location).

**Service Naming**: Service file `service/mantle/order/OrderServices.xml` creates namespace `mantle.order.OrderServices`. 
Services defined as `<service verb="get" noun="OrderInfo">` are called as `mantle.order.OrderServices.get#OrderInfo`.

**Accessing Services**:
- **Custom REST API**: Define in `*.rest.xml` files using resource/id structure (see rest-api-3.xsd)
  - `<resource name="parties">` → `/rest/{api-name}/parties`
  - `<id name="partyId">` → `/rest/{api-name}/parties/{partyId}` (path parameter)
  - `<resource name="contact">` → `/rest/{api-name}/parties/{partyId}/contact`
  - Body parameters from service in-parameters or entity fields
  - Example: `/rest/s1/mantle/parties` (custom REST API defined in mantle.rest.xml)
- **ServiceRun UI**: http://localhost:8080/qapps/tools/Service/ServiceRun

**Entity Query Patterns**:
- **In Groovy scripts**: `ec.entity.find('mantle.product.Product').condition('productId', productId).one()`
- **In XML (actions DSL)**: 
  ```xml
  <entity-find-one entity-name="mantle.product.Product" value-field="product"/>
  <entity-find entity-name="mantle.order.OrderItem" list="orderItemList">
      <econdition field-name="orderId"/>
  </entity-find>
  ```

When implementing or modifying services, follow this iterative workflow:

1. **Implement/modify the service** in `service/*.xml` (with XML actions, inline Groovy, or script reference)

2. **Test immediately via REST API** (no server restart needed for service changes - changes take effect after a few seconds):
   ```bash
   # Custom REST API (services must be exposed via *.rest.xml)
   curl -X POST -H "Content-Type: application/json" \
        -u john.doe:moqui \
        -d '{"inputMessage": "test"}' \
        http://localhost:8080/rest/s1/example/testAgent
   ```
   
   Or use the ServiceRun UI tool:
   - Navigate to: http://localhost:8080/qapps/tools/Service/ServiceRun
   - Enter service name: `mantle.order.OrderServices.get#OrderInfo`
   - Fill parameters and submit

3. **Monitor the server console output** from `./gradlew run` for:
   - Execution logs showing service calls
   - Error messages and stack traces
   - Parameter validation failures
   - Transaction issues

4. **Examine the response**:
   - Check the JSON response from curl for service output parameters
   - Or view results in the ServiceRun UI form
   - Look for error messages in the response

5. **Fix issues and repeat** steps 1-4 until the service executes correctly

### Entity Development Feedback Loop

When adding or modifying entities:

1. **Define/modify entity** in `entity/*.xml` files

2. **Restart the server** (entity definition changes require restart):
   ```bash
   # Stop the running server (terminate subagent task)
   # Then start it again in a new subagent
   ./gradlew run
   ```

3. **Test entity operations via REST API**:
   ```bash
   # Find/list records via custom REST API (e.g., mantle.rest.xml)
   curl -X GET -u john.doe:moqui \
        http://localhost:8080/rest/s1/mantle/parties
   
   # Note: Entity REST API (/rest/e1/) requires special permissions
   # not granted by default to john.doe user
   ```
   
   Or use the EntityDataFind UI tool:
   - Navigate to: http://localhost:8080/qapps/tools/Entity/DataEdit/EntityDataFind?selectedEntity=YourEntityName
   - Search, create, edit, or delete records through the UI

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

2. **Reload in browser** (usually no restart needed for screen changes in dev mode)

3. **Navigate to the screen**: `http://localhost:8080/apps/YourApp/YourScreen`

4. **Test transitions** (form submissions, links) and **monitor server output** for rendering errors

5. **Inspect browser output** using developer tools (HTML, network tab, console errors)

6. **Fix issues and refresh** browser until UI works correctly

### When to Restart the Server

- **No restart needed**: Service XML/Groovy changes, Screen XML changes, data file changes
- **Restart required**: Entity definition changes, SECA/EECA rule changes, configuration changes (MoquiConf.xml), dependency changes

### Key Testing Endpoints

- **Service Runner UI**: http://localhost:8080/qapps/tools/Service/ServiceRun
- **Entity Data Browser**: http://localhost:8080/qapps/tools/Entity/DataEdit/EntityDataFind
- **Custom REST APIs**: Defined in `*.rest.xml` files, paths follow resource/id structure
  - Example: `http://localhost:8080/rest/s1/mantle/parties` (mantle.rest.xml)
  - Example: `http://localhost:8080/rest/s1/example/examples` (example.rest.xml)
- **Entity REST**: `http://localhost:8080/rest/e1/{entityname}` (CRUD operations - requires special permissions)
- **Master Entity REST**: `http://localhost:8080/rest/m1/{entityname}` (entity with related records via relationships)

### Reading Server Output for Errors

The server console output (`./gradlew run` stdout/stderr) shows all logging. Common error patterns:

- **Service not found**: `Could not find service with name [...]` → Check service name spelling and path
- **Entity not found**: `Could not find definition for entity name [...]` → Verify entity name
- **Missing required parameter**: `Required parameter [...] not found` → Check service in-parameters
- **Field constraint violation**: SQL constraint errors → Review entity field definitions and required flags
- **Authentication failure**: `User does not have permission` → Check service authenticate attribute and user permissions
- **Null pointer exceptions**: Often indicate missing data or incorrect entity relationships

Always read the full stack trace in the server output - Moqui provides detailed error context including the service call stack and entity operation details.

The same logs are written to `runtime/log/moqui.log` if you need to search or review past output.

## Coding standards and criteria

- Prefer existing entities and services (especially in `mantle-udm` and `mantle-usl`) before adding new ones.
- Use the DSLs (entity/service/screen/form/actions) where appropriate; avoid custom Java/Groovy when a DSL exists.
- Keep changes minimal and consistent with existing Groovy/Java style.
- Avoid editing generated/output directories (e.g., `build/`, `framework/build/`, `runtime/log/`, `runtime/db/`).

## Component discovery

Community components are listed in `addons.xml`. Use it to see the ecosystem and defaults, but keep component-specific details inside each component's own AGENTS.md.

## Documentation entry points (framework)

- Index: https://moqui.org/docs/framework
- Full page list: https://moqui.org/m/alldocs/framework
- Key references: Tool/Config Overview, Data Model, Services, XML Actions, XML Screen/Form, Source Management, Security.

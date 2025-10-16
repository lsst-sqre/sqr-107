# TAP_SCHEMA Migration to CloudSQL - Architecture & Implementation Breakdown

```{abstract}
Currently the TAP_SCHEMA database is packaged into a Docker image and pre-populated with schema metadata, built from the `sdm_schemas` YAML schema definitions using Felis.
This image is deployed as a separate deployment (`tap-schema-db`) alongside the main TAP service.
This approach requires rebuilding and pushing Docker images for every
schema change, couples schema updates to application deployments, and results in ephemeral schema storage that is lost on pod restarts.

**Proposed Solution:** Migrate TAP_SCHEMA to a persistent CloudSQL instance, where Felis loads the schema metadata directly via a Kubernetes Job triggered by Helm hooks during our regular updates via ArgoCD.

This architecture simplifies schema management by eliminating the containerized TAP_SCHEMA database pod and removing the Docker image build cycle for schema updates. Instead of rebuilding containers for every schema change, a lightweight Helm hook job loads YAML schema definitions directly to CloudSQL during ArgoCD deployments. The TAP service connects to CloudSQL for schema metadata, gaining better backup, recovery, and monitoring capabilities. Although schema updates still flow through Phalanx (by updating schemaVersion), we improve the maintenance overhead and get less coupling between the tap application and the TAP_SCHEMA database.

**Scope:** This architecture applies to both the QServ and the Postgres backed TAP services (tap & ssotap applications).
```

---

## 2. Current State

### 2.1 Existing Architecture

#### TAP Service (`lsst-tap-service` & `tap-postgres`)

- CADC-based TAP implementation - https://github.com/opencadc/tap
- Deployed via Phalanx/ArgoCD - https://github.com/lsst-sqre/phalanx
- Uses:
  - UWS database (PostgreSQL in CloudSQL, but with an option to configure 
    with a cluster database) 
  - TAP_SCHEMA database (Cluster database deployment)

#### TAP_SCHEMA Container Image

The TAP_SCHEMA container image is built from the `sdm_schemas` repository 
and includes a MySQL database pre-populated with TAP_SCHEMA metadata. 
Felis generates SQL files from YAML schema definitions, which are used to 
prepopulate the image with TAP_SCHEMA metadata. This image is deployed as a separate Kubernetes deployment, and the TAP service connects to it via a Kubernetes service endpoint using the JDBC URL `jdbc:mysql://{{ tapSchemaAddress }}/`.

#### Schema Generation Process

- Source: `sdm_schemas/yml/*.yaml` files (https://github.com/lsst/sdm_schemas/)
- Tool: Felis (Python-based schema management tool) (https://github.com/lsst/felis)
- Build script: `sdm_schemas/tap-schema/build-all`
- Output: Docker image with pre-populated TAP_SCHEMA database

#### TAP DataLink Templates

If certain database columns are included in a TAP result, that result should contain additional service descriptors that point the user to other services that may be used in combination with that data.
This addition is done by the TAP server when constructing the result footer, but the metadata and templates for what service descriptors to include is maintained in sdm_schemas in the `datalink` directory.
That metadata needs to be available to the TAP server and match the current TAP schema.

- Source: `sdm_schemas/datalink/*.xml` and `sdm_schemas/datalink/*.json` files (https://github.com/lsst/sdm_schemas)
- Tool: `sdm-tools build-datalink-metadata` (https://github.com/lsst/sdm_tools)
- Output: `datalink-snippets.zip` included in the sdm_schemas release artifacts

#### TAP Column Metadata

Some services that wrap TAP queries need to know what sets of columns to include in their results and how to order those columns.
This metadata is derived from the table schemas maintained in sdm_schemas.
That derived metadata needs to be made available to those services.

- Source: `sdm_schemas/yml/*.yaml` files (https://github.com/lsst/sdm_schemas)
- Tool: `sdm-tools build-datalink-metadata` (https://github.com/lsst/sdm_tools)
- Output: `datalink-columns.zip` included in the sdm_schemas release artifacts

### 2.2 Problems with Current Approach

The current containerized approach creates unnecessary friction in schema 
management. Every schema update requires rebuilding a Docker image, pushing 
it to a registry, and redeploying pods which is a time-consuming process for 
what is essentially metadata changes. 

Also container storage is ephemeral, meaning schema data is lost on pod 
restarts and lacks the backup & recovery, logging and monitoring 
and robustness capabilities of CloudSQL databases. 

The URLs to the current DataLink templates and TAP column metadata have to be updated separately from the version of the schema Docker images and is often not kept up-to-date, even though that data is maintained together and should be kept in sync.

The TAP server and the datalinker server download ZIP files from GitHub on startup and unpack them in a local writable directory, which means the services have to be restarted for metadata updates, part of the local image has to be writable (not ideal for security), and frequent restarts could result in GitHub rate limiting.

There is no easy way for Science Platform services to know what version of the TAP schemas and supporting metadata is in use at runtime so that it can be checked for inconsistencies, reported in debugging output, etc.

---

## 3. Proposed Architecture

### 3.1 High-Level Design

**Core Changes:**

**TAP_SCHEMA in CloudSQL**: Move from containerized DB to persistent CloudSQL database

**Helm Hook Automation**: Trigger schema updates automatically during ArgoCD deployments

**Metadata Publication**: Metadata about a schema release is published to some public versioned URL as part of the release process

**Versioning and Discovery**: The deployed version of the schema is recorded in the database and available from service discovery along with the corresponding links to published metadata

### 3.2 TAP_SCHEMA Structure

TAP_SCHEMA consists of several metadata tables:

```sql
-- Core TAP_SCHEMA tables
tap_schema.schemas     -- List of schemas (e.g., dp02_dc2, apdb)
tap_schema.tables      -- Tables in each schema
tap_schema.columns     -- Columns in each table
tap_schema.keys        -- Foreign key relationships
tap_schema.key_columns -- Columns involved in foreign keys
```

We will add a `versions` table that contains the version number of the published schema.

**How Felis Populates These:**

Felis populates these by reading the YAML schema definition (e.g., `yml/dp02_dc2.yaml`), converting to TAP_SCHEMA INSERT/UPDATE statements which in the current setup are written as .sql scripts which are then mounted and executed during startup of the database pod.

### 3.3 Metadata Publication

On sdm_schemas release or ticket branch build, its GitHub Actions will publish the metadata generated by `sdm-tools build-datalink-metadata` to a public Google Cloud Storage bucket in addition to the existing release artifacts.
The path within that bucket will start with the version number or ticket branch tag, followed by the current file names (`w.2025.36/datalink-snippets.zip`, for example).

### 3.4 Service Discovery Integration

To expose the TAP schema version through service discovery, [Repertoire](https://repertoire.lsst.io) will be granted access to the Cloud SQL databases for the various TAP services.
It will connect to the database and read the `versions` table to determine the currently deployed schema version and include that in service discovery results.

Repertoire will also use that version information to construct URLs to the TAP DataLink templates and additional TAP metadata.
The services that need to download that data (currently the TAP servers and datalinker) will then query service discovery for those URLs and retrieve the data from those URLs.
We will continue to do this at startup or on first use for now, but the code will be written to allow for checking and updating a local cache at runtime in the future.

### 3.3 Proposed Architecture Diagram

```
[Developer] → [sdm_schemas repo] → [GitHub Release v1.2.4] → [GCS bucket]
                                           ↓
[Phalanx values.yaml] ← [Manual PR] ← [schemaVersion: v1.2.4]
         ↓
    [ArgoCD Sync]
         ↓
    [Helm pre-upgrade hook]
         ↓
    [Job: felis-updater] → [CloudSQL Proxy] → [CloudSQL: tap_schema DB]
         ↓
    [TAP Service Deployment] → [CloudSQL Proxy] → [CloudSQL: tap_schema + uws DBs]
         ↑               ↑                        ↓
     [GCS Bucket]   [Repertoire]  ←  [CloudSQL Proxy]
```

```{figure} diagram.png
:figclass: technote-wide-content
:scale: 50%

Proposed Architecture Diagram
```

---

## 4. Detailed Design

### 4.1 Schema Update Strategies

One complexity that the new architecture introduces is how to handle schema updates.
Previously we simply rebuilt the entire TAP_SCHEMA image from scratch for 
every change, and then during upgrades via GitOps the new image would 
replace the old one in a rolling update, transparent to the user.

With CloudSQL this is potentially a bit more complex because we have to
consider how to handle changes to existing schemas, additions of new schemas,
and removals of old schemas, but also how to do so while minimizing 
downtime for the user.

#### Option A: Full Replacement

- Drop existing TAP_SCHEMA tables (Or drop TAP_SCHEMA entirely)
- Recreate from scratch
- Simplest option
- Brief TAP service interruption during update

Would have to test whether a full drop of the parent TAP_SCHEMA schema or 
individual deletes of each schema is faster. Dropping the entire schema
is probably simpler, but would require re-initialization of the TAP_SCHEMA
tables.

#### Option B: Incremental Updates (Potential Future Enhancement)

- Update only changed schemas
- Keep existing schemas intact
- No service interruption
- More complex logic needed

Incremental updates would be more complex because they would potentially 
require changes to Felis.

With Felis, if each schema was loaded individually using `felis load-tap-schema`,
a rough outline of what this may require could be:
- Add `--update-mode=incremental` flag to felis load-tap-schema
- When loading a schema, check if it exists:
  - If exists, compare tables/columns
  - ALTER existing tables to match new definition
  - Add new tables/columns
  - Drop removed tables/columns (optional)

If on the other hand we chose to generate the SQL files and run the updates 
in a single transaction, incremental updates would require a custom script 
that would:
- Generate SQL for new schema: `felis load-tap-schema --dry-run new_version.yaml > new.sql`
- Query current TAP_SCHEMA state to see what exists
- Write custom diff logic to:
  - Compare new schema definition against current database state
  - Generate UPDATE statements for changed columns/tables
  - Generate INSERT statements only for new tables/columns
  - Generate DELETE statements for removed elements (optional)
- Execute the diff-generated SQL instead of full DELETE + INSERT

This approach seems more complex because, as Felis seems more geared towards
full INSERTs, so this would require much more custom logic and risk of 
errors.


#### Option C: Blue-Green Pattern (MVP)

Another option is following the blue-green pattern where we maintain two sets 
of schema tables: 

- `tap_schema` (active) 
- `tap_schema_staging` (inactive - updated during deployments) 

**Update Flow:**

1. **Load new version into staging**:
   - Clear `tap_schema_staging` (DROP CASCADE + recreate, or DELETE all data)
   - Populate `tap_schema_staging` with new schema metadata
   - Validate staging

2. **Atomic swap** (just rename, no DROP):
```sql
   BEGIN;
   ALTER SCHEMA tap_schema RENAME TO tap_schema_temp;
   ALTER SCHEMA tap_schema_staging RENAME TO tap_schema;
   ALTER SCHEMA tap_schema_temp RENAME TO tap_schema_staging;
   COMMIT;
```
   
3. **Result**: 
   - `tap_schema` now has new version (active)
   - `tap_schema_staging` now has old version (available for instant rollback)

**Rollback** (instant):
```sql
BEGIN;
ALTER SCHEMA tap_schema RENAME TO tap_schema_temp;
ALTER SCHEMA tap_schema_staging RENAME TO tap_schema;
ALTER SCHEMA tap_schema_temp RENAME TO tap_schema_staging;
COMMIT;
```

**This provides some advantages:**
- Zero-downtime updates (only brief rename)
- Instant rollback capability (just swap names again)

Postgres should in theory clear existing connections to the old schema during the rename,
so queries should be automatically routed to the new schema without requiring a restart of the TAP service.
However this would need to be tested to confirm.

#### Comparison of single transaction upgrade with DELETE vs blue-green

In terms of service disruption, both approaches would likely result in no 
downtime. However the single transaction would block queries for a brief period
while the update is in progress. Complexity-wise, single transaction is simpler to implement.
A full replacement option would also use less storage compared to 
the blue-green deployment has to at least temporarily have two copies, 
although this probably is a minimal cost since the TAP_SCHEMA tables 
shouldn't take up too much disk space.

**Conclusion:** For MVP, the plan is to implement Option C (blue-green).
This gives us a good balance of minimal downtime and simplicity.


### 4.2 Schema Distribution Methods

Another aspect of the design to consider is how the Helm hook job gets the
schema files to load. A few options exist:

**Option A: Download from GitHub Releases**
- Downloads the specified release from GitHub (`sdm_schemas` releases)
- Extracts the `schemas.tar.gz` file containing all YAML schema definitions
- Validates that the release contains the expected files

**Pros:** Simple to implement initially, no additional build steps

**Cons:** Runtime dependency on GitHub API

**Option B: Bake into Container Image (Probably Preferred for Long-Term)**

Build image: `ghcr.io/lsst/tap-schemas:v1.2.4`

Contains:
- All YAML files
- Felis tool
- Update scripts

**Pros:** No runtime GitHub dependency.

**Cons:** Requires additional CI/CD build step

Although we could also modify the existing `sdm_schemas` GitHub Actions 
workflow to build and push this image whenever a new release is created instead of the 
MySQL database.

**Option C: Mount from ConfigMap**

Commit the Felis YAML (or SQL) to the Phalanx Git repo, render them into a ConfigMap, and Felis reads locally against that.

**Pros:** Fully GitOps native

**Cons:** ConfigMap size limits & probably clutters Phalanx repo


**Option D:** Store release artifacts in GCS bucket and download from there.

This would involve modifying the `sdm_schemas` CI/CD to upload the `schemas.tar.gz` 
to a GCS bucket whenever a new release is created. 
The Helm hook job would then download from this GCS bucket.
The urls to the artifacts would be constructed based on the release version,
and would be available publicly, and discoverable via Repertoire.

**Pros:** More control over availability, no GitHub dependency

**Cons:** Requires GCS bucket management, additional complexity

For our MVP, the current plan is to implement Option D and download from GCS.

### 4.3 Update Script Logic

The update script (`update-tap-schema.sh`) will perform the following operations:

#### 1. Fetch Schemas

Based on selected distribution method (see 4.2)

#### 2. Initialize TAP_SCHEMA Tables

- Creates the standard TAP_SCHEMA tables if they don't exist:
  - `tap_schema.schemas`
  - `tap_schema.tables`
  - `tap_schema.columns`
  - `tap_schema.keys`
  - `tap_schema.key_columns`
- This probably would also be a call to a felis command `init-tap-schema` before we run the updates.
- Set up appropriate indexes for query performance

#### 3. Validate Configuration

- Parses the comma-separated list of schemas from `SCHEMAS_TO_LOAD`
- Verifies that each configured schema exists in the downloaded release
- Reports available schemas if any configured schema is missing
- If there is a validation issue we probably want to revert the update process

#### 4. Load Each Schema

For each schema in the configuration:
- Validates the YAML file using Felis (`felis validate`)
  - Generates INSERT SQL using `felis load-tap-schema --dry-run --output-file`
- **Execute DELETE + INSERT in a single transaction** (see Section 4.9 for details)
  - Delete existing schema data from all TAP_SCHEMA tables
  - Execute generated INSERT SQL

**Note:** All schemas DELETE + INSERT operations are wrapped in a single outer transaction to ensure atomicity across all schemas.

#### 5. Optional Cleanup

If `CLEANUP_OLD_SCHEMAS` is enabled:
- Identifies schemas in CloudSQL not in the configured list
- Removes obsolete schemas and their metadata

Whether this is necessary depends on if our full replacement strategy is to 
delete the metadata for each schema from the TAP_SCHEMA tabless, or to drop 
TAP_SCHEMA altogether and recreate it from scratch. If we drop and recreate 
then this step is not needed.

#### 6. Add Version

- Add or update a `versions` table with the current schema version

#### 7. Report Results (Optional)

- Should we include some sort of verification step and report somehow?
- Perhaps the simplest is to exit with appropriate status code and let the 
  Helm job report success/failure.

**Idempotency:** The script is designed to be idempotent, running it multiple times with the same schema version is safe and should produce the same result.

**Transparency:** Since the TAP services query TAP_SCHEMA tables at runtime, updates to TAP_SCHEMA in CloudSQL should be transparent, and the next query from TAP will see the new schema.

### 4.4 Helm Hook Implementation

We'll use Helm hooks to trigger schema updates, following the pattern used by other Phalanx apps like `wobbly`.

Rough draft of the Job template:

```yaml
# File: phalanx/charts/cadc-tap/templates/job-schema-update.yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: tap-schema-update-{{ .Values.tapSchema.schemaVersion | replace "." "-" }}
  namespace: {{ .Release.Namespace }}
  annotations:
    helm.sh/hook: "pre-install,pre-upgrade"
    helm.sh/hook-delete-policy: "before-hook-creation"
    helm.sh/hook-weight: "-5"
  labels:
    {{- include "cadc-tap.labels" . | nindent 4 }}
    app.kubernetes.io/component: "schema-update"
spec:
  ttlSecondsAfterFinished: 86400
  backoffLimit: 3
  template:
    metadata:
      labels:
        {{- include "cadc-tap.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: "schema-update"
    spec:
      serviceAccountName: {{ include "cadc-tap.serviceAccountName" . }}
      restartPolicy: OnFailure
      
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
      
      containers:
      - name: felis-updater
        image: ghcr.io/lsst/felis:{{ .Values.tapSchema.felisVersion | default "1.0.0" }}
        imagePullPolicy: IfNotPresent
        
        command: ["/bin/bash", "/scripts/update-tap-schema.sh"]
        
        env:
        - name: PGHOST
          value: "127.0.0.1"
        - name: PGPORT
          value: "5432"
        - name: PGDATABASE
          value: {{ .Values.tapSchema.cloudSqlDatabase | quote }}
        - name: SCHEMA_VERSION
          value: {{ .Values.tapSchema.schemaVersion | quote }}
        - name: SCHEMAS_TO_LOAD
          value: {{ .Values.tapSchema.schemas | join "," | quote }}
        - name: CLEANUP_OLD_SCHEMAS
          value: {{ .Values.tapSchema.cleanupOldSchemas | default "false" | quote }}
        
        volumeMounts:
        - name: update-script
          mountPath: /scripts
          readOnly: true
        
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
              - all
          readOnlyRootFilesystem: true
      
      # Cloud SQL Proxy sidecar
      - name: cloud-sql-proxy
        image: {{ .Values.cloudsql.image.repository }}:{{ .Values.cloudsql.image.tag }}
        imagePullPolicy: {{ .Values.cloudsql.image.pullPolicy }}
        
        args:
        - "--structured-logs"
        - "--port=5432"
        - "--max-sigterm-delay=30s"
        - {{ .Values.cloudsql.instanceConnectionName | quote }}
        
        securityContext:
          runAsNonRoot: true
          runAsUser: 65532
          allowPrivilegeEscalation: false
          capabilities:
            drop:
              - all
          readOnlyRootFilesystem: true
        
        resources:
          {{- toYaml .Values.cloudsql.resources | nindent 10 }}
      
      volumes:
      - name: update-script
        configMap:
          name: tap-schema-update-script
          defaultMode: 0755
```

**How it would work:**

1. Developer updates schema version in Phalanx
2. Sync using ArgoCD
3. Helm hook executes automatically:
   - Helm renders template with new `schemaVersion: "v1.2.4"`
   - `pre-upgrade` hook ensures job runs BEFORE the main deployment updates
   - Job `tap-schema-update-v1-2-4` is created and runs
   - Job loads schemas to CloudSQL
   - After job succeeds, the main TAP deployment proceeds
4. Job cleanup:
   - `hook-delete-policy: "before-hook-creation"` deletes previous schema update jobs before creating new ones
   - `ttlSecondsAfterFinished: 86400` keeps completed jobs for 24 hours for debugging

### 4.5 Phalanx Configuration

#### Values File Structure

(Ommmitting unchanged sections for conciseness)

```yaml
# applications/tap/values.yaml (base configuration)

cloudSQL:
  enabled: false
  instanceConnectionName: ""
  serviceAccount: ""
  image:
    repository: ""
    tag: ""
    pullPolicy: ""

uws:
  useCloudSQL: true
  cloudSqlDatabase: "uws"
  image:
    repository: ""
    pullPolicy: ""
    tag: ""
  
tapSchema:
  useCloudSQL: true
  cloudSqlDatabase: "tap_schema"
  schemaVersion: "v1.2.3"
  felisVersion: "1.0.0"
  schemas:
    - dp02_dc2
    - apdb
  cleanupOldSchemas: false

  # Keep legacy image config for cases where CloudSQL is not enabled
  image:
    repository: ""
    pullPolicy: ""
    tag: ""
```


#### Environment-Specific Configuration

Different environments may serve different data, so each needs different 
schemas loaded. The idea proposed here is to allow each env to specify which
schemas to load in their respective values files.

```yaml
# applications/tap/values-idfint.yaml
tapSchema:
  schemaVersion: "v1.2.4"
  
  # Define which schemas to load in this environment
  schemas:
    - dp02_dc2
    - dp1
```


#### Helm Chart Templates

```yaml
# applications/tap/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "cadc-tap.fullname" . }}
  labels:
    {{- include "cadc-tap.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "cadc-tap.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "cadc-tap.selectorLabels" . | nindent 8 }}
    spec:
      {{- if .Values.cloudsql.enabled }}
      serviceAccountName: {{ include "cadc-tap.serviceAccountName" . }}
      {{- end }}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      containers:
      - name: tap
        image: "{{ .Values.config.qserv.image.repository }}:{{ .Values.config.qserv.image.tag }}"
        imagePullPolicy: {{ .Values.config.qserv.image.pullPolicy }}
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        env:
        {{- if .Values.tapSchema.useCloudSQL }}
        - name: TAP_SCHEMA_JDBC_URL
          value: "jdbc:postgresql://127.0.0.1:5432/{{ .Values.tapSchema.cloudSqlDatabase }}"
        {{- else }}
        - name: TAP_SCHEMA_JDBC_URL
          value: "jdbc:mysql://{{ .Values.config.tapSchemaAddress }}"
        {{- end }}
        {{- if .Values.uws.useCloudSQL }}
        - name: UWS_JDBC_URL
          value: "jdbc:postgresql://127.0.0.1:5432/{{ .Values.uws.cloudSqlDatabase }}"
        {{- else }}
        - name: UWS_JDBC_URL
          value: "jdbc:postgresql://cadc-tap-uws-db:5432/uwsdb"
        {{- end }}
        # ... (other env vars)        

      {{- if .Values.cloudsql.enabled }}
      - name: cloud-sql-proxy
        image: "{{ .Values.cloudsql.image.repository }}:{{ .Values.cloudsql.image.tag }}"
        imagePullPolicy: {{ .Values.cloudsql.image.pullPolicy }}
        args:
        - "--structured-logs"
        - "--port=5432"
        - {{ .Values.cloudsql.instanceConnectionName | quote }}
        securityContext:
          runAsNonRoot: true
          runAsUser: 65532
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - all
          readOnlyRootFilesystem: true
        resources:
          {{- toYaml .Values.cloudsql.resources | nindent 10 }}
      {{- end }}
      volumes:
      - name: config
        configMap:
          name: {{ include "cadc-tap.fullname" . }}-config
      - name: gcs-credentials
        secret:
          secretName: {{ include "cadc-tap.fullname" . }}-gcs-credentials
```

```yaml
# applications/tap/templates/configmap-update-script.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tap-schema-update-script
  namespace: {{ .Release.Namespace }}
data:
  update-tap-schema.sh: |
{{ .Files.Get "files/update-tap-schema.sh" | indent 4 }}
```

**Note:** The tap-schema-db deployment and related resources in the cadc-tap chart need to be optional and disabled when CloudSQL is enabled for the tapSchema database.

**Single CloudSQL Instance:** The plan is to use one CloudSQL instance that hosts both `uws` and `tap_schema` databases so the above changes reflect that. If for some reason we need separate instances we can adjust accordingly in the future.

### 4.6 Testing and Validation

The update script would include some sort of built-in verification to ensure
the schemas in the configuration are all being loaded, and perhaps we may 
want to include some sort of post-deployment validation tests later.

### 4.7 Felis Functionality Analysis

#### Existing Felis Capabilities

Based on current Felis documentation and implementation, the following functionality already exists:

**1. Schema Validation** 
- Command: `felis validate` validates one or more schema files in YAML format
- Returns non-zero exit code on validation errors

**2. TAP_SCHEMA Initialization**
- Command: `felis init-tap-schema` creates an empty TAP_SCHEMA database
- Supports custom schema name via `--tap-schema-name` option
- Creates standard TAP_SCHEMA tables (schemas, tables, columns, keys, key_columns)

**3. TAP_SCHEMA Population**
- Command: `felis load-tap-schema` can generate SQL statements or update an existing TAP_SCHEMA database directly
- Can save SQL to file: `--output-file tap_schema.sql`
- Can update existing database: `--engine-url=mysql+mysqlconnector://user:password@host/TAP_SCHEMA`
- The `felis load-tap-schema` command **only performs INSERT operations**. It does NOT:
  - Check if data already exists
  - Update existing rows
  - Delete old data

**4. Database Creation**
- Command: `felis create` creates database objects from a schema including tables, columns, indexes, and constraints
- Supports environment variable `FELIS_ENGINE_URL` for database connection

#### Required Functionality

**The following exists in felis already:** 

- Initialize TAP_SCHEMA tables: `felis init-tap-schema`
- Load schema to TAP_SCHEMA: `felis load-tap-schema`
- Validate schema YAML: `felis validate`

**The following does NOT exist:**

**Incremental Updates:** No `--update-mode=incremental` flag exists
- No built-in mechanism to compare existing schema with new schema
- No ALTER TABLE support for modifying existing schemas

**Multi-Schema Management:**
- Felis loads one schema file at a time
- Our update script needs to loop through multiple schemas
- Need custom logic to handle `SCHEMAS_TO_LOAD` configuration

**Selective Schema Cleanup:**
- No built-in way to identify and remove old schemas
- Our update script needs custom logic for `CLEANUP_OLD_SCHEMAS`

**Transaction Management:**
- Felis wraps each `felis load-tap-schema` call in its own transaction and commits independently
- We cannot wrap multiple Felis calls in an outer transaction
- But, we can use `--dry-run` mode to generate SQL, then execute all SQL in 
  one script-managed transaction (see Section 4.9)

**GitHub Release Download:**
- Felis has no built-in functionality to fetch schemas from GitHub
- Update script needs to handle download and extraction


#### Implementation Requirements Summary

**What Felis provides out of the box:**
```bash

# Initialize TAP_SCHEMA
felis init-tap-schema --engine-url=postgresql://user:pass@host:port/tap_schema

# Validate a schema
felis validate --check-description --check-tap-principal dp02_dc2.yaml

# Load schema to TAP_SCHEMA
felis load-tap-schema --engine-url=postgresql://user:pass@host:port/tap_schema dp02_dc2.yaml
```

**What We Need to Build in `update-tap-schema.sh`:**

- Download and extract schema files from GitHub releases (or use pre-baked 
container)
- Initialize TAP_SCHEMA tables if they don't exist using felis `init-tap-schema`
- Parse `SCHEMAS_TO_LOAD` and iterate through schemas
- For each schema:
  - Validate with felis validate
  - DELETE existing schema data from all TAP_SCHEMA tables (required because 
   Felis only INSERTs)
  - Execute felis load-tap-schema to INSERT new data

- Transaction management: wrap DELETE + INSERT operations for ALL schemas in a single BEGIN/COMMIT block
- Cleanup old schemas if CLEANUP_OLD_SCHEMAS is enabled
- Error handling and reporting

**Future Felis Enhancements (Nice to Have):**

`felis load-tap-schema --update-mode=incremental` to update only changed schemas

`felis list-schemas --engine-url=...` to show loaded schemas

`felis remove-schema schema_name --engine-url=...` to remove a schema

### 4.8 Transaction Strategy

Felis manages its own transaction internally when loading schemas. Each `felis load-tap-schema` call commits independently, preventing us from wrapping multiple Felis calls in an outer transaction.
The workaround suggested here is to use Felis in dry-run mode to generate SQL, 
then execute all SQL in a single script-managed transaction:

```bash
# Phase 1: Clear and repopulate staging
psql <<EOF
-- Clear staging (keep structure, delete data)
DELETE FROM tap_schema_staging.key_columns;
DELETE FROM tap_schema_staging.keys;
DELETE FROM tap_schema_staging.columns;
DELETE FROM tap_schema_staging.tables;
DELETE FROM tap_schema_staging.schemas;
EOF

# Phase 2: Load new schemas into staging
for schema in $SCHEMAS_TO_LOAD; do
    felis load-tap-schema \
        --engine-url=postgresql://${PGUSER}@${PGHOST}:${PGPORT}/${PGDATABASE} \
        --tap-schema-name=tap_schema_staging \
        ${schema}.yaml
done

# Phase 3: Validate staging
psql -c "SELECT COUNT(*) FROM tap_schema_staging.schemas WHERE schema_name IN ('dp02_dc2', 'apdb');"
# (more validation tests)

# Phase 4: Atomic three-way swap
psql <<EOF
BEGIN;
ALTER SCHEMA tap_schema RENAME TO tap_schema_temp;
ALTER SCHEMA tap_schema_staging RENAME TO tap_schema;
ALTER SCHEMA tap_schema_temp RENAME TO tap_schema_staging;
COMMIT;
EOF
```

Essentially all schemas in `SCHEMAS_TO_LOAD` are updated within a single 
`BEGIN`...`COMMIT` block.

Only the schema rename operations are in a transaction (milliseconds) and data 
loading happens outside transaction in staging schema, so there would be no 
blocking of TAP service queries during data load.

The above describes the blue-green approach (Option C in Section 4.1).
If we eventually instead choose to do a full replacement the above would essentially
be simpler as we would not need to create the staging schema and could just
delete the existing schema data directly from `tap_schema` before inserting the new data.



### 4.9 Felis Docker Image

For the Helm hook job, we would need a container image with Felis installed.
This should be created via Github actions and pushed to GHCR.

### 4.10 Managing Datalink templates

Currently, the datalink template files (datalink-snippets) are packaged into a 
tarball and then the TAP service fetches them at startup from github.
With the new architecture, the current plan is that these templates would 
be pushed to GCS as part of the release.

[Repertoire](https://repertoire.lsst.io/) will be configured with a list of SQL databases (Cloud SQL when running at Google) for where the TAP schemas are written, and will conned to those databases to read the version (from the `versions` table) of the deployed schemas.
With that version information and additional Phalanx configuration setting the base path of the GCS bucket into which templates and metadata will be published, Repertoire can then construct the URLs for the corresponding metadata and include them in service discovery.

The TAP schemas for each TAP server, and therefore the corresponding versions and URLs, may be different, so Repertoire's service discovery information must include a dictionary of different TAP services to version information and links to the relevant metadata.

The TAP service will then request the link to the datalink template files
from Repertoire and fetch them (at startup at first, maybe dynamically
later), instead of storing a link to the datalink payload URL as it does
now.

There is potentially some room for improvement here in terms of how we handle
the datalinks in TAP through this template mechanism, as ideally we want 
better separation between the schema definitions and any other products 
like the datalink template files. 

However this is out of scope for this document, and may be something to consider in the future and 
outlined in it's own technote.

---

## 5. Migration Plan

### Phase 1: Infrastructure Setup

**Objective:** Update CloudSQL instance and configure access

TAP_SCHEMA will be stored as a Postgres schema in the existing CloudSQL `tap` database alongside the UWS schema.
This approach re-uses existing infrastructure, requires only a single CloudSQL proxy sidecar and keeps maintenance simple. 
TAP_SCHEMA's small size and infrequent updates mean it won't impact UWS performance.
If future requirements show the need for complete separation, the `tap_schema` schema can be migrated to a separate database.

**Tasks:**
1. Ensure existing:
   - Database tap and schema for `tap_schema`
   - Service accounts
   - Workload Identities
2. Verify service account has CREATE/DROP SCHEMA permissions on `tap` database
3. No new CloudSQL instance needed - reusing existing

**Deliverables:**
- Terraform configuration (Not obvious if anything needs to be changed)
- Service accounts configured (May re-use existing)




### Phase 2: TAP_SCHEMA Update Mechanism

**Objective:** Implement schema loading and update logic

**Tasks:**
1. Verify Felis capabilities and commands
2. Create `update-tap-schema.sh` script
3. Implement schema distribution method (Need to choose which option we want 
   to implement first)
4. Test schema loading in dev environment
5. Implement validation and error handling

**Deliverables:**
- Working update script
- Schema validation logic
- Error handling and validate rollback procedures

#### High-Level Flow of `update-tap-schema.sh`:

```bash

Parse configuration - Read SCHEMA_VERSION, SCHEMAS_TO_LOAD

Fetch schema files - Download from GCS

Initialize schemas (first time only):
    felis init-tap-schema --tap-schema-name=tap_schema
    felis init-tap-schema --tap-schema-name=tap_schema_staging

Validate all schemas - Run felis validate on each YAML file

Clear staging schema:
    DELETE all rows from tap_schema_staging tables

Load into staging:
    For each schema in SCHEMAS_TO_LOAD:
        felis load-tap-schema --tap-schema-name=tap_schema_staging schema.yaml

Validate staging:
    Verify all expected schemas exist in tap_schema_staging
    Verify each schema has tables
    Check foreign key integrity

Atomic swap:
    BEGIN TRANSACTION
        ALTER SCHEMA tap_schema RENAME TO tap_schema_temp
        ALTER SCHEMA tap_schema_staging RENAME TO tap_schema
        ALTER SCHEMA tap_schema_temp RENAME TO tap_schema_staging
    COMMIT TRANSACTION

Report results

Exit - Return 0 if succeeded, 1 if failed

```

**Note**: If we choose to delete the entire TAP_SCHEMA and recreate it from
scratch, the script becomes simpler.


### Phase 3: Phalanx Configuration Updates

**Objective:** Update Helm charts and configuration

**Tasks:**
1. Update TAP application values files
2. Create Helm hook Job template
3. Create ConfigMap for update script
4. Update deployment templates for CloudSQL connectivity
5. Make tap-schema-db deployment conditional/optional
6. Test in on dev / int environments

**Deliverables:**
- Updated Helm charts
- Environment-specific values files
- Working deployment in idfdev

### Phase 4: sdm_schemas CI/CD Updates

**Objective:** Automate schema packaging and release

**Tasks:**
1. Update GitHub Actions workflow
2. Install Felis in CI
3. Validate all schemas in CI
4. Create release assets:
   - `schemas.tar.gz` OR
   - Pre-baked container image
5. Deprecate old Docker image build process
6. Update documentation

**Deliverables:**
- Updated CI/CD pipeline
- Automated schema releases
- Deprecated old build process


**(Optional) Automating schema version updates in Phalanx with GitHub 
workflows**

If we want to provide further automation, we could set up GitHub Actions in the **sdm_schemas** repository to automatically create PRs in Phalanx when a new release is tagged.

The workflow would detect new release tags in sdm_schemas, checkout the Phalanx repository, update the `schemaVersion` in appropriate values files and create a pull request for review.
This would further reduce manual steps but would add complexity to the GitHub workflows. Manually creating a PR in Phalanx seems straightforward enough that probably makes this not worth the effort for MVP.


### Phase 5: Production Migration

**Tasks:**
1. Deploy to idfdev, then idfint
2. Monitor for a week or two
3. Deploy to production

---

## 6. Operations Runbook

### 6.1 Standard Schema Update Workflow

```bash
1. Make change in sdm_schemas repo
Example: change sdm_schemas/yml/dp02_dc2.yaml

2. Validate locally
felis validate --check-description yml/dp02_dc2.yaml

3. Create PR and release in sdm_schemas
Example: tag v1.2.3

4. Update Phalanx
In phalanx, change applications/tap/values-idfdev.yaml
Change: schemaVersion: "v1.2.3" then commit/push/PR

5. ArgoCD syncs and runs update job automatically

6. Job downloads v1.2.3 and loads all schemas

7. TAP service automatically uses new schemas
```

### 6.2 Selective Schema Update

Update only specific schemas without changing version:

```yaml
# In values file, change which schemas are loaded
tap:
  schemaVersion: "v1.2.4"
  schemas:
    - dp02_dc2
    - apdb
    - new_schema  # Add a new schema

# Commit and push - ArgoCD syncs and loads the new schema
```

### 6.3 Rollback Procedure

If issues occur after a schema update, we can rollback using GitOps via a 
rollback or a git revert + sync. By syncing to a previous version, the Helm hook job will run again 
and reload the previous schema version. The previous schema version's 
GitHub release (or docker image depending on which approach we go with) must 
still exist. If for whatever reason the previous version is not available 
the option is also there to manually restore schema from CloudSQL backup.

In the case of blue-green deployment we also have the option of swapping
the schema names in the database, since we can keep the previous version 
after a new deploy around. However this would be a manual process (or at 
best some script we can run manually) and not part of the automated
Helm hook job, unless we have some sort of flag to indicate a rollback.

**GitOps Rollback:**

1. Revert `schemaVersion` in Phalanx
2. ArgoCD sync reruns Helm hook
3. Job reloads old version from GCS into staging
4. Swap completes

**Instant Rollback using staging schema(Primary Method):**

Since we maintain both schemas permanently, rollback could also be done via:
```bash
psql <<EOF
BEGIN;
ALTER SCHEMA tap_schema RENAME TO tap_schema_temp;
ALTER SCHEMA tap_schema_staging RENAME TO tap_schema;
ALTER SCHEMA tap_schema_temp RENAME TO tap_schema_staging;
COMMIT;
EOF
```

The current plan is to use the GitOps rollback as the primary method, and 
we can always revisit and add the instant swap method later if needed.

---

## 7. Security Considerations

### 7.1 IAM Authentication

We will use IAM authentication for CloudSQL access, following the pattern used by UWS
so no password management is required.

### 7.2 Database Roles

The current design grants the felis-updater job full access to the tap_schema database. 
We should probably consider two roles here, a **reader role** used by the TAP 
service and a **writer role** used by the felis-updater job with full CRUD 
permissions on the tap_schema tables.

---

## 9. Open Questions

### Transaction Size
Our current design wraps all schemas in a single transaction. 
Are we concerned about transaction size and potential timeouts? TAP service queries against TAP_SCHEMA might be blocked during the entire update?
Note: This question is only relevant if we choose the single transaction approach instead of blue-green.


### Schema Distribution
How should the job updater get the schema files? Download from GitHub? Bake into a container image? Mount from ConfigMap?
I'm thinking start with Option A (GitHub download) for MVP or Option B (baked container) if we want to avoid runtime dependency on GitHub.

### CloudSQL Proxy Configuration
Is it ok for UWS and TAP_SCHEMA to share a single CloudSQL proxy sidecar? Or do we want separate proxies?
Current design uses single proxy on port 5432, with database selection via JDBC URL database parameter. Need to verify this works correctly.

### Update Strategy
How should we handle the update strategy? Full replacement vs incremental updates?
I'd probably aim for full replacement for MVP, implement incremental updates as future enhancement

### Version History
Do we want to maintain schema version history in the database or record the schema version in the database somehow or does 
 that add unnecessary complexity?


---

## 10. Documentation Updates Required

- Update TAP service README with new architecture
- Document new schema update process
- Add troubleshooting guide for common issues


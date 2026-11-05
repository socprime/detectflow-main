<p align="center">
  <a href="https://primedetect.live"><img src="static/image/Prime Detect_x2__1280px.png" alt="Prime Detect"></a>
</p>

<p align="center">
  <a href="https://socprime.com/detectflow/#demo-form"><img src="static/image/Button Contact sales.png" alt="Contact sales" height="48"></a>
  &nbsp;&nbsp;&nbsp;
  <a href="https://www.youtube.com/watch?v=FYI4y1YiW7c"><img src="static/image/Button Youtube.png" alt="Watch video" height="48"></a>
</p>

# Getting Started
_For the general architecrute and other info about Prime Detect, see [README.md](README.md). DetectFlow is being renamed to Prime Detect. The new name will be reflected across all resources and assets in the next release._

Prime Detect matches log events from Kafka topics with Sigma detection rules and tags each matched event with additional metadata related to the matched rules:
- ID
- Title
- Severity
- MITRE ATT&CK (sub-)technique IDs

To get started:

1. Ensure your Kafka instance has:
   - Topics with log source events you want to match with detection rules (**source topics** for Prime Detect)
   - Topics for tagged events that are consumed by a SIEM/EDR/Data Lake (**destination topics** for Prime Detect)
2. Create **repositories** with detection rules in Sigma format:
   - Local repositories
   - Cloud repositories that are synchronized with the SOC Prime Platform (optional). To enable this, set up the Threat Detection Marketplace integration (API key required)
   - Cloud repository that is synchronized with SigmaHQ GitHub public open source repository (optional)
3. Create **Log Source** configurations to parse events and map their fields to the fields of detection rules.
4. Optionally, create **Filters** to filter out some events before matching.
5. Create **Matching Pipelines** using topics, repositories, log sources, and filters from the previous steps.
6. Monitor Matching Pipeline operation on the **Dashboard**.
7. Create a **Chaining Pipeline** to correlate Sigma rule findings against stock Higher Order Sigma rules pulled during deployment.

## Accessing the Application

1. Navigate to the Admin Panel at `http://{network-service-ip}:4173`
2. Log in with the first user account:
   - **Username:** `admin@soc.local`
   - **Password:** `admin`
3. You'll be redirected to the **Pipelines** page upon successful authentication.

## Account Information

Each user can update their full name and password under **account icon → Account Settings → Account**.

> **Note:** The initial password must be changed on first login.

## Pipelines

The **Pipelines** page provides an overview of Matching Pipelines and Chaining Pipelines. The page also includes a Dashboard for monitoring the health and performance of Matching Pipelines.

### Dashboard

The Dashboard shows the health and performance of your Matching Pipelines in real time. The graph displays Source Topics, Repositories, Pipelines, and Destination Topics. Connections between them represent data flow.

**Interactions:**

- Click a Source Topic, Destination Topic, or Repository to view data flow for it. Click again to remove the selection.
- If a Matching Pipeline includes multiple items of one type (for example, three Repositories or two Source Topics), clicking one of them selects all of them.
- Click the Pipelines circle to view detailed metrics.

### Matching Pipelines

Matching Pipelines are the core processing units that consume events from Kafka, apply detection rules, and output tagged events. Each pipeline runs on its own Match Node.

#### Creating a Matching Pipeline

1. Navigate to the **Matching** tab → **New Matching Pipeline**.
2. Fill in the fields:
   - **Pipeline Name** – Unique identifier for the pipeline
   - **Source Topic** – Kafka topic to consume events from (must exist)
   - **Destination Topic** – Kafka topics to write tagged events to (must exist), each with its own settings:
     - **Output Format** – Preserve the source format or apply Log Source parsing
     - **Save Events** – All, tagged only, or untagged only. Saving only tagged events reduces storage costs upstream
   - **Repositories** – One or more rule repositories to apply (at least one required)
   - **Log Source** – Pre-configured log source with parsing script and mapping

     > **Important:** Ensure your pipeline's source topic and repositories match the Log Source configuration for optimal results.

   - **Filters** (optional) – Pre-filters to reduce false positives
   - **Custom Fields** (optional) – Static key-value pairs in YAML format to enrich all events
   - **Pipeline Runtime** (optional) – Per-pipeline override of the default runtime settings

**Important Notes:**

- Changing Source Topic or Destination Topic, as well as enabling or disabling the pipeline, requires a pipeline restart (60–90s downtime).
- Changing Repositories, Filters, Log Source, or Custom Fields uses hot-reload (zero downtime).
- The pipeline must be enabled for changes to take effect.

#### Enabling/Disabling Matching Pipelines

- **Enable** – Creates a FlinkDeployment in Kubernetes and starts processing
- **Disable** – Stops processing and removes the FlinkDeployment

### Chaining Pipelines

A Chaining Pipeline defines the configuration used to correlate events from one or more destination topics and form Attack Chains. Each Chaining Pipeline specifies the Matching Pipelines to monitor and the parameters that control Attack Chain formation.

#### Creating a Chaining Pipeline

1. Navigate to the **Chaining** tab → **New Chaining Pipeline**.
2. Fill in the fields:
   - **Pipeline Name** – Unique identifier for the pipeline
   - **Threshold for chain forming** – Minimum percentage of Higher Order Sigma rule sequences that must match tagged events to form a chain
   - **Correlation Window** – How far back in time tagged events are considered (30 seconds to 3 minutes)
   - **Auto-add New Pipelines For Monitoring** – Automatically includes new Matching Pipelines whose names match a wildcard pattern
   - **Matching Pipelines** – Pipelines used for correlation
   - **Pipeline Runtime** – Resources for the Correlation Flink Job and the Agentic Flink Job, with optional overrides

## Repositories and Rules

Repositories are folders with detection rules. It's recommended to keep rules for a single log source in each Repository so that all its rules apply to a topic with logs from that source.

Repositories come in two types:

- **Local** – Managed locally via the Prime Detect Admin Panel and not synced; suitable for air-gapped environments. Rules can be added in the editor or uploaded as files, folders, or archives (up to 5,000 rules).
- **Cloud** – Pulled from a cloud platform:
  - **SOC Prime Platform** – Synchronized in both directions via the Threat Detection Marketplace integration
  - **Third-party GitHub repositories** – SigmaHQ, Splunk, Elastic, and Microsoft Azure Sentinel (pull only)
  - **GitLab** – Synchronized via the GitLab integration

Synchronization can be turned on or off per Repository. 

Rules are automatically organized into `_product`, `_category`, and `_service` folders based on their Sigma log source definition. GitLab repositories keep the structure of the connected GitLab project.

### Creating a Local Repository

1. Go to **Repositories** and click the plus icon.
2. Select the **Local** tab, enter a name, and click **Create Local Repository**.
3. The new, empty Repository appears in the sidebar. Click it to start adding rules.

**To add rules:**

- **Upload rules** – Click **Upload** or **Upload Rules from File**. Drag and drop or select an archive, a folder, or multiple files, and click **Upload**.
- **Add a single rule** – Click **Add Rule**, enter the rule name and detection code, and click **Save**. The rule is saved to the Repository selected when you created it.

## Attack Chains

Attack Chains are possible threats inferred by correlating events tagged across all connected topics. Correlation is performed by agentic AI with Apache Flink Agents around Active Threats pulled from the SOC Prime Platform.

To configure Attack Chain formation, go to **Pipelines → Chaining** and select **New Chaining Pipeline**.

Attack Chains are formed through a 3-tier correlation process:

1. **Tier 1** – Events are matched against Sigma rules and tagged with the rule, MITRE ATT&CK techniques, and relevant context.
2. **Tier 2** – Tagged events related to the same host are evaluated against Higher Order Sigma rules that describe multi-event attack sequences within the correlation window.
3. **Tier 3** – A Flink agent combined with an LLM enriches the chain with context, analyzes the event sequence, and classifies it as a confirmed detection or a likely false positive.

The **Attack Chains** page includes:

- **Overview statistics** – Attack Chains, Monitored Threats, Affected Hostnames, Tagged Events, and Matched Rules
- **Attack Chains tab** – List of chains with severity, pipeline, hostname, username, match percentage, tagged events, matched rules, first/last seen, and status. Supports filtering and grouping by hostname or chain name
- **Attack Chain details** – AI Summary, associated Monitored Threat, a techniques-over-time chain view, detected techniques and rules, and the Higher Order Sigma rule (copy or download as YAML). Use **View in SIEM** to query related events in a connected SIEM
- **Investigation workflow** – Statuses: New, Investigating, Confirmed, False Positive, Resolved, Closed, and Filtered Out (chains the AI agentic flow marked as false positives or low priority)
- **Monitored Threats tab** – Active Threats from the last 30 days included in monitoring. Enable or disable threats per Chaining Pipeline. An Issue Navigator flags threats whose Sigma rules are unsupported, disabled, or not connected to any pipeline
- **Active Threat details** – AI-generated summary with investigation, mitigation, and response recommendations; related detections and Higher Order Sigma rules; and a visual Attack Flow based on MITRE ATT&CK

## Topics

Topics represent Kafka topics in your cluster. The system automatically discovers topics and shows their associations with pipelines.

Topics are used in:

- **Matching Pipeline Configuration** – As source and destination topics
- **Log Source Testing** – As test topics for parsing scripts
- **Dashboard Visualization** – Displayed in the pipeline graph

> **Note:** Topics must exist in Kafka before they can be used in pipelines.

## Log Sources

Log Sources define how raw events are parsed and transformed before rule matching. Each log source contains:

- **Parsing Script** – DSL query to extract and transform fields
- **Field Mapping** – YAML mapping to align parsed fields with Sigma rule fields

### Creating a Log Source

1. Navigate to **Settings → Log Sources → New Log Source**.
2. Fill in the required fields:
   - **Name** – Unique identifier for the log source
   - **Test Topics** – Kafka topics to use for testing (at least one required)
   - **Parsing Script** – DSL query using parser functions (see [Parsing DSL Functions](#parsing-dsl-functions))
   - **Mapping (YAML)** – Field mapping configuration
   - **Repositories** – Repositories to use for mapping generation

## Filters

Filters are pre-detection filters that reduce false positives by filtering events before rule matching.

### Creating a Filter

1. Navigate to **Settings** → **Filters** → **New Filter**
2. Fill in the required fields:

Required Fields

- **Filter Name** - Unique identifier for the filter
- **Filter (YAML)** - Sigma detection condition in YAML format

Filters use Sigma detection syntax:

```yaml
detection:
  condition: selection
  selection:
    EventID: 4624
    LogonType: 3
    SubjectUserName|re: '.*\$'  # Exclude machine accounts
```

Filter Logic:

- Filters are applied **before** rule matching
- Events that match the filter condition are **excluded** from rule matching
- Multiple filters can be combined (AND logic)
- Filters use the same syntax as Sigma detection conditions

## Integrations

Integrations connect Prime Detect to external platforms. Manage them on **Settings → Integrations**.

- **Threat Detection Marketplace** – Connects to the SOC Prime Platform for cloud Repositories, AI field mapping, and Active Threats
  - Requires an API key with Prime Core, Prime Architect (including AI features), and Prime Detect permissions.
  - The **Push** toggle controls whether changes made in Prime Detect are pushed back.
- **SIEM integrations** – Microsoft Sentinel, Splunk, Elastic Stack, OpenSearch
- **GitLab** – Synchronizes Sigma rules with a GitLab project. Rules edited in Prime Detect can be pushed back with **Save & Push**; new rules must be created in GitLab

## System Status

**Settings → System Status** shows the health of system components and their versions.

- **Service areas** – Kafka, Kubernetes Permissions, PostgreSQL, and Cloud Repositories. Each shows *Operational*, *Warning*, *Error*, or *Not Enabled*
- **Download Debug Logs** – Exports logs for troubleshooting

## Pipeline Runtime

**Settings → Pipeline Runtime** sets the default resources for new pipelines:

- Parallelism
- TaskManager Memory
- TaskManager CPU
- Window Size
- Checkpoint interval

> **Note:** Editing settings on this page requires an Admin role.

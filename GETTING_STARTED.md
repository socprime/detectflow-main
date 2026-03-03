# Getting Started
_For the general architecrute and other info about DetectFlow, see [README.md](README.md)._

DetectFlow matches log events from Kafka topics with Sigma detection rules and tags each matched event with additional metadata related to the matched rules:
- ID
- Title
- Severity
- MITRE ATT&CK (sub-)technique IDs

To get started:
1. Ensure your Kafka instance has:
    - Topics with log source events you want to match with detection rules (source topics for DetectFlow)
    - Topics for tagged events that are consumed by a SIEM/EDR/Data Lake (destination topics for DetectFlow)
2. Create repositories with detection rules in Sigma format:
    - Local repositories
    - Cloud repositories that are synchronized with the SOC Prime Platform (optional, API key required)
    - Cloud repositories that are synchronized with GitHub public open source repositories, including SigmaHQ, Microsoft, Splunk, and Elastic (optional)
3. Create Log Source configurations to parse events and map their fields to the fields of detection rules.
4. Optionally, create Filters to filter out some events before matching.
5. Create pipelines using topics, repositories, log sources, and filters from the previous steps.
6. Monitor pipeline operation on the Dashboard.
7. Use tagged events for threat inference: pass them to a correlation ML model, such as MITRE TIE, or build aggregations directly in your SIEM based on the metadata added to the events.

## Accessing the Application

1. Navigate to the Admin Panel URL
2. Log in with your credentials set up at deployment
3. You'll be redirected to the Dashboard upon successful authentication

## Dashboard

The Dashboard provides a real-time overview of your detection pipeline infrastructure with live metrics and visualizations.

The center of the dashboard displays an interactive graph showing:

- **Source Topics** (upper left side) - Kafka topics that feed events into pipelines
- **Repositories** (lower left side) - Rule repositories with active rule counts
- **Pipelines** (center) - Active pipelines
- **Destination Topics** (right side) - Output topics where tagged events are written

**Interactions:**
- Click on any node to view detailed information
- Nodes are color-coded based on their status and activity
- Connections between nodes represent data flow

Click the pipelines icon to view detailed statistics for all pipelines:
- Pipeline name and status
- Source and destination topics
- Input/output throughput (events per second)
- Kafka consumer lag (pending records)
- Processing status (running, suspended, failed)

## Pipelines

Pipelines are the core processing units that consume events from Kafka, apply detection rules, and output tagged events.

### Creating a Pipeline

1. Navigate to **Pipelines** → **New Pipeline**
2. Fill in the required fields:

Required Fields

- **Pipeline Name** - Unique identifier for the pipeline
- **Source Topic** - Kafka topic to consume events from (must exist)
- **Destination Topic** - Kafka topic to write tagged events to (must exist)
- **Repositories** - One or more rule repositories to apply (at least one required)
- **Log Source** - Pre-configured log source with parsing script and mapping. **Important:** Ensure your pipeline's source topic and repositories match the LogSource configuration for optimal results.

Optional Fields

- **Save Untagged Events** - Toggle to retain events that don't match any rules
- **Filters** - Pre-filters to reduce false positives (optional)
- **Custom Fields** - Static key-value pairs in YAML format to enrich all events

**Important Notes:**
- Changing **Source Topic** or **Destination Topic** as well as enabling or disabling the pipeline requires a pipeline restart (60-90s downtime)
- Changing **Repositories**, **Filters**, **Log Source**, or **Custom Fields** uses hot-reload (zero downtime)
- Pipeline must be **enabled** for changes to take effect

### Enabling/Disabling Pipelines

- **Enable** - Creates FlinkDeployment in Kubernetes and starts processing
- **Disable** - Stops processing and removes FlinkDeployment

## Log Sources

Log Sources define how raw events are parsed and transformed before rule matching. Each log source contains:

1. **Parsing Script** - DSL query to extract and transform fields
2. **Field Mapping** - YAML mapping to align parsed fields with Sigma rule fields

### Creating a Log Source

1. Navigate to **Settings** → **Log Sources** → **New Log Source**
2. Fill in the required fields:

- **Name** - Unique identifier for the log source
- **Source Topics** - Kafka topics to use for testing (at least one required)
- **Parsing Script** - DSL query using parser functions (see [Parser Functions Reference](https://github.com/socprime/detectflow-parser))
- **Mapping (YAML)** - Field mapping configuration
- **Repositories** - Repositories to use for mapping generation 

## Repositories and Rules

Repositories are collections of Sigma detection rules. There are two types:

1. **Cloud Repositories** - Synced from SOC Prime Platform or open-source GitHub repositories via API
2. **Local Repositories** - Manually created and managed

Details on the scope of supported Sigma rules can be found [here](https://github.com/socprime/detectflow-parser/blob/main/SUPPORTED_SIGMAS.md).

### Cloud Repositories

Cloud repositories are automatically synced via API:
- From the SOC Prime Platform (API key required)
- From open-source GitHub repositories

### Local Repositories

Local repositories allow you to create and manage rules independently.

Creating a Local Repository

1. Navigate to **Settings** → **Repositories**
2. Click **New Repository** (or **+** button)
3. Enter repository name
4. Click **Create Repository**

### Managing Rules in Local Repositories
Rules must be in Sigma YAML format.
**Adding Rules:**

1. Select a local repository from the sidebar
2. Click **Add Rule** button
3. Enter rule name
4. Paste the Sigma rule YAML in the editor
5. Click **Create**

**Uploading Multiple Rules:**

1. Select a local repository
2. Click **Upload** button
3. Drag and drop YAML files or click to select. You can also upload a password-protected archive.
4. Rules are automatically parsed and uploaded
5. Review upload results

## Topics

Topics represent Kafka topics in your cluster. The system automatically discovers topics and shows their associations with pipelines.

Topics are used in:

1. **Pipeline Configuration** - As source and destination topics
2. **Log Source Testing** - As test topics for parsing scripts
3. **Dashboard Visualization** - Displayed in the pipeline graph

**Note:** Topics must exist in Kafka before they can be used in pipelines.

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

# Strobes Jinja Template Fields Reference

This guide provides a comprehensive list of all available fields in the Strobes Jinja templating system for creating customized security reports.

## Basic Information

| Object | Field/Property | Type | Description |
|--------|----------------|------|-------------|
| **info** | organization_name | string | Name of the organization/client |
| | deployment_mode | string | "enterprise" or "cloud" |
| | assets_count | integer | Number of assets |
| | finding_count | integer | Total number of findings |
| | exported_on | datetime | Export timestamp |
| | exported_by | object | User who exported the report |
| | exported_by.first_name | string | First name of exporter |
| | exported_by.last_name | string | Last name of exporter |
| | exported_by.email | string | Email of exporter |
| | exported_by.scan | boolean | Whether report was scanner-generated |
| | hosted_address | string | Server address |

## Assets

| Object | Field/Property | Type | Description |
|--------|----------------|------|-------------|
| **asset** | name | string | Asset name |
| | target | string | Target URL/package/IP |
| | type | integer | Asset type ID |
| | exposed | integer | Asset exposure level ID |
| | data | object | Asset data details |
| | data.ipaddress | string | IP address for network assets |
| | data.hostname | string | Hostname for network assets |
| | data.mac_address | string | MAC address for network assets |
| | data.cpe | string | CPE for network assets |

## Findings/Vulnerabilities

| Object | Field/Property | Type | Description |
|--------|----------------|------|-------------|
| **bug** | id | string | Finding ID |
| | title | string | Finding title |
| | severity_label | string | Severity level (critical/high/medium/low/info) |
| | state_label | string | Status text (active/resolved/accepted) |
| | state | integer | Status ID (0/1=Active, 2=Resolved, 6=Accepted Risk) |
| | asset | object | Associated asset |
| | created_time | datetime | Creation time |
| | created_date | date | Creation date |
| | due_date | date | SLA due date |
| | sla_violated | boolean | Whether SLA is violated |
| | exploit_available | boolean | Exploit availability |
| | patch_available | boolean | Patch availability |
| | prioritization_score | float | Priority score |
| | cwe | list | List of CWE objects |
| | cwe[n].type | string | CWE type description |
| | cwe[n].cwe_id | string | CWE ID number |
| | cve | list | List of CVE objects |
| | cve[n].cve_id | string | CVE ID |
| | bug_tags | list | List of tag objects |
| | description | markdown | Finding description |
| | mitigation | markdown | Mitigation steps |
| | steps_to_reproduce | markdown | Steps to reproduce |
| | bug_level | integer | Bug level identifier (1=Code, 2=Web, 4=Network, 5=Cloud, 6=Package) |
| | content_object | object | Level-specific content |
| | bug_attachments | list | List of attachments |
| | reported_by | object | User who reported the vulnerability |
| | connector_config | object | Connector information |
| | attack_vector | string | CVSS score/attack vector information |
| | vulnerable_since | date | When vulnerability was first detected |

## Content Object by Bug Level

| Bug Level | Field | Description |
|-----------|-------|-------------|
| **Level 1 (Code)** | file_name | Filename |
| | start_line_number | Start line number |
| | end_line_number | End line number |
| | vulnerable_code | Vulnerable code snippet |
| **Level 2 (Web)** | endpoints_list | List of affected endpoints |
| | port | Port object |
| | request | HTTP request details |
| | response | HTTP response details |
| **Level 4 (Network)** | port | Port object |
| | cpe | CPE information |
| **Level 5 (Cloud)** | region | Cloud region |
| | vulnerable_id | Vulnerable resource ID |
| | aws_category | AWS category |
| | azure_category | Azure category |
| | azure_resource | Azure resource type |
| **Level 6 (Package)** | package_name | Package name |
| | fixed_version | Fixed version |
| | installed_version | Installed version |

## Helper Objects

| Object | Field/Property | Description |
|--------|----------------|-------------|
| **enums** | asset_type_choices | Dictionary mapping asset type IDs to names |
| | asset_exposed_choices | Dictionary mapping exposure IDs to names |
| **summary_data** | severity_metrics | Severity distribution metrics |
| | state_metrics | Status distribution metrics |
| **index_data** | [list] | Table of contents data structure |

## Common Jinja Filters

| Filter | Description | Example |
|--------|-------------|---------|
| markdownify | Render markdown as HTML | `{{ bug.description \| markdownify }}` |
| capitalize | Capitalize first letter | `{{ bug.severity_label \| capitalize }}` |
| default | Provide default value | `{{ bug.asset.target \| default('None') }}` |
| float | Convert to float | `{{ bug.prioritization_score \| float(default=0) }}` |
| length | Get length of list | `{{ bug.cwe \| length }}` |
| format_datetime | Format date objects | `{{ custom.auditEndDate_date \| format_datetime("%d %b, %Y") }}` |
| slice_str | Slice string to specific length | `{{ index.title \| slice_str(':75') }}` |

## Examples of Common Templating Patterns

### Counting Findings by Severity
```jinja
{% set critical_count = findings | selectattr('severity_label', 'equalto', 'critical') | list | length %}
{% set high_count = findings | selectattr('severity_label', 'equalto', 'high') | list | length %}
{% set medium_count = findings | selectattr('severity_label', 'equalto', 'medium') | list | length %}
{% set low_count = findings | selectattr('severity_label', 'equalto', 'low') | list | length %}
{% set info_count = findings | selectattr('severity_label', 'equalto', 'info') | list | length %}
```

### Conditional Display Based on Status
```jinja
{% if bug.state == 2 %}
    <span class="status-resolved">Resolved</span>
{% elif bug.state in [0, 1] %}
    <span class="status-active">Active</span>
{% elif bug.state == 6 %}
    <span class="status-accepted">Accepted Risk</span>
{% else %}
    <span class="status-unknown">Unknown</span>
{% endif %}
```

### Working with Asset-Specific Data
```jinja
{% if asset.type == 1 %}
    <!-- Web Asset -->
    <p>URL: {{ asset.target|default('None') }}</p>
{% elif asset.type == 2 %}
    <!-- Package Asset -->
    <p>Package: {{ asset.target|default('None') }}</p>
{% elif asset.type == 3 %}
    <!-- Network Asset -->
    <p>IP Address: {{ asset.data.ipaddress|default('None') }}</p>
    <p>Hostname: {{ asset.data.hostname|default('None') }}</p>
{% endif %}
```

# Templating guide in Strobes

## Introduction

This guide will help you create compelling security reports using Jinja templating within Strobes. By leveraging Strobes' data model and Jinja's powerful templating features, you can generate dynamic, professional reports that automatically adapt to your findings and assets.

## Understanding Strobes Data Model

In Strobes, you can access several key data structures:

### 1. Basic Information
```jinja
{{ info.organization_name }}   <!-- Organization/client name -->
{{ info.deployment_mode }}     <!-- Enterprise or cloud -->
{{ info.hosted_address }}      <!-- Server address -->
{{ info.finding_count }}       <!-- Total number of findings -->
```

### 2. Custom Fields
Custom fields are form fields that appear when generating a report:

```jinja
{{ custom.auditStartDate_date | format_datetime("%d %b, %Y") }}  <!-- Audit start date -->
{{ custom.auditEndDate_date | format_datetime("%d %b, %Y") }}    <!-- Audit end date -->
{{ custom.reportby }}          <!-- Report author -->
{{ custom.reviewedby }}        <!-- Report reviewer -->
{{ custom.statement_of_limitations_if_any_markdown | markdownify }}  <!-- Limitations section -->
{{ custom.tools_markdown | markdownify }}                            <!-- Tools used -->
{{ custom.pentesters_markdown | markdownify }}                       <!-- Pentesting team -->
```

### 3. Engagement Details
Access engagement fields through the engagement object:

```jinja
{{ engagement.fields['assessment_cycle'] }}
{{ engagement.fields['client_spoc_name'] }}
{{ engagement.fields['client_spoc_role'] }}
{{ engagement.fields['client_spoc_email'] }}
{{ engagement.fields['test_location'] }}
{{ engagement.fields['reason_for_conducting_testing'] }}
```

### 4. Assets
Iterate through assets to access their properties:

```jinja
{% for item in assets %}
    {% for asset in item %}
        <h3>{{ asset.name }}</h3>
        {% if asset.fields %}
            <p>Staging URL: {{ asset.fields['staging_url'] }}</p>
            <p>Production URL: {{ asset.fields['production_url'] }}</p>
            <p>Testing Type: {{ asset.fields['type_of_testing'] }}</p>
        {% endif %}
    {% endfor %}
{% endfor %}
```

### 5. Findings/Vulnerabilities
Process and display findings:

```jinja
{% for bug in findings %}
    <div class="vulnerability-{{ bug.severity_label }}">
        <h3>{{ bug.title }}</h3>
        <p>Severity: {{ bug.severity_label | title }}</p>
        <p>{{ bug.description | markdownify }}</p>
    </div>
{% endfor %}
```

## Advanced Jinja Techniques for Strobes Reports

### Counting and Filtering Findings

```jinja
<!-- Count findings by severity -->
{% set critical_count = findings | selectattr('severity_label', 'equalto', 'critical') | list | length %}
{% set high_count = findings | selectattr('severity_label', 'equalto', 'high') | list | length %}
{% set medium_count = findings | selectattr('severity_label', 'equalto', 'medium') | list | length %}
{% set low_count = findings | selectattr('severity_label', 'equalto', 'low') | list | length %}
{% set info_count = findings | selectattr('severity_label', 'equalto', 'info') | list | length %}

<!-- Count by status -->
{% set active_findings = findings | selectattr('state', 'in', [0, 1]) | list | length %}
{% set resolved_findings = findings | selectattr('state', 'equalto', 2) | list | length %}
{% set accepted_risk = findings | selectattr('state', 'equalto', 6) | list | length %}

<!-- Total issues -->
{% set total_issues = critical_count + high_count + medium_count + low_count + info_count %}
```

### Creating Dynamic Charts with SVG

#### Pie Chart for Risk Distribution

```html
<svg width="400" height="400" viewBox="-200 -200 400 400">
    {% set total = critical_count + high_count + medium_count + low_count + info_count %}
    {% set data = [
        {'value': high_count, 'color': '#EE7077', 'label': 'High'},
        {'value': medium_count, 'color': '#FEB297', 'label': 'Medium'},
        {'value': low_count, 'color': '#FCDBA6', 'label': 'Low'},
        {'value': info_count, 'color': '#BAE6AA', 'label': 'Info'}
    ] %}
    
    {% set start_angle = 0 %}
    {% for item in data %}
        {% if item.value > 0 %}
            {% set angle = (item.value / total * 360) %}
            {% set rad = 3.14159 * 2 %}
            {% set x1 = 150 * cos(start_angle * rad / 360) %}
            {% set y1 = 150 * sin(start_angle * rad / 360) %}
            {% set x2 = 150 * cos((start_angle + angle) * rad / 360) %}
            {% set y2 = 150 * sin((start_angle + angle) * rad / 360) %}
            
            <path 
                d="M 0,0 L {{ x1 }},{{ y1 }} A 150,150 0 {{ '1' if angle > 180 else '0' }},1 {{ x2 }},{{ y2 }} Z"
                fill="{{ item.color }}"
            />
            <!-- Label -->
            {% set label_angle = start_angle + (angle / 2) %}
            {% set label_x = 170 * cos(label_angle * rad / 360) %}
            {% set label_y = 170 * sin(label_angle * rad / 360) %}
            <text x="{{ label_x }}" y="{{ label_y }}" style="font-size: 14px; font-weight: bold;">
                {{ item.label }}: {{ item.value }}
            </text>
            
            {% set start_angle = start_angle + angle %}
        {% endif %}
    {% endfor %}
</svg>
```

#### OWASP Top 10 Bar Chart

```html
<svg width="100%" height="400" viewBox="0 0 800 400">
    <!-- Grid Lines -->
    {% for i in range(7) %}
    <line 
        x1="50" 
        y1="{{ 350 - (i * 50) }}" 
        x2="750" 
        y2="{{ 350 - (i * 50) }}" 
        stroke="#ddd" 
        stroke-width="1" 
        stroke-dasharray="5,5"
    />
    <text 
        x="30" 
        y="{{ 355 - (i * 50) }}" 
        text-anchor="end" 
        style="font-size: 12px; fill: #666;">{{ i }}
    </text>
    {% endfor %}

    <!-- OWASP Categories -->
    {% set categories = [
        {'id': 'A1', 'name': 'Broken Access Control', 'count': findings|selectattr('fields.owasp_category', 'equalto', 'Broken Access Control')|list|length},
        {'id': 'A2', 'name': 'Cryptographic Failures', 'count': findings|selectattr('fields.owasp_category', 'equalto', 'Cryptographic Failures')|list|length},
        {'id': 'A3', 'name': 'Injection', 'count': findings|selectattr('fields.owasp_category', 'equalto', 'Injection')|list|length}
        <!-- Add other OWASP categories -->
    ] %}

    {% for cat in categories %}
        {% set x = loop.index0 * 140 + 100 %}
        {% set height = cat.count * 50 %}
        <rect 
            x="{{ x }}"
            y="{{ 350 - height }}"
            width="40"
            height="{{ height }}"
            fill="{% if cat.count > 0 %}#FF6384{% else %}#4CAF50{% endif %}"
        />
        <text 
            x="{{ x + 20 }}"
            y="370"
            text-anchor="middle"
            transform="rotate(45 {{ x + 20 }}, 370)"
            style="font-size: 12px;">{{ cat.name }}
        </text>
        <text 
            x="{{ x + 20 }}"
            y="{{ 345 - height }}"
            text-anchor="middle"
            style="font-size: 14px; font-weight: bold;">{{ cat.count }}
        </text>
    {% endfor %}

    <!-- Axes -->
    <line x1="50" y1="350" x2="750" y2="350" stroke="black" stroke-width="2"/>
    <line x1="50" y1="50" x2="50" y2="350" stroke="black" stroke-width="2"/>
</svg>
```

### Conditional Formatting

```html
<!-- Color-coding vulnerabilities by severity -->
<div class="vulnerability" style="
    background-color: {% if bug.severity_label == 'critical' %}#EE7077{% 
                      elif bug.severity_label == 'high' %}#FEB297{% 
                      elif bug.severity_label == 'medium' %}#FCDBA6{% 
                      elif bug.severity_label == 'low' %}#FFF1C0{% 
                      else %}#BAE6AA{% endif %};
    padding: 15px;
    margin: 10px 0;
    border-radius: 5px;
">
    <h3>{{ bug.title }}</h3>
    <p>{{ bug.description }}</p>
</div>

<!-- Status indicators -->
<span class="status-badge" style="
    background-color: {% if bug.state == 2 %}#4CAF50{% 
                      elif bug.state == 1 %}#EE7077{% 
                      elif bug.state == 6 %}#FFC107{% 
                      else %}#EE7077{% endif %};
    color: white;
    padding: 3px 8px;
    border-radius: 3px;
    font-size: 12px;
">
    {% if bug.state == 2 %}
        Resolved
    {% elif bug.state == 0 or bug.state == 1 %}
        Active
    {% elif bug.state == 6 %}
        Accepted Risk
    {% else %}
        Active
    {% endif %}
</span>
```

### Rendering Safe Content

When working with user-provided content, use proper rendering:

```jinja
<!-- Safe markdown rendering -->
<div class="description">
    {{ bug.description | markdownify }}
</div>

<!-- Handling fields that might not exist -->
{% if bug.fields %}
    {% for key, value in bug.fields.items() %}
        {% if key == 'short_recommendation' %}
            <div class="recommendation">{{ value | markdownify }}</div>
        {% endif %}
    {% endfor %}
{% endif %}
```

## Best Practices for Strobes Reports

1. **Check for Field Existence**: Always check if fields exist before accessing them
   ```jinja
   {% if asset.fields and 'staging_url' in asset.fields %}
      {{ asset.fields['staging_url'] }}
   {% endif %}
   ```

2. **Provide Default Values**: Use default values for fields that might be empty
   ```jinja
   {{ bug.mitigation | default('No mitigation steps provided') }}
   ```

3. **Use Format Filters**: Format dates and numbers consistently
   ```jinja
   {{ custom.auditEndDate_date | format_datetime("%d %b, %Y") }}
   ```

4. **Organize Data for Charts**: Pre-process data before creating visualizations
   ```jinja
   {% set severity_counts = {
       'critical': findings | selectattr('severity_label', 'equalto', 'critical') | list | length,
       'high': findings | selectattr('severity_label', 'equalto', 'high') | list | length,
       'medium': findings | selectattr('severity_label', 'equalto', 'medium') | list | length,
       'low': findings | selectattr('severity_label', 'equalto', 'low') | list | length,
       'info': findings | selectattr('severity_label', 'equalto', 'info') | list | length
   } %}
   ```

5. **Keep Styles Separate**: Use CSS classes instead of inline styles where possible
   ```html
   <style>
       .severity-critical { background-color: #EE7077; color: white; }
       .severity-high { background-color: #FEB297; }
       .severity-medium { background-color: #FCDBA6; }
       .severity-low { background-color: #FFF1C0; }
       .severity-info { background-color: #BAE6AA; }
   </style>
   
   <div class="severity-{{ bug.severity_label }}">
       {{ bug.title }}
   </div>
   ```

## Example: Complete Executive Summary Section

```html
<div class="section executive-summary">
    <h2 class="section-title">Executive Summary</h2>
    
    <p>Security assessment was performed on <strong>{% for asset in assets %}{{ asset.name }}{% endfor %}</strong> application from 
    {{ custom.auditStartDate_date | format_datetime("%d %b, %Y") }} to {{ custom.auditEndDate_date | format_datetime("%d %b, %Y") }}.</p>
    
    <h3>Vulnerability Overview</h3>
    
    {% set critical_count = findings | selectattr('severity_label', 'equalto', 'critical') | list | length %}
    {% set high_count = findings | selectattr('severity_label', 'equalto', 'high') | list | length %}
    {% set medium_count = findings | selectattr('severity_label', 'equalto', 'medium') | list | length %}
    {% set low_count = findings | selectattr('severity_label', 'equalto', 'low') | list | length %}
    {% set info_count = findings | selectattr('severity_label', 'equalto', 'info') | list | length %}
    
    <div class="summary-table">
        <table class="info-table">
            <tr>
                <th>Risk Level</th>
                <th>Count</th>
                <th>Recommended Action</th>
            </tr>
            <tr class="risk-critical">
                <td>Critical</td>
                <td>{{ critical_count }}</td>
                <td>Immediate action required</td>
            </tr>
            <tr class="risk-high">
                <td>High</td>
                <td>{{ high_count }}</td>
                <td>Fix within 7 days</td>
            </tr>
            <tr class="risk-medium">
                <td>Medium</td>
                <td>{{ medium_count }}</td>
                <td>Fix within 30 days</td>
            </tr>
            <tr class="risk-low">
                <td>Low</td>
                <td>{{ low_count }}</td>
                <td>Fix within 90 days</td>
            </tr>
            <tr class="risk-info">
                <td>Informational</td>
                <td>{{ info_count }}</td>
                <td>Review as needed</td>
            </tr>
        </table>
    </div>
    
    <!-- Security Posture Determination -->
    {% if critical_count > 0 or high_count > 0 %}
        {% set security_posture = 'weak' %}
    {% elif medium_count > 0 or low_count > 0 %}
        {% set security_posture = 'moderate' %}
    {% elif info_count > 0 %}
        {% set security_posture = 'good' %}
    {% else %}
        {% set security_posture = 'excellent' %}
    {% endif %}
    
    <div class="security-posture" style="
        color: {{ '#EE7077' if security_posture == 'weak' else 
                 '#FF9933' if security_posture == 'moderate' else 
                 '#70AD47' if security_posture == 'good' else 
                 '#00CC00' if security_posture == 'excellent' else '' }};
        font-size: 28px;
        font-weight: 100;
        text-transform: uppercase;
        margin: 20px 0;
    ">
        SECURITY POSTURE: {{ security_posture|upper }}
    </div>
</div>
```

## Conclusion

Mastering Jinja templating in Strobes allows you to create professional, data-driven security reports that accurately reflect your findings and provide actionable insights. By leveraging the techniques in this guide, you can transform raw security data into compelling visual narratives that effectively communicate risks and recommendations to stakeholders.

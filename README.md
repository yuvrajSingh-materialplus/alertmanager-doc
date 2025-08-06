# Alert Routing Implementation

This guide details the steps to configure severity-based alert routing for Prometheus and Alertmanager.

Critical alerts will be sent to Squadcast.
Warning alerts will be sent to Microsoft Teams.

## Implementation Steps

### Step 1: Add MS Teams Webhook to Ansible Variables
Store the MS Teams webhook URL as a secure variable.
Action: Add the msteams_webhook_url variable to your Ansible variables file (e.g., group_vars/all.yml).
```
# In your Ansible variables file (e.g., group_vars/all.yml)

# Your existing Squadcast URL variable
squadcast_url: "YOUR_EXISTING_SQUADCAST_URL"

# Add this new variable for the MS Teams webhook
msteams_webhook_url: "<PASTE_YOUR_MS_TEAMS_WEBHOOK_URL_HERE>"

```
### Step 2: Update the Alertmanager Template

Replace the current Alertmanager configuration with the new routing logic.
Action: Replace the entire content of roles/alertmanager/templates/alertmanager.yml.j2 with the code below.

```
global:
  resolve_timeout: 5m

route:
  # Default receiver if no other routes match. This does nothing.
  receiver: 'null-receiver'

  # Default timing for all alerts.
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 3h
  
  # Group alerts by these labels to combine related notifications.
  group_by: ['alertname', 'cluster', 'service']

  # A list of sub-routes that are checked in order.
  routes:
    # --- ROUTE FOR CRITICAL ALERTS ---
    - receiver: 'squadcast-criticals'
      matchers:
        - severity = "critical"
      # Stop processing further routes if this one matches.
      continue: false

    # --- ROUTE FOR WARNING ALERTS ---
    - receiver: 'msteams-warnings'
      matchers:
        - severity = "warning"

# This section defines all possible destinations for your alerts.
receivers:
  # Receiver for CRITICAL alerts -> Squadcast
  - name: 'squadcast-criticals'
    webhook_configs:
      - url: '{{ squadcast_url }}'
        send_resolved: true

  # Receiver for WARNING alerts -> Microsoft Teams
  - name: 'msteams-warnings'
    msteams_configs:
      - webhook_url: '{{ msteams_webhook_url }}'
        send_resolved: true
        title: '[WARNING] {{ .CommonLabels.alertname }}'
        text: >-
          {{ range .Alerts }}
          **Summary:** {{ .Annotations.summary }}<br>
          **Description:** {{ .Annotations.description }}<br>
          **Instance:** {{ .Labels.host | default .Labels.instance }}<br>
          **Service:** {{ .Labels.service | default "N/A" }}
          {{ end }}

  # A "do-nothing" receiver for any alerts that don't match the routes above.
  - name: 'null-receiver'

```
## Deployment and Verification Steps
### Step 3: Deploy the Changes with Ansible

Action: Run the playbook targeting the alertmanager role.
```
cd ~/ren3-debian-vm-ansible/playbooks/
ansible-playbook deploy-monitoring.yml 
```

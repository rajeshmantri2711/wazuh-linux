
<div align="center">

<h1>A comprehensive guide for installing and configuring Wazuh on Linux systems</h1>

</div>

---

## About Wazuh

Wazuh is a **free, open-source security platform** that monitors systems, detects threats, analyzes logs, and ensures compliance. It combines **SIEM** (Security Information and Event Management) and **HIDS** (Host-based Intrusion Detection System) capabilities to protect endpoints and servers in real time.

### Key Features

- **Threat Detection** - Real-time monitoring and alerting
- **Log Analysis** - Comprehensive log management and correlation
- **Intrusion Detection** - Advanced threat detection capabilities
- **File Integrity Monitoring** - Track changes to critical files
---

## Supported Operating Systems

### Wazuh Manager And Dashboard

Currently Wazuh supports the following operating system versions:

| OS Family | Versions |
|-----------|----------|
| **Ubuntu** | 16.04, 18.04, 20.04, 22.04, 24.04 |
| **Red Hat Enterprise Linux** | 7, 8, 9 |
| **CentOS** | 7, 8 |
| **Amazon Linux** | 2, 2023 |

---
<div align="center">
<h2>Installation Guide</h2>
</div>
### Prerequisites

Ensure you have `curl` installed on your system:

```bash
sudo apt update -y
sudo apt install curl -y
```

### Wazuh Manager & Dashboard Installation

#### Step 1: Download and Install

Install Wazuh Manager and Dashboard on Ubuntu 22.04:

```bash
sudo curl -sO https://packages.wazuh.com/4.5/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

#### Step 2: Access Dashboard

After installation, access the dashboard using the default admin credentials:

| Field | Value |
|-------|-------|
| **User** | `admin` |
| **Password** | `<ADMIN_PASSWORD>` |

**Dashboard URLs:**
- `https://<WAZUH_DASHBOARD_IP_ADDRESS>`
- `https://localhost`

**Forgot Password?** Retrieve it using:
```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

#### Step 3: Service Management

**Check Service Status:**
```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
sudo systemctl status filebeat
```

**Start Services (if not running):**
```bash
sudo systemctl start wazuh-manager
sudo systemctl start wazuh-dashboard
sudo systemctl start filebeat
```

**Enable Services at Startup:**
```bash
sudo systemctl enable wazuh-manager
sudo systemctl enable wazuh-dashboard
sudo systemctl enable filebeat
```

### Agent Management

**Manage Agents:**
```bash
sudo /var/ossec/bin/manage_agents
```

**View Active Agents:**
```bash
sudo /var/ossec/bin/agent_control -l
```

---

## Uninstalling Wazuh Manager

### Step 1: Stop Services
```bash
sudo systemctl stop wazuh-manager
sudo systemctl stop wazuh-dashboard
sudo systemctl stop filebeat
sudo systemctl stop wazuh-indexer
```

### Step 2: Disable Services
```bash
sudo systemctl disable wazuh-manager
sudo systemctl disable wazuh-dashboard
sudo systemctl disable filebeat
sudo systemctl disable wazuh-indexer
```

### Step 3: Remove Configuration Files
```bash
sudo rm -rf /var/ossec
sudo rm -rf /etc/filebeat
sudo rm -rf /usr/share/wazuh-dashboard
sudo rm -rf /etc/wazuh-indexer
sudo rm -rf /var/lib/wazuh-indexer
sudo rm -rf /usr/share/wazuh-indexer
sudo rm -rf /var/log/wazuh-indexer
```

**Remove Packages:**
```bash
sudo dpkg --purge wazuh-manager wazuh-dashboard filebeat wazuh-indexer
```

### Step 4: Verify Removal
```bash
dpkg -l | grep wazuh
```

---

## Wazuh Agent Installation

### About Wazuh Agents

Wazuh agents are **software components** installed on endpoints (servers, workstations, cloud instances, etc.) that you want to monitor. They collect security data and log information from the host system and send it to the Wazuh Manager for analysis and further processing.

### Agent Installation Process

**Prerequisites:** Ensure `curl` is installed on your target machine.

#### Step 1: Automated Installation

Run the following command on your Debian/Ubuntu machine:

```bash
sudo curl -s https://raw.githubusercontent.com/rajeshmantri2711/wazuh-linux/main/Linux-Install.sh | bash
```
> **Note:** This script fetches and installs the latest supported agent version.
> You can inspect the script [here](https://github.com/rajeshmantri2711/wazuh-linux/blob/main/Linux-Install.sh)

#### Step 2: Enable and Start Agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

**Check Agent Status:**
```bash
sudo grep ^status /var/ossec/var/run/wazuh-agentd.state
```

---
<div align="center">
<h2>Integrating Suricata for Network Intrusion Detection</h2>
</div>

To enhance threat detection with network-level monitoring, you can integrate Suricata. This allows Wazuh to analyze network traffic and generate alerts for suspicious activity. These steps should be performed on the machine running the **Wazuh agent**.

### Step 1: Install Suricata

```bash
sudo apt update
sudo apt install suricata -y
```

### Step 2: Download Suricata Rules

Download the Emerging Threats Open ruleset, a widely used and effective set of open-source IDS rules.

```bash
cd /tmp/
curl -LO https://rules.emergingthreats.net/open/suricata-6.0.8/emerging.rules.tar.gz
sudo tar -xvzf emerging.rules.tar.gz
# The rules path may already exist
sudo mkdir -p /etc/suricata/rules
sudo mv rules/*.rules /etc/suricata/rules/
sudo chmod 640 /etc/suricata/rules/*.rules
```

### Step 3: Configure Suricata

Edit the Suricata configuration file at `/etc/suricata/suricata.yaml` and make the following changes:

1.  **Set your local network address:**
    Replace the default `HOME_NET` with the IP address of your agent machine.

    ```yaml
    address-groups:
      HOME_NET: "[10.10.10.10]"   # Change this to your agent's IP
    ```

2.  **Set the monitoring interface:**
    Find the `af-packet` section and specify the network interface Suricata should monitor.
    > You can find your interface name using the `ip addr` command.

    ```yaml
    af-packet:
      - interface: eth0 # Change this to your network interface
    ```

3.  **Verify rule paths:**
    Ensure the rule path and files are correctly configured. This is usually the default setting.
    ```yaml
    default-rule-path: /etc/suricata/rules
    rule-files:
      - "*.rules"
    ```

### Step 4: Update Rules and Start Suricata

Run `suricata-update` to process the new rules and then start and enable the Suricata service.

```bash
sudo suricata-update
sudo systemctl enable --now suricata
```

### Step 5: Configure Wazuh Agent for Suricata

Edit the Wazuh agent configuration file at `/var/ossec/etc/ossec.conf` on the agent machine and add the following block to tell the agent to read Suricata's JSON alert log:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

### Step 6: Restart Services

For all the configuration changes to take effect, restart the Wazuh agent and the Wazuh manager.

```bash
# On the Wazuh Agent machine
sudo systemctl restart wazuh-agent

# On the Wazuh Manager machine
sudo systemctl restart wazuh-manager
```

You should now be able to see Suricata network alerts in your Wazuh dashboard.

---

## Uninstalling Wazuh Agent

### Step 1: Stop and Disable
```bash
sudo systemctl stop wazuh-agent
sudo systemctl disable wazuh-agent
```

### Step 2: Remove Agent
```bash
sudo dpkg --purge wazuh-agent
sudo rm -rf /var/ossec
```

### Step 3: Verify Removal
```bash
dpkg -l | grep wazuh
```

---

## Additional Resources

For more detailed information, visit the [Wazuh Documentation](https://documentation.wazuh.com/current/getting-started/index.html).

---

## Credits

This project makes use of resources and code from:

- [**Wazuh GitHub Repository**](https://github.com/wazuh/wazuh) - Official Wazuh source code
- [**Wazuh Official Website**](https://wazuh.com) - Documentation and resources

**Special thanks** to the Wazuh team for their contributions to the open-source community.

---

<div align="center">

## Quick Start Commands

```bash
# Install Wazuh Manager
curl -sO https://packages.wazuh.com/4.5/wazuh-install.sh && sudo bash ./wazuh-install.sh -a

# Install Wazuh Agent
sudo curl -s https://raw.githubusercontent.com/rajeshmantri2711/wazuh-linux/main/Linux-Install.sh | bash
```

---

### Related Links

[![Documentation](https://img.shields.io/badge/Documentation-Wazuh-blue?style=flat-square)](https://documentation.wazuh.com/)
[![GitHub](https://img.shields.io/badge/GitHub-wazuh--linux-black?style=flat-square&logo=github)](https://github.com/i-am-paradoxx/wazuh-linux)
[![Website](https://img.shields.io/badge/Website-wazuh.com-green?style=flat-square)](https://wazuh.com/)

---

**Questions or Issues?** Open an issue in this repository
**Found this helpful?** Give it a star on GitHub!

</div>

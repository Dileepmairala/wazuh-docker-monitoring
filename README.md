# wazuh-docker-monitoring

# Wazuh Docker Container Monitoring Guide

This guide outlines the process of setting up Docker container monitoring with Wazuh. This will allow monitoring of container events and metrics as part of the NOC solution, complementing hardware failure alerts and service response time monitoring.

## Prerequisites

- Wazuh agent installed on Linux hosts
- Docker installed on the target system
- Python 3.x installed
- Docker Python SDK

## Installation Process

### 1. Verify Environment

First, check the Python version to ensure it's compatible:

```bash
python3 --version
# Python 3.9.18
```

### 2. Install Required Python Packages

Install pip if not available:

```bash
sudo yum install -y python3-pip
```

Install the Docker Python SDK:

```bash
pip3 install docker
```

### 3. Configure User Permissions

Create or configure a user for Docker operations:

```bash
# Create ossec user if not already present
useradd ossec

# Add ossec user to docker group to grant Docker access permissions
sudo usermod -aG docker ossec

# Restart Docker service to apply group changes
sudo systemctl restart docker

# Restart Wazuh agent
sudo systemctl restart wazuh-agent
```

### 4. Verify Docker SDK Access

Ensure the ossec user can access the Docker SDK:

```bash
# Check if the ossec user can use the Docker SDK
sudo -u ossec pip3 install docker

# Verify Docker module is accessible
sudo -u ossec python3 -c "import docker; print('Docker module is available')"
# Docker module is available
```

### 5. Check Docker Socket Permissions

Verify Docker socket permissions:

```bash
ls -la /var/run/docker.sock
# srw-rw----. 1 root docker 0 Mar 19 11:06 /var/run/docker.sock
```

### 6. Configure Wazuh for Docker Monitoring

Edit the Wazuh agent configuration file:

```bash
sudo vi /var/ossec/etc/ossec.conf
```

Add the Docker listener module configuration:

```xml
<wodle name="docker-listener">
  <disabled>no</disabled>
  <run_on_start>yes</run_on_start>
  <interval>10m</interval>
  <attempts>5</attempts>
</wodle>
```

### 7. Enable Debug Mode (Optional)

For troubleshooting, enable debug mode in the Wazuh agent configuration:

```bash
sed -i 's/<log_format>plain<\/log_format>/<log_format>plain<\/log_format>\n    <debug>2<\/debug>/' /var/ossec/etc/ossec.conf
```

### 8. Restart Wazuh Agent

Apply the new configuration:

```bash
sudo systemctl restart wazuh-agent
```

### 9. Verify Docker Monitoring

Test Docker events to ensure they're being captured:

```bash
# Run a test container
docker run -d --name test-container httpd

# Generate some Docker events
docker pause test-container
docker unpause test-container
docker stop test-container
docker rm test-container
docker rmi httpd

# Check logs for Docker events
tail -f /var/ossec/logs/ossec.log | grep -i docker
```

## Monitoring Capabilities

With this setup, Wazuh will now monitor:

### Container Events
- Container creation, start, stop, pause, unpause
- Container removal
- Image pull, removal
- Network creation/deletion
- Volume operations

### Container Metrics
- CPU utilization
- Memory usage
- Network I/O statistics
- Disk I/O

## Custom Wazuh Rules for Docker

Create custom rules in `/var/ossec/etc/rules/local_rules.xml`:

```xml
<group name="docker,">
  <!-- Container status change -->
  <rule id="100100" level="3">
    <decoded_as>json</decoded_as>
    <field name="integration">docker</field>
    <description>Docker: Container status changed</description>
  </rule>

  <!-- Container stopped -->
  <rule id="100101" level="5">
    <if_sid>100100</if_sid>
    <field name="status">stop</field>
    <description>Docker: Container $(container.name) stopped</description>
  </rule>
  
  <!-- Container killed -->
  <rule id="100102" level="10">
    <if_sid>100100</if_sid>
    <field name="status">kill</field>
    <description>Docker: Container $(container.name) killed</description>
  </rule>
  
  <!-- High CPU usage -->
  <rule id="100110" level="8">
    <decoded_as>json</decoded_as>
    <field name="integration">docker</field>
    <field name="cpu.usage">\.+</field>
    <description>Docker: Container $(container.name) high CPU usage: $(cpu.usage)%</description>
    <regex>^([8-9][0-9]|100)\.?[0-9]*$</regex>
  </rule>
</group>
```

## Integration with NOC Monitoring Requirements

This Docker monitoring setup addresses key NOC requirements:

### Network Performance Metrics
- Container network interface status
- Network throughput of containers
- Failure alerts of container networking

### System Health Metrics
- CPU Usage of containers
- Memory Utilization of containers
- Process & Service Uptime of containerized applications

### Application & Service Availability
- Container service status
- Application availability via container status

## Conclusion

The Docker listener wodle in Wazuh provides comprehensive monitoring of container events and metrics, making it a valuable component of the NOC solution. Combined with hardware failure alerts and service response time monitoring, this setup fulfills the customer's monitoring requirements for their network infrastructure.

# Flow 004: Security Implementation

## Introduction

Security is a critical component of hub-spoke network topology. This flow covers Network Security Groups (NSGs), Azure Firewall, network security best practices, and defense-in-depth strategies essential for securing hub-spoke architectures.

## Security Layers in Hub-Spoke

Hub-spoke architectures support multiple security layers:

```
Security Layers:
├── Network Security Groups (NSGs) - Subnet/NIC level filtering
├── Azure Firewall / NVA - Centralized inspection and control
├── Application Security Groups (ASGs) - Simplified rule management
├── DDoS Protection - Network protection
├── Azure Policy - Governance and compliance
└── Azure Private Link - Private service access
```

## Network Security Groups (NSGs)

### What are NSGs?

Network Security Groups provide stateful packet filtering:

- Applied at subnet or NIC level
- Allow or deny traffic based on rules
- Stateful: Return traffic automatically allowed
- First line of defense

### NSG Components

```
Network Security Group:
├── Inbound Security Rules
├── Outbound Security Rules
├── Default Security Rules (cannot be deleted)
└── Subnet/NIC Associations
```

### Security Rule Properties

Each rule contains:

1. **Priority**: 100-4096 (lower number = higher priority)
1. **Name**: Descriptive identifier
1. **Source**: IP address, CIDR, service tag, or ASG
1. **Source Port**: Port or range (usually *)
1. **Destination**: IP address, CIDR, service tag, or ASG
1. **Destination Port**: Specific port or range
1. **Protocol**: TCP, UDP, ICMP, ESP, AH, or Any
1. **Action**: Allow or Deny
1. **Direction**: Inbound or Outbound

### Default NSG Rules

**Inbound Default Rules** (cannot be deleted):

|Priority|Name                         |Source           |Destination   |Action|
|--------|-----------------------------|-----------------|--------------|------|
|65000   |AllowVnetInBound             |VirtualNetwork   |VirtualNetwork|Allow |
|65001   |AllowAzureLoadBalancerInBound|AzureLoadBalancer|*             |Allow |
|65500   |DenyAllInBound               |*                |*             |Deny  |

**Outbound Default Rules**:

|Priority|Name                 |Source        |Destination   |Action|
|--------|---------------------|--------------|--------------|------|
|65000   |AllowVnetOutBound    |VirtualNetwork|VirtualNetwork|Allow |
|65001   |AllowInternetOutBound|*             |Internet      |Allow |
|65500   |DenyAllOutBound      |*             |*             |Deny  |

**Important**: Default rules allow VNet-to-VNet traffic, including across peerings

### NSG Rule Processing

Azure evaluates rules in priority order:

1. Lower priority number evaluated first
1. First matching rule is applied
1. Processing stops after first match
1. If no rules match, default rules apply

**Example**:

```
Priority 100: Allow TCP 443 from Internet → Matches, allows HTTPS
Priority 200: Deny all from Internet → Never reached for HTTPS traffic
```

## NSG Design for Hub-Spoke

### Hub NSG Strategy

#### Gateway Subnet NSG

**Generally not recommended** - Can break VPN connectivity

- Gateway subnet typically has no NSG
- If required, carefully plan to allow gateway traffic

#### Azure Firewall Subnet NSG

**Not supported** - Cannot apply NSG to AzureFirewallSubnet

- Azure Firewall manages its own security
- Use firewall rules instead of NSG

#### Hub Management Subnet NSG

```
NSG: nsg-hub-management

Inbound Rules:
├── Priority 100: Allow RDP from Corporate IP ranges
├── Priority 110: Allow SSH from Corporate IP ranges
├── Priority 200: Allow monitoring from Azure Monitor
└── Priority 4096: Deny All

Outbound Rules:
├── Priority 100: Allow HTTPS to Azure services
├── Priority 110: Allow management traffic to spokes
└── Default: Allow VNet, Internet
```

### Spoke NSG Strategy

#### Web Tier NSG

```
NSG: nsg-spoke-web

Inbound Rules:
├── Priority 100: Allow HTTP/HTTPS from Internet
├── Priority 110: Allow HTTP/HTTPS from VirtualNetwork
├── Priority 200: Allow health probes from AzureLoadBalancer
└── Priority 4096: Deny All

Outbound Rules:
├── Priority 100: Allow HTTPS to app tier subnet
├── Priority 110: Allow HTTPS to Azure services
└── Default rules apply
```

#### Application Tier NSG

```
NSG: nsg-spoke-app

Inbound Rules:
├── Priority 100: Allow app traffic from web tier subnet
├── Priority 110: Allow management from hub management subnet
└── Priority 4096: Deny All

Outbound Rules:
├── Priority 100: Allow database traffic to data tier subnet
├── Priority 110: Allow HTTPS to Azure services
└── Default: Deny Internet outbound (override default)
```

#### Data Tier NSG

```
NSG: nsg-spoke-data

Inbound Rules:
├── Priority 100: Allow database traffic from app tier only
├── Priority 110: Allow management from hub management subnet
└── Priority 4096: Deny All

Outbound Rules:
├── Priority 100: Allow HTTPS to Azure backup/storage
├── Priority 200: Deny Internet (explicit)
└── Default rules overridden
```

## Service Tags

### What are Service Tags?

Service tags represent Azure service IP address prefixes:

- Maintained by Microsoft
- Automatically updated
- Simplify rule management
- Reduce configuration errors

### Common Service Tags

|Service Tag         |Represents                       |
|--------------------|---------------------------------|
|VirtualNetwork      |Current VNet and all peered VNets|
|AzureLoadBalancer   |Azure Load Balancer health probes|
|Internet            |Public internet addresses        |
|AzureCloud          |All Azure datacenter IPs         |
|Storage             |Azure Storage service IPs        |
|Sql                 |Azure SQL Database IPs           |
|AzureActiveDirectory|Azure AD service IPs             |
|AzureKeyVault       |Azure Key Vault IPs              |
|AzureMonitor        |Azure Monitor IPs                |

### Regional Service Tags

More specific tags for reduced exposure:

- `Storage.WestEurope`
- `Sql.EastUS`
- `AzureKeyVault.NorthEurope`

**Best Practice**: Use regional tags when possible

### Service Tag Usage Example

```
Instead of:
Source: 13.73.xxx.xxx/16 (Azure Storage IP range)

Use:
Source: Storage.WestEurope
```

**Benefits**:

- Automatic updates as Azure changes IPs
- No maintenance required
- More secure and reliable

## Application Security Groups (ASGs)

### What are ASGs?

ASGs group virtual machines logically:

- Assign VMs to ASG instead of using IPs
- Simplify NSG rules
- Reduce rule count
- Easier management at scale

### ASG Benefits

**Without ASG**:

```
Rule 1: Allow from 10.1.1.4 to 10.1.2.5 (web to app)
Rule 2: Allow from 10.1.1.5 to 10.1.2.5 (web to app)
Rule 3: Allow from 10.1.1.6 to 10.1.2.6 (web to app)
... (multiple rules for each VM)
```

**With ASG**:

```
ASG: asg-web-servers
ASG: asg-app-servers

Rule 1: Allow from asg-web-servers to asg-app-servers
(Single rule covers all web-to-app traffic)
```

### ASG Implementation Example

```
Application Security Groups:
├── asg-web-tier (all web VMs)
├── asg-app-tier (all app VMs)
└── asg-data-tier (all database VMs)

NSG Rule:
├── Source: asg-web-tier
├── Destination: asg-app-tier
├── Port: 443
└── Action: Allow
```

**Advantages**:

- Add/remove VMs without changing rules
- Clear, meaningful names
- Reduced complexity
- Better documentation

## Azure Firewall

### What is Azure Firewall?

Azure Firewall is a managed, cloud-based network security service:

- Fully stateful firewall
- Built-in high availability
- Unrestricted cloud scalability
- Application and network-level filtering
- Threat intelligence integration

### Azure Firewall SKUs

#### Standard SKU

- Network and application filtering
- Threat intelligence
- TLS inspection (with Premium)
- Basic DDoS protection
- Throughput: 30 Gbps

#### Premium SKU

- All Standard features
- TLS inspection
- IDPS (Intrusion Detection and Prevention)
- URL filtering
- Web categories
- Throughput: 100 Gbps

### Azure Firewall Components

```
Azure Firewall:
├── Firewall Policy
│   ├── Network Rules
│   ├── Application Rules
│   ├── NAT Rules
│   └── Threat Intelligence
├── Public IP Address(es)
├── Private IP Address
└── Diagnostic Settings
```

### Firewall Rule Types

#### 1. Network Rules

Filter traffic based on IP address, port, and protocol:

```
Network Rule Collection:
├── Priority: 100
├── Action: Allow
└── Rules:
    ├── spoke1-to-spoke2
    │   ├── Source: 10.1.0.0/16
    │   ├── Destination: 10.2.0.0/16
    │   ├── Protocol: Any
    │   └── Ports: *
    └── spoke-to-onprem
        ├── Source: 10.0.0.0/8 (all spokes)
        ├── Destination: 192.168.0.0/16 (on-premises)
        ├── Protocol: TCP
        └── Ports: 443, 445
```

**Use for**:

- Non-HTTP/HTTPS traffic
- Simple IP-based filtering
- Lower inspection overhead

#### 2. Application Rules

Filter HTTP/HTTPS traffic based on FQDNs:

```
Application Rule Collection:
├── Priority: 100
├── Action: Allow
└── Rules:
    ├── allow-microsoft
    │   ├── Source: 10.1.0.0/16, 10.2.0.0/16
    │   ├── Protocol: HTTPS
    │   ├── Target FQDNs:
    │   │   ├── *.microsoft.com
    │   │   ├── *.windows.net
    │   │   └── *.azure.com
    └── allow-ubuntu-updates
        ├── Source: *
        ├── Protocol: HTTP, HTTPS
        └── FQDN Tags: UbuntuUpdates
```

**Use for**:

- Web traffic filtering
- FQDN-based access control
- URL filtering (Premium)
- Web category filtering (Premium)

#### 3. NAT Rules (DNAT)

Translate and forward internet traffic to internal resources:

```
DNAT Rule Collection:
├── Priority: 100
└── Rules:
    ├── web-server-access
    │   ├── Source: Internet
    │   ├── Destination: {firewall-public-ip}
    │   ├── Port: 443
    │   ├── Translated Address: 10.1.1.4 (internal web server)
    │   └── Translated Port: 443
```

**Use for**:

- Publishing internal services
- Port forwarding
- Inbound internet access

### FQDN Tags

Pre-defined groups of FQDNs for common scenarios:

|FQDN Tag                        |Purpose                    |
|--------------------------------|---------------------------|
|WindowsUpdate                   |Windows Update services    |
|WindowsDiagnostics              |Windows diagnostics        |
|MicrosoftActiveProtectionService|Microsoft security services|
|AzureBackup                     |Azure Backup service       |
|AzureKubernetesService          |AKS requirements           |

**Example**:

```
Instead of listing all Windows Update FQDNs, use:
FQDN Tag: WindowsUpdate
```

### Threat Intelligence

Azure Firewall can alert or deny traffic to/from known malicious IPs:

**Modes**:

- **Off**: Disabled
- **Alert only**: Log suspicious traffic
- **Alert and deny**: Block and log

**Integration**: Microsoft’s global threat intelligence feed

### Firewall Policies

Azure Firewall Policy centralizes rule management:

- **Single policy**: Multiple firewalls
- **Inheritance**: Policy hierarchy
- **Rule collections**: Organized by priority
- **RBAC**: Role-based access control

**Best Practice**: Use Firewall Policy for all deployments

## Security Architecture Patterns

### Defense in Depth

Implement multiple security layers:

```
Layer 1: NSGs
├── Subnet-level filtering
├── Allow only required protocols
└── Deny by default approach

Layer 2: Azure Firewall
├── Centralized inspection
├── Application-aware filtering
└── Threat intelligence

Layer 3: Application Security
├── Web Application Firewall (WAF)
├── API authentication
└── Application-level controls

Layer 4: Data Security
├── Encryption at rest
├── Encryption in transit
└── Access controls
```

### Micro-Segmentation

Segment workloads for granular control:

```
Spoke VNet:
├── Web Tier Subnet
│   └── NSG: Allow Internet → Web only
├── App Tier Subnet
│   └── NSG: Allow Web → App only
└── Data Tier Subnet
    └── NSG: Allow App → Data only
```

### Zero Trust Network Access

Implement zero trust principles:

- **Verify explicitly**: Always authenticate and authorize
- **Least privilege**: Minimum required access
- **Assume breach**: Segment and monitor

**Implementation**:

- NSG rules deny by default
- Just-in-time VM access
- Conditional access policies
- Continuous monitoring

## Hub-Spoke Security Configuration

### Hub Firewall Configuration

```
Azure Firewall in Hub:
├── Network Rules
│   ├── spoke-to-spoke: Allow inter-spoke traffic
│   ├── spoke-to-onprem: Allow on-premises access
│   └── monitoring: Allow Azure Monitor
├── Application Rules
│   ├── azure-services: Allow *.azure.com, *.microsoft.com
│   ├── package-managers: Allow apt, yum, pip repos
│   └── business-apps: Allow specific application FQDNs
└── DNAT Rules
    └── Published services (if required)
```

### Spoke Security Configuration

```
Spoke 1 (Web Application):
├── Subnet: web-tier
│   └── NSG: Allow HTTPS from Internet, to app-tier
├── Subnet: app-tier
│   └── NSG: Allow from web-tier, to data-tier
├── Subnet: data-tier
│   └── NSG: Allow from app-tier only
└── Route Table: All traffic → Hub Firewall
```

## Monitoring and Logging

### NSG Flow Logs

Enable NSG flow logs for traffic analysis:

**Configuration**:

```
NSG Flow Logs:
├── Version: 2 (recommended)
├── Storage Account: Retention policy
├── Traffic Analytics: Enable
└── Log Analytics Workspace: Integration
```

**Data Captured**:

- Source and destination IPs
- Ports and protocols
- Traffic direction
- Allow/deny decision
- Bytes and packets

**Use Cases**:

- Security analysis
- Compliance auditing
- Traffic pattern analysis
- Troubleshooting

### Azure Firewall Logs

Enable comprehensive firewall logging:

**Diagnostic Settings**:

```
Categories:
├── AzureFirewallApplicationRule: Application rule hits
├── AzureFirewallNetworkRule: Network rule hits
├── AzureFirewallDnsProxy: DNS queries (if enabled)
└── Metrics: Performance and health
```

**Destinations**:

- Log Analytics workspace (recommended)
- Storage account (long-term retention)
- Event Hub (SIEM integration)

### Azure Monitor Workbooks

Use built-in workbooks for visualization:

- NSG Flow Logs workbook
- Azure Firewall workbook
- Traffic Analytics

### Alerts

Configure alerts for security events:

**NSG Alerts**:

- Unexpected deny rules triggered
- High volume of dropped packets
- Configuration changes

**Firewall Alerts**:

- Threat intelligence hits
- Unusual traffic patterns
- Rule hits from suspicious sources

## Best Practices

### NSG Best Practices

1. **Deny by Default**: Start with deny all, add specific allows
1. **Document Rules**: Clear naming and descriptions
1. **Use Service Tags**: Leverage Microsoft-maintained tags
1. **Implement ASGs**: Simplify management at scale
1. **Enable Flow Logs**: Monitor all NSG traffic
1. **Regular Review**: Audit rules quarterly
1. **Test Changes**: Validate before production deployment

### Azure Firewall Best Practices

1. **Use Firewall Policy**: Centralized management
1. **Implement Rule Collections**: Organize by function
1. **Leverage FQDN Tags**: Simplify common scenarios
1. **Enable Threat Intelligence**: Alert and deny mode
1. **Force Tunneling**: Consider for hybrid scenarios
1. **Monitor Logs**: Set up alerts and dashboards
1. **Plan Capacity**: Consider throughput requirements

### General Security Best Practices

1. **Least Privilege**: Grant minimum required access
1. **Defense in Depth**: Multiple security layers
1. **Automation**: Use IaC for consistency
1. **Monitoring**: Continuous visibility
1. **Incident Response**: Plan for security events
1. **Regular Updates**: Keep security policies current
1. **Compliance**: Align with regulatory requirements

## Common Security Misconfigurations

### Overly Permissive NSG Rules

**Problem**:

```
Source: * (any)
Destination: * (any)
Port: * (any)
Action: Allow
```

**Solution**: Be specific about source, destination, and ports

### Missing Spoke-to-Spoke Inspection

**Problem**: VNet peering allows direct spoke-to-spoke traffic without inspection

**Solution**:

- Configure UDRs to route through firewall
- Disable direct spoke-to-spoke peering if not needed

### Forgotten Outbound Internet Access

**Problem**: Default NSG allows all outbound to internet

**Solution**:

- Explicitly deny internet outbound in data tier
- Route through firewall for inspection
- Use service endpoints for Azure services

## Troubleshooting Security Issues

### Connectivity Problems

**Diagnostic Steps**:

1. Check effective NSG rules on both source and destination
1. Verify route tables point traffic correctly
1. Review Azure Firewall logs for denies
1. Use Network Watcher IP Flow Verify
1. Check application-level security

**Tools**:

- Network Watcher: Next Hop, IP Flow Verify
- NSG Flow Logs
- Azure Firewall logs
- Connection Monitor

### Performance Issues

**Symptoms**:

- High latency
- Packet drops
- Throughput limitations

**Investigation**:

1. Check Azure Firewall metrics (throughput, SNAT ports)
1. Review NSG flow logs for drops
1. Monitor VM network metrics
1. Consider firewall SKU upgrade if needed

## Infrastructure as Code Examples

### Bicep - NSG with Rules

```bicep
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-04-01' = {
  name: 'nsg-spoke1-web'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowHTTPS'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'Internet'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '443'
        }
      }
      {
        name: 'DenyAllInbound'
        properties: {
          priority: 4096
          direction: 'Inbound'
          access: 'Deny'
          protocol: '*'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '*'
        }
      }
    ]
  }
}
```

### Terraform - Azure Firewall

```hcl
resource "azurerm_firewall" "hub" {
  name                = "fw-hub-prod"
  location            = var.location
  resource_group_name = var.resource_group_name
  sku_name            = "AZFW_VNet"
  sku_tier            = "Premium"
  firewall_policy_id  = azurerm_firewall_policy.hub.id

  ip_configuration {
    name                 = "configuration"
    subnet_id            = azurerm_subnet.firewall.id
    public_ip_address_id = azurerm_public_ip.firewall.id
  }
}

resource "azurerm_firewall_policy" "hub" {
  name                = "fwpol-hub-prod"
  location            = var.location
  resource_group_name = var.resource_group_name
  sku                 = "Premium"

  threat_intelligence_mode = "Alert"
}
```

## Exam Relevance

### AZ-700 Objectives

- Configure NSGs and ASGs
- Implement Azure Firewall
- Configure firewall rules and policies
- Monitor network security
- Troubleshoot security issues

### AZ-305 Objectives

- Design network security solutions
- Design for defense in depth
- Recommend security architecture
- Design for compliance requirements

## Summary

Effective security in hub-spoke architectures requires:

- **NSGs** for subnet and NIC-level filtering
- **Azure Firewall** for centralized inspection
- **Defense in depth** with multiple security layers
- **Monitoring** and logging for visibility
- **Best practices** consistently applied

Key principles:

- Deny by default, allow specific traffic
- Layer security controls
- Monitor and log all traffic
- Regular review and updates

## Next Steps

- **Flow 005**: Hybrid Connectivity (VPN and ExpressRoute)
- **Task 004**: Deploy Azure Firewall
- **Task 006**: Implement NSGs
- **Lab 02**: Add Azure Firewall to Hub-Spoke

-----

*Part of the Learning Azure Networking Topology: Hub-Spoke series*

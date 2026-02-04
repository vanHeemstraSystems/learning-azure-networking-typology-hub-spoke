# Flow 002: Virtual Network Peering

## Introduction

Virtual Network (VNet) peering is the foundation of hub-spoke topology in Azure. This flow provides a comprehensive understanding of VNet peering concepts, configuration, limitations, and best practices essential for implementing hub-spoke architectures.

## What is VNet Peering?

VNet peering connects two Azure virtual networks, allowing resources in both VNets to communicate with each other using private IP addresses. Traffic between peered VNets:

- Uses Microsoft’s private backbone network
- Does not traverse the public internet
- Remains within Azure’s network infrastructure
- Provides low latency and high bandwidth

## Types of VNet Peering

### 1. Regional VNet Peering

Connects VNets in the same Azure region:

- Lowest latency option
- No bandwidth restrictions beyond VM limits
- Most common for single-region deployments
- Simplest configuration

**Use Cases**:

- Spokes connecting to hub within same region
- Multi-tier applications in same region
- Development/test environment isolation

### 2. Global VNet Peering

Connects VNets across different Azure regions:

- Enables multi-region architectures
- Slightly higher latency than regional peering
- Same security and private connectivity
- Supports disaster recovery scenarios

**Use Cases**:

- Multi-region hub-spoke architectures
- Global application deployments
- Cross-region disaster recovery
- Geographic distribution of workloads

## Key Characteristics

### Private Connectivity

- **No public IP required**: Communication uses private IP addresses only
- **Microsoft backbone**: Traffic never touches public internet
- **Encrypted in transit**: Protected by Microsoft’s network security
- **No gateway required**: Direct VNet-to-VNet connectivity (except for transit scenarios)

### Performance

- **Low latency**: Near-identical to intra-VNet communication
- **High bandwidth**: Full VM network bandwidth available
- **No bottleneck**: No central device limiting throughput
- **Scalable**: Supports large-scale deployments

### Billing

- **Ingress/Egress charges**:
  - Regional peering: Minimal data transfer costs
  - Global peering: Higher data transfer costs between regions
- **No peering connection charge**: Only pay for data transfer
- **Predictable costs**: Based on actual data transferred

## Peering Configuration

### Prerequisites

1. **Non-overlapping IP address spaces**
- Each VNet must have unique CIDR blocks
- Cannot peer VNets with overlapping ranges
- Plan address spaces carefully before deployment
1. **Proper permissions**
- Network Contributor role or higher
- Access to both VNets being peered
1. **Same Azure Active Directory tenant** (for standard peering)
- Cross-tenant peering requires additional configuration
- Typically used within same organization

### Peering Properties

#### Allow Virtual Network Access

- **Default**: Enabled
- **Purpose**: Permits traffic between peered VNets
- **When to disable**: Temporarily block traffic without deleting peering

#### Allow Forwarded Traffic

- **Default**: Disabled
- **Purpose**: Allows traffic forwarded by network virtual appliance (NVA) or Azure Firewall
- **Hub-spoke requirement**: Must be enabled on spoke peerings
- **Use case**: Essential for spoke-to-spoke communication through hub

#### Allow Gateway Transit

- **Default**: Disabled
- **Purpose**: Allows peered VNet to use this VNet’s gateway
- **Hub configuration**: Enable on hub VNet
- **Requirement**: VNet must have VPN or ExpressRoute gateway
- **Benefit**: Spokes can access on-premises through hub gateway

#### Use Remote Gateway

- **Default**: Disabled
- **Purpose**: Use gateway in remote peered VNet
- **Spoke configuration**: Enable on spoke VNets
- **Mutual exclusivity**: Cannot enable if VNet has its own gateway
- **Dependency**: Remote VNet must have “Allow Gateway Transit” enabled

## Hub-Spoke Peering Configuration

### Hub VNet Configuration

```
Hub VNet Settings:
├── Allow virtual network access: Enabled
├── Allow forwarded traffic: Enabled
├── Allow gateway transit: Enabled (if gateway present)
└── Use remote gateways: Disabled
```

**Rationale**:

- Accept traffic from spokes
- Allow forwarded traffic from Azure Firewall/NVA
- Share VPN/ExpressRoute gateway with spokes

### Spoke VNet Configuration

```
Spoke VNet Settings:
├── Allow virtual network access: Enabled
├── Allow forwarded traffic: Enabled
├── Allow gateway transit: Disabled
└── Use remote gateways: Enabled (if hub has gateway)
```

**Rationale**:

- Communicate with hub
- Accept traffic forwarded through hub (for spoke-to-spoke)
- Use hub’s gateway for on-premises connectivity

## Peering States

### Connected

- Peering successfully established
- Traffic flowing between VNets
- Normal operational state

### Initiated

- Peering created but not yet accepted
- Occurs during cross-subscription peering setup
- Waiting for remote peer configuration

### Disconnected

- Peering configuration issue
- Possibly due to one side being deleted
- No traffic flow

### Updating

- Peering properties being modified
- Temporary state during configuration changes

## Limitations and Constraints

### Address Space Limitations

- **No overlapping CIDR blocks**: Absolutely required
- **Cannot change after peering**: Address spaces are locked once peered
- **Planning critical**: Design IP scheme before implementation

### Peering Limits

|Resource                 |Limit|Notes                                       |
|-------------------------|-----|--------------------------------------------|
|Peerings per VNet        |500  |Can be increased to 1000 via support request|
|Global peerings per VNet |100  |Subset of total peering limit               |
|VNets peered to same VNet|500  |Default limit                               |

### Non-Transitivity

**Critical Concept**: VNet peering is non-transitive

Example:

- VNet A peers with VNet B
- VNet B peers with VNet C
- VNet A **cannot** communicate with VNet C through VNet B

**Hub-Spoke Implication**:

- Spoke A cannot directly reach Spoke B
- Traffic must be routed through hub
- Requires User-Defined Routes (UDRs) and hub routing device

### Service Chaining Limitations

- Cannot chain multiple peerings for transit
- Hub must have routing capability (Azure Firewall, NVA)
- UDRs required for spoke-to-spoke communication

## Advanced Peering Scenarios

### Cross-Subscription Peering

**Setup Process**:

1. Create peering from Subscription A to VNet in Subscription B
1. Create reciprocal peering from Subscription B to VNet in Subscription A
1. Both subscriptions must be in same Azure AD tenant (or use cross-tenant)
1. Requires appropriate permissions in both subscriptions

**Use Cases**:

- Different business units with separate subscriptions
- Hub owned by IT, spokes owned by application teams
- Cost separation and chargeback

### Cross-Tenant Peering

**Additional Requirements**:

- Azure Active Directory B2B invitation
- Guest access granted between tenants
- More complex permission management

**Use Cases**:

- Partner organization connectivity
- Merger and acquisition scenarios
- Managed service provider scenarios

### Peering with Service Endpoints

- Service endpoints can coexist with VNet peering
- Traffic to Azure services can bypass peering
- Plan service endpoint usage carefully
- May affect routing and security policies

### Peering with Private Endpoints

- Private endpoints work within peered VNets
- Private DNS zones should be in hub or centralized
- Consider DNS resolution across peerings
- Plan private endpoint subnet allocation

## Security Considerations

### Network Security Groups (NSGs)

- NSGs apply to traffic within peered VNets
- Configure NSGs to control cross-VNet traffic
- Can allow/deny specific protocols and ports
- Apply at subnet or NIC level

**Best Practice**: Use NSGs for defense in depth even with peering

### Azure Firewall Integration

- Deploy Azure Firewall in hub VNet
- Force spoke traffic through firewall with UDRs
- Centralized inspection and logging
- Application and network rule enforcement

### Service Tags

- Use VirtualNetwork service tag for peered VNets
- Simplifies NSG rule management
- Automatically includes peered VNet ranges

## Monitoring and Troubleshooting

### Key Metrics to Monitor

1. **Peering State**: Ensure “Connected” status
1. **Bytes In/Out**: Monitor data transfer
1. **Packets Dropped**: Identify connectivity issues
1. **Peering Health**: Overall peering status

### Azure Monitor Integration

```
Metrics Available:
├── VNet Peering Bytes In
├── VNet Peering Bytes Out
├── VNet Peering Packets In
├── VNet Peering Packets Out
└── VNet Peering Packets Dropped
```

### Troubleshooting Tools

#### 1. Connection Monitor

- Proactive monitoring of peering connectivity
- Latency and packet loss tracking
- Alerts on connectivity issues

#### 2. Network Watcher

- **Next Hop**: Verify routing through peering
- **IP Flow Verify**: Check NSG rules impact
- **Connection Troubleshoot**: Diagnose connectivity problems
- **Topology View**: Visualize peering relationships

#### 3. NSG Flow Logs

- Detailed traffic logs across peering
- Identify blocked or allowed traffic
- Security and compliance auditing

### Common Issues and Solutions

#### Issue: Peering shows “Connected” but no traffic flows

**Possible Causes**:

- NSG blocking traffic
- Route table misconfiguration
- Firewall rules blocking traffic

**Resolution**:

1. Check NSG rules on both VNets
1. Verify route tables
1. Use IP Flow Verify to test
1. Check Azure Firewall/NVA rules

#### Issue: Cannot create peering - “Address spaces overlap”

**Resolution**:

1. Plan new, non-overlapping address space
1. Create new VNet with different range
1. Migrate resources to new VNet
1. Cannot be fixed without changing IP ranges

#### Issue: Gateway transit not working

**Possible Causes**:

- “Allow Gateway Transit” not enabled on hub
- “Use Remote Gateway” not enabled on spoke
- Gateway not fully deployed
- Spoke has its own gateway (mutually exclusive)

**Resolution**:

1. Verify gateway transit settings
1. Check gateway deployment status
1. Remove spoke gateway if present
1. Update peering configuration

## Cost Optimization

### Regional Peering Costs

- **Same region**: ~$0.01/GB (both ingress and egress)
- **Minimal cost**: For most workloads
- **Best practice**: Keep frequently communicating workloads in same region

### Global Peering Costs

- **Cross-region**: ~$0.035-$0.08/GB depending on regions
- **Higher cost**: Especially for high-traffic scenarios
- **Optimization**: Minimize unnecessary cross-region traffic

### Cost Reduction Strategies

1. **Optimize data transfer**: Reduce unnecessary cross-VNet traffic
1. **Regional consolidation**: Keep related workloads in same region
1. **Caching**: Implement caching to reduce repeated data transfer
1. **Compression**: Compress data before transfer where possible
1. **Monitoring**: Track data transfer costs and optimize high-cost paths

## Infrastructure as Code Examples

### Azure CLI

```bash
# Create regional peering from hub to spoke
az network vnet peering create \
  --name hub-to-spoke1 \
  --resource-group rg-hub \
  --vnet-name vnet-hub \
  --remote-vnet /subscriptions/{sub-id}/resourceGroups/rg-spoke1/providers/Microsoft.Network/virtualNetworks/vnet-spoke1 \
  --allow-vnet-access \
  --allow-forwarded-traffic \
  --allow-gateway-transit

# Create reciprocal peering from spoke to hub
az network vnet peering create \
  --name spoke1-to-hub \
  --resource-group rg-spoke1 \
  --vnet-name vnet-spoke1 \
  --remote-vnet /subscriptions/{sub-id}/resourceGroups/rg-hub/providers/Microsoft.Network/virtualNetworks/vnet-hub \
  --allow-vnet-access \
  --allow-forwarded-traffic \
  --use-remote-gateways
```

### PowerShell

```powershell
# Hub to Spoke peering
Add-AzVirtualNetworkPeering `
  -Name "hub-to-spoke1" `
  -VirtualNetwork $hubVnet `
  -RemoteVirtualNetworkId $spoke1Vnet.Id `
  -AllowForwardedTraffic `
  -AllowGatewayTransit

# Spoke to Hub peering
Add-AzVirtualNetworkPeering `
  -Name "spoke1-to-hub" `
  -VirtualNetwork $spoke1Vnet `
  -RemoteVirtualNetworkId $hubVnet.Id `
  -AllowForwardedTraffic `
  -UseRemoteGateways
```

## Best Practices

### 1. Plan IP Address Spaces Carefully

- Document all address ranges before deployment
- Use RFC 1918 private ranges consistently
- Leave room for growth in each VNet
- Avoid common ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 default splits)

### 2. Implement Naming Conventions

Example format: `peer-{source-vnet}-to-{destination-vnet}`

- `peer-hub-prod-to-spoke-web-prod`
- `peer-spoke-web-prod-to-hub-prod`

### 3. Document Peering Relationships

Maintain documentation including:

- Peering name and direction
- VNets involved
- Configuration settings
- Purpose and business justification
- Created date and owner

### 4. Automate Peering Creation

- Use Bicep or Terraform templates
- Version control configurations
- Implement validation tests
- Automate reciprocal peering setup

### 5. Monitor Peering Health

- Set up alerts for disconnected peerings
- Monitor data transfer costs
- Track peering capacity against limits
- Regular health checks

### 6. Secure Peering Traffic

- Always use NSGs
- Implement Azure Firewall in hub
- Enable diagnostic logging
- Regular security audits

## Exam Relevance

### AZ-700 Objectives

- Configure VNet peering (regional and global)
- Implement hub-spoke network topology
- Configure gateway transit
- Troubleshoot VNet connectivity

### AZ-305 Objectives

- Design VNet peering solutions
- Design for network connectivity
- Recommend network topology patterns
- Design for security and governance

## Hands-On Practice

### Exercise 1: Basic Peering

1. Create two VNets with non-overlapping address spaces
1. Configure bidirectional peering
1. Deploy VMs in each VNet
1. Test connectivity with ping
1. Monitor traffic with Network Watcher

### Exercise 2: Hub-Spoke Peering

1. Create hub VNet with Azure Firewall
1. Create two spoke VNets
1. Configure peering with appropriate settings
1. Implement UDRs for spoke-to-spoke traffic
1. Test and validate traffic flow

### Exercise 3: Cross-Region Peering

1. Create VNets in different regions
1. Configure global VNet peering
1. Measure latency between regions
1. Monitor data transfer costs
1. Compare with regional peering

## Summary

VNet peering provides:

- **Foundation** for hub-spoke topology
- **Private, secure** connectivity between VNets
- **High performance** with low latency
- **Scalable** architecture for enterprise deployments

Key considerations:

- Non-overlapping address spaces required
- Non-transitive nature requires hub routing
- Configuration differs for hub vs spoke
- Security must be implemented separately

Understanding peering is essential before implementing routing and security controls in subsequent flows.

## Next Steps

- **Flow 003**: Routing Configuration (UDRs and traffic control)
- **Flow 004**: Security Implementation (NSGs and Azure Firewall)
- **Task 003**: Configure VNet Peering (hands-on implementation)

-----

*Part of the Learning Azure Networking Topology: Hub-Spoke series*

# Flow 005: Hybrid Connectivity

## Introduction

Hybrid connectivity enables Azure hub-spoke networks to communicate with on-premises datacenters. This flow covers VPN Gateway, ExpressRoute, and hybrid networking patterns essential for enterprise Azure deployments.

## Hybrid Connectivity Overview

Hybrid connectivity connects Azure to on-premises infrastructure:

- Extends corporate network to cloud
- Enables workload migration
- Supports hybrid applications
- Provides disaster recovery options

**Two primary options**:

1. **VPN Gateway**: Encrypted over internet
1. **ExpressRoute**: Private dedicated connection

## VPN Gateway

### What is VPN Gateway?

VPN Gateway provides encrypted cross-premises connectivity:

- IPsec/IKE VPN tunnel over internet
- Encrypted traffic
- Multiple connection types supported
- Managed Azure service

### VPN Gateway SKUs

|SKU     |Tunnels|Throughput|BGP Support|Zone Redundant|
|--------|-------|----------|-----------|--------------|
|VpnGw1  |30     |650 Mbps  |Yes        |No            |
|VpnGw2  |30     |1 Gbps    |Yes        |No            |
|VpnGw3  |30     |1.25 Gbps |Yes        |No            |
|VpnGw4  |100    |5 Gbps    |Yes        |No            |
|VpnGw5  |100    |10 Gbps   |Yes        |No            |
|VpnGw1AZ|30     |650 Mbps  |Yes        |Yes           |
|VpnGw2AZ|30     |1 Gbps    |Yes        |Yes           |
|VpnGw3AZ|30     |1.25 Gbps |Yes        |Yes           |

**Recommendation**: Use AZ SKUs for production (availability zone support)

### VPN Gateway Types

#### 1. Site-to-Site (S2S)

Connects on-premises network to Azure:

- IPsec/IKE VPN tunnel
- Requires VPN device on-premises
- Always-on connectivity
- Hub-spoke standard configuration

**Hub-Spoke Use**: Primary method for on-premises connectivity

#### 2. Point-to-Site (P2S)

Connects individual clients to Azure:

- Individual computer VPN connection
- No VPN device required
- On-demand connectivity
- Remote worker access

**Hub-Spoke Use**: Remote administration access

#### 3. VNet-to-VNet

Connects two Azure VNets via VPN:

- Alternative to VNet peering
- Encrypted tunnel
- Cross-region connectivity
- Different subscription support

**Hub-Spoke Use**: Multi-region hub connectivity (though peering preferred)

### VPN Gateway Components

```
VPN Gateway Deployment:
├── Gateway Subnet
│   ├── Name: GatewaySubnet (required name)
│   ├── Size: Minimum /27, recommend /26
│   └── No NSG (not recommended)
├── VPN Gateway
│   ├── SKU: VpnGw1AZ or higher
│   ├── Type: VPN
│   ├── VPN Type: Route-based
│   └── Public IP Address
├── Local Network Gateway
│   ├── On-premises public IP
│   └── On-premises address space
└── Connection
    ├── Connection Type: IPsec
    ├── Shared Key
    └── BGP Settings (optional)
```

### VPN Configuration in Hub-Spoke

**Hub VNet Configuration**:

```
Hub VNet: 10.0.0.0/16
├── GatewaySubnet: 10.0.255.0/27
├── AzureFirewallSubnet: 10.0.1.0/26
└── Management Subnet: 10.0.2.0/24

VPN Gateway:
├── Name: vpngw-hub-prod
├── SKU: VpnGw2AZ
├── VPN Type: Route-based
├── BGP: Enabled
└── ASN: 65515
```

**Local Network Gateway**:

```
On-Premises Configuration:
├── Public IP: 203.0.113.10
├── Address Space: 192.168.0.0/16
└── BGP Peer IP: 192.168.255.1 (if using BGP)
```

### BGP Configuration

Border Gateway Protocol enables dynamic routing:

**Benefits**:

- Automatic route propagation
- Redundancy and failover
- Multi-site connectivity
- Simplified route management

**BGP Configuration**:

```
Azure VPN Gateway:
├── ASN: 65515 (Azure default)
├── BGP Peer IP: 10.0.255.254 (auto-assigned)
└── Routes: Advertises VNet and peered ranges

On-Premises Router:
├── ASN: 65001 (example)
├── BGP Peer IP: 192.168.255.1
└── Routes: Advertises on-premises ranges
```

**Hub-Spoke BGP**:

- Gateway advertises hub and all peered spoke ranges
- Spokes automatically learn on-premises routes
- Enable BGP route propagation on spoke route tables

## ExpressRoute

### What is ExpressRoute?

ExpressRoute provides private, dedicated connectivity:

- Does not traverse public internet
- Predictable performance
- Higher reliability
- Lower latency than VPN

### ExpressRoute Connection Models

#### 1. CloudExchange Co-location

Located in datacenter with exchange:

- Direct connection to Microsoft network
- Physical cross-connection
- Lowest latency

#### 2. Point-to-Point Ethernet

Dedicated connection:

- Private circuit to Microsoft
- Service provider managed
- Guaranteed bandwidth

#### 3. Any-to-Any (IPVPN)

Integrated with existing MPLS:

- ExpressRoute as part of WAN
- Service provider network
- Simplified management

### ExpressRoute SKUs

|SKU     |Bandwidth        |Redundancy|Cost Model          |
|--------|-----------------|----------|--------------------|
|Local   |1-10 Gbps        |Redundant |Unlimited data      |
|Standard|50 Mbps - 10 Gbps|Redundant |Metered or Unlimited|
|Premium |50 Mbps - 10 Gbps|Redundant |Metered or Unlimited|

**Premium Features**:

- Global VNet connectivity
- Increased route limits
- Microsoft 365 connectivity
- Multi-region support

### ExpressRoute Components

```
ExpressRoute Deployment:
├── ExpressRoute Circuit
│   ├── Service Provider
│   ├── Bandwidth
│   ├── SKU (Standard/Premium)
│   └── Service Key
├── Gateway Subnet (same as VPN)
│   └── Minimum /27, recommend /26
├── ExpressRoute Gateway
│   ├── SKU (Standard, HighPerf, ErGw1AZ, etc.)
│   ├── Public IP Address
│   └── Virtual Network
└── Connection
    ├── Links gateway to circuit
    └── Authorization key (if cross-subscription)
```

### ExpressRoute Gateway SKUs

|SKU             |Throughput|VNet Connections|Zone Redundant|
|----------------|----------|----------------|--------------|
|Standard        |1 Gbps    |10              |No            |
|HighPerformance |2 Gbps    |10              |No            |
|UltraPerformance|10 Gbps   |10              |No            |
|ErGw1AZ         |1 Gbps    |10              |Yes           |
|ErGw2AZ         |2 Gbps    |10              |Yes           |
|ErGw3AZ         |10 Gbps   |10              |Yes           |

**Recommendation**: Use AZ SKUs for production

### ExpressRoute Peering Types

#### Microsoft Peering

Access to Microsoft services:

- Azure public services
- Microsoft 365 (with Premium)
- Dynamics 365
- Public IP addresses required

#### Private Peering

Access to Azure VNets:

- Private IP connectivity
- Hub-spoke standard configuration
- Extends on-premises to Azure
- Optimal for IaaS workloads

### ExpressRoute in Hub-Spoke

**Hub Configuration**:

```
Hub VNet: 10.0.0.0/16
├── GatewaySubnet: 10.0.255.0/27
├── ExpressRoute Gateway
│   ├── SKU: ErGw2AZ
│   ├── Connected to Circuit
│   └── BGP enabled (automatic)
└── Advertises: All hub and spoke ranges

Spokes:
└── Route Tables: BGP propagation enabled
    └── Automatically learn on-premises routes
```

**Routing Behavior**:

- ExpressRoute gateway advertises all peered VNet ranges
- On-premises automatically learns Azure routes
- Spokes automatically learn on-premises routes via BGP
- No manual route configuration needed

## Hybrid Connectivity Patterns

### Pattern 1: VPN-Only Connectivity

**Use Case**:

- Small to medium deployments
- Cost-sensitive scenarios
- Temporary connectivity
- Disaster recovery backup

**Configuration**:

```
Hub VNet:
└── VPN Gateway
    ├── S2S connection to on-premises
    └── Spokes use remote gateway
```

**Considerations**:

- Internet bandwidth dependent
- Encryption overhead
- Shared throughput among spokes
- Latency variable

### Pattern 2: ExpressRoute-Only

**Use Case**:

- Large enterprises
- High bandwidth requirements
- Latency-sensitive applications
- Production workloads

**Configuration**:

```
Hub VNet:
└── ExpressRoute Gateway
    ├── Connected to ExpressRoute circuit
    └── Spokes use remote gateway
```

**Considerations**:

- Higher cost than VPN
- Requires service provider
- Setup time longer
- Most reliable option

### Pattern 3: ExpressRoute with VPN Failover

**Use Case**:

- Mission-critical connectivity
- High availability requirement
- Disaster recovery
- Maximum uptime

**Configuration**:

```
Hub VNet:
├── ExpressRoute Gateway (primary)
└── VPN Gateway (backup)
    └── Both can coexist in same GatewaySubnet
```

**Failover Behavior**:

- ExpressRoute preferred (better BGP metrics)
- Automatic failover to VPN if ExpressRoute fails
- Return to ExpressRoute when restored
- Requires BGP configuration

**Important**: Use different public IPs for each gateway

### Pattern 4: Multi-Region Hub-Spoke

**Use Case**:

- Global deployments
- Regional redundancy
- Geo-distributed applications
- Disaster recovery across regions

**Configuration**:

```
Region 1:
└── Hub VNet 1
    └── ExpressRoute Gateway
        └── Connected to Circuit 1

Region 2:
└── Hub VNet 2
    └── ExpressRoute Gateway
        └── Connected to Circuit 2

Hubs connected via:
├── VNet Peering (recommended), or
└── VPN Gateway (encrypted cross-region)
```

## Gateway Transit

### What is Gateway Transit?

Gateway transit allows spokes to use hub’s gateway:

- Spokes don’t need their own gateway
- Cost optimization
- Simplified management
- Centralized connectivity

### Configuration Requirements

**Hub VNet Peering Settings**:

```
Allow Gateway Transit: Enabled
└── Requires gateway to exist in hub
```

**Spoke VNet Peering Settings**:

```
Use Remote Gateway: Enabled
├── Cannot have own gateway
└── Hub must allow gateway transit
```

**Route Propagation**:

```
Spoke Route Tables:
└── BGP Route Propagation: Enabled
    └── Automatically learns on-premises routes
```

### Gateway Transit Troubleshooting

**Common Issues**:

1. **Spokes cannot reach on-premises**
- Verify “Allow Gateway Transit” on hub
- Verify “Use Remote Gateway” on spoke
- Check BGP route propagation enabled
- Confirm no gateway in spoke VNet
1. **Routes not propagating**
- Enable BGP route propagation on route tables
- Verify gateway is advertising routes
- Check effective routes on spoke VMs
1. **Asymmetric routing**
- Ensure on-premises advertises all Azure ranges
- Check route tables don’t override gateway routes
- Verify firewall routing if present

## Routing with Hybrid Connectivity

### On-Premises to Spoke

**Traffic Flow**:

```
On-Premises → VPN/ExpressRoute Gateway → Hub → Spoke
```

**Spoke Route Table**:

```
BGP Route Propagation: Enabled
└── Learns on-premises routes automatically
```

**No UDRs needed** for on-premises connectivity when using BGP

### Spoke to On-Premises via Firewall

**Scenario**: Inspect spoke-to-on-premises traffic

**Configuration**:

```
Spoke Route Table:
├── BGP Route Propagation: Disabled
└── Route: on-premises-via-firewall
    ├── Address Prefix: 192.168.0.0/16 (on-premises)
    ├── Next Hop: Virtual Appliance
    └── Next Hop Address: 10.0.1.4 (firewall)

Firewall Subnet Route Table:
└── BGP Route Propagation: Enabled
    └── Learns gateway routes for forwarding
```

**Traffic Flow**:

```
Spoke VM → Firewall (via UDR) → Gateway → On-Premises
```

### Internet Breakout Options

#### Option 1: Azure Egress

Internet traffic exits through Azure:

```
On-Premises → ExpressRoute → Spoke → Firewall → Internet
```

**Configuration**: Default behavior with forced tunneling disabled

#### Option 2: On-Premises Egress (Forced Tunneling)

Internet traffic returns to on-premises:

```
Spoke → Firewall → ExpressRoute → On-Premises → Internet
```

**Firewall Route Table**:

```
Route: force-tunnel-internet
├── Address Prefix: 0.0.0.0/0
├── Next Hop: Virtual Network Gateway
└── Effect: Internet via on-premises
```

## High Availability and Redundancy

### Gateway Redundancy

#### Active-Active VPN

```
VPN Gateway Configuration:
├── Enable Active-Active: Yes
├── Public IP 1: Primary tunnel
├── Public IP 2: Secondary tunnel
└── Both tunnels active simultaneously
```

**Benefits**:

- Increased throughput (both tunnels used)
- Faster failover
- Better availability

#### Zone-Redundant Gateways

```
Use AZ SKUs:
├── VpnGwXAZ (VPN)
└── ErGwXAZ (ExpressRoute)

Deployment:
└── Automatically spans availability zones
```

**Benefits**:

- Protection from zone failures
- Higher SLA (99.99%)
- No additional configuration needed

### ExpressRoute Resiliency

#### Dual Circuits

```
Primary Circuit:
└── Location: Amsterdam

Secondary Circuit:
└── Location: London

Configuration:
└── Both connected to same gateway
    └── BGP handles failover automatically
```

#### ExpressRoute Direct

Dedicated 10 Gbps or 100 Gbps connection:

- Direct connection to Microsoft global network
- Multiple circuits on single connection
- Maximum bandwidth
- Enterprise-grade connectivity

## Monitoring and Diagnostics

### Gateway Metrics

**VPN Gateway Metrics**:

```
Key Metrics:
├── Tunnel Ingress/Egress Bytes
├── Tunnel Packet Drop Count
├── P2S Connection Count
├── Gateway S2S Bandwidth
└── BGP Routes Advertised/Learned
```

**ExpressRoute Metrics**:

```
Key Metrics:
├── BitsInPerSecond / BitsOutPerSecond
├── Gateway CPU / Memory
├── Packets per Second
├── Routes Advertised to Peer
└── BGP Availability
```

### Connection Monitor

Set up connection monitoring:

```
Connection Monitor:
├── Source: On-premises endpoint
├── Destination: Spoke VM
├── Protocol: ICMP / TCP
└── Alerts: Latency or connectivity issues
```

### Diagnostic Logs

Enable gateway diagnostics:

```
Diagnostic Settings:
├── GatewayDiagnosticLog
├── TunnelDiagnosticLog
├── RouteDiagnosticLog
├── IKEDiagnosticLog
└── AllMetrics
```

**Send to**:

- Log Analytics (recommended)
- Storage account
- Event Hub

## Cost Optimization

### VPN Gateway Costs

**Cost Components**:

- Gateway hourly rate (based on SKU)
- Egress data transfer
- No ingress charges

**Optimization**:

- Right-size gateway SKU
- Use Basic SKU for dev/test
- Consider reserved capacity for production

### ExpressRoute Costs

**Cost Components**:

- Circuit monthly fee
- Data transfer (metered plans)
- Service provider fees

**Optimization**:

- Unlimited data plan if high traffic
- Local SKU for same-region traffic
- Right-size circuit bandwidth

### Gateway Transit Savings

**Savings with Hub-Spoke**:

```
Without Hub-Spoke:
├── 5 VNets × VPN Gateway cost = 5× cost
└── Each VNet has own gateway

With Hub-Spoke:
├── 1 Hub VNet with gateway
├── 5 Spoke VNets use remote gateway
└── Single gateway cost
```

**Savings**: ~80% on gateway costs

## Security Considerations

### VPN Security

1. **Strong Encryption**: Use AES256
1. **Certificate Authentication**: P2S connections
1. **MFA Integration**: Azure AD authentication for P2S
1. **Rotate Shared Keys**: Regular key rotation
1. **Monitor Logs**: Track connection attempts

### ExpressRoute Security

1. **Private Connectivity**: Traffic never touches internet
1. **MACsec Encryption**: Optional layer 2 encryption
1. **Network Security**: Still use NSGs and firewalls
1. **Access Control**: Limit ExpressRoute access
1. **Monitoring**: Track circuit usage and health

### Best Practices

1. **Separate Gateway Subnet**: Dedicated /27 or larger
1. **No NSG on Gateway Subnet**: Can break connectivity
1. **Enable Diagnostics**: Full logging
1. **BGP Authentication**: MD5 authentication if supported
1. **Backup Connectivity**: VPN as backup for ExpressRoute

## Infrastructure as Code Examples

### Bicep - VPN Gateway

```bicep
resource gatewaySubnet 'Microsoft.Network/virtualNetworks/subnets@2023-04-01' = {
  name: 'GatewaySubnet'
  parent: hubVnet
  properties: {
    addressPrefix: '10.0.255.0/27'
  }
}

resource publicIP 'Microsoft.Network/publicIPAddresses@2023-04-01' = {
  name: 'pip-vpngw-hub'
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    publicIPAllocationMethod: 'Static'
  }
  zones: ['1', '2', '3']
}

resource vpnGateway 'Microsoft.Network/virtualNetworkGateways@2023-04-01' = {
  name: 'vpngw-hub-prod'
  location: location
  properties: {
    gatewayType: 'Vpn'
    vpnType: 'RouteBased'
    sku: {
      name: 'VpnGw2AZ'
      tier: 'VpnGw2AZ'
    }
    enableBgp: true
    bgpSettings: {
      asn: 65515
    }
    ipConfigurations: [
      {
        name: 'default'
        properties: {
          subnet: {
            id: gatewaySubnet.id
          }
          publicIPAddress: {
            id: publicIP.id
          }
        }
      }
    ]
  }
}
```

### Terraform - ExpressRoute Gateway

```hcl
resource "azurerm_public_ip" "ergw" {
  name                = "pip-ergw-hub"
  location            = var.location
  resource_group_name = var.resource_group_name
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]
}

resource "azurerm_virtual_network_gateway" "ergw" {
  name                = "ergw-hub-prod"
  location            = var.location
  resource_group_name = var.resource_group_name

  type     = "ExpressRoute"
  vpn_type = "RouteBased"

  sku = "ErGw2AZ"

  ip_configuration {
    name                          = "default"
    public_ip_address_id          = azurerm_public_ip.ergw.id
    private_ip_address_allocation = "Dynamic"
    subnet_id                     = azurerm_subnet.gateway.id
  }
}

resource "azurerm_virtual_network_gateway_connection" "ergw" {
  name                = "conn-ergw-to-circuit"
  location            = var.location
  resource_group_name = var.resource_group_name

  type                       = "ExpressRoute"
  virtual_network_gateway_id = azurerm_virtual_network_gateway.ergw.id
  express_route_circuit_id   = var.expressroute_circuit_id
}
```

## Exam Relevance

### AZ-700 Objectives

- Configure VPN Gateway
- Configure ExpressRoute
- Implement hybrid connectivity
- Troubleshoot gateway connectivity
- Configure gateway transit

### AZ-305 Objectives

- Design hybrid network connectivity
- Recommend gateway solutions
- Design for high availability
- Design for disaster recovery

## Summary

Hybrid connectivity in hub-spoke architectures:

- **VPN Gateway** for encrypted internet-based connectivity
- **ExpressRoute** for dedicated private connectivity
- **Gateway Transit** for spoke access through hub
- **BGP** for dynamic routing and failover
- **High availability** through redundancy options

Key considerations:

- Choose appropriate gateway SKU
- Implement redundancy for production
- Enable BGP route propagation on spokes
- Monitor gateway health and performance
- Plan for failover scenarios

## Next Steps

- **Flow 006**: Advanced Patterns (Virtual WAN, multi-region)
- **Task 007**: Setup VPN Gateway
- **Lab 03**: Hybrid Connectivity Implementation

-----

*Part of the Learning Azure Networking Topology: Hub-Spoke series*

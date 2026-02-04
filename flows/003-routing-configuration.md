# Flow 003: Routing Configuration

## Introduction

Routing is the mechanism that controls how network traffic flows in Azure hub-spoke topologies. This flow covers User-Defined Routes (UDRs), route tables, routing principles, and best practices essential for implementing effective hub-spoke architectures.

## Azure Routing Fundamentals

### How Azure Routes Traffic

Azure automatically routes traffic between:

- Subnets within a VNet
- Peered VNets
- To the internet
- To on-premises networks (via gateway)

**Default routing** handles basic connectivity, but hub-spoke architectures require **custom routing** to:

- Force spoke-to-spoke traffic through the hub
- Direct internet traffic through centralized firewall
- Control traffic flow for security and compliance

### Route Selection Process

When routing traffic, Azure uses this priority order:

1. **User-defined routes (UDRs)** - Highest priority
1. **BGP routes** - From VPN or ExpressRoute
1. **System routes** - Azure’s default routes

**Longest prefix match**: Most specific route (longest subnet mask) wins

Example:

- Route 1: 10.0.0.0/8 → Next hop A
- Route 2: 10.1.0.0/16 → Next hop B
- Traffic to 10.1.5.10 uses Route 2 (more specific)

## Route Tables

### What is a Route Table?

A route table is a collection of routes that controls traffic flow:

- Contains custom routes (UDRs)
- Associates with one or more subnets
- Overrides default Azure routing
- Enables centralized traffic control

### Route Table Components

```
Route Table:
├── Name: Descriptive identifier
├── Routes: Collection of route entries
├── Subnet Associations: Subnets using this table
└── BGP Route Propagation: Enable/disable gateway routes
```

### Route Entry Properties

Each route contains:

1. **Name**: Descriptive route identifier
1. **Address Prefix**: Destination CIDR (e.g., 10.2.0.0/16)
1. **Next Hop Type**: Where to send the traffic
1. **Next Hop Address**: Specific IP (for Virtual Appliance)

## Next Hop Types

### 1. Virtual Appliance

- **Purpose**: Route through Network Virtual Appliance (NVA) or Azure Firewall
- **Requires**: Next hop IP address
- **Use case**: Hub-spoke security inspection
- **Example**: 10.0.1.4 (Azure Firewall private IP)

### 2. Virtual Network Gateway

- **Purpose**: Send to VPN or ExpressRoute gateway
- **No IP needed**: Azure handles gateway routing
- **Use case**: Traffic to on-premises
- **Automatic**: Often via BGP propagation

### 3. Virtual Network (VNet)

- **Purpose**: Use VNet peering
- **No IP needed**: Azure uses peering automatically
- **Use case**: Standard VNet-to-VNet traffic
- **Default behavior**: Often redundant with system routes

### 4. Internet

- **Purpose**: Force traffic to internet
- **No IP needed**: Azure routes to internet
- **Use case**: Override default internet routing
- **Common**: Default route (0.0.0.0/0)

### 5. None

- **Purpose**: Explicitly drop/block traffic
- **Black hole**: Traffic is discarded
- **Use case**: Security blocking of specific ranges
- **Testing**: Temporary traffic isolation

## Hub-Spoke Routing Patterns

### Spoke-to-Spoke Routing

**Requirement**: Spokes must communicate through hub

**Spoke Route Table Configuration**:

```
Route Table: rt-spoke1
├── Route: spoke2-traffic
│   ├── Address Prefix: 10.2.0.0/16 (Spoke 2 range)
│   ├── Next Hop Type: Virtual Appliance
│   └── Next Hop Address: 10.0.1.4 (Hub firewall)
└── Route: spoke3-traffic
    ├── Address Prefix: 10.3.0.0/16 (Spoke 3 range)
    ├── Next Hop Type: Virtual Appliance
    └── Next Hop Address: 10.0.1.4 (Hub firewall)
```

**Traffic Flow**:

1. VM in Spoke 1 sends packet to Spoke 2
1. Route table directs to hub firewall (10.0.1.4)
1. Firewall inspects and forwards to Spoke 2
1. Return traffic follows reverse path

### Internet-Bound Routing

**Requirement**: Centralize internet egress through hub

**Spoke Route Table Configuration**:

```
Route Table: rt-spoke1
└── Route: default-internet
    ├── Address Prefix: 0.0.0.0/0
    ├── Next Hop Type: Virtual Appliance
    └── Next Hop Address: 10.0.1.4 (Hub firewall)
```

**Benefits**:

- Centralized security inspection
- Unified internet access policies
- Simplified compliance auditing
- Consistent egress IP addresses

### On-Premises Routing

**Requirement**: Access on-premises through hub gateway

**Configuration Options**:

#### Option 1: BGP Route Propagation (Recommended)

```
Route Table: rt-spoke1
└── BGP Route Propagation: Enabled
```

- Gateway automatically advertises on-premises routes
- Dynamic updates as routes change
- No manual route maintenance
- Best for most scenarios

#### Option 2: Static Routes

```
Route Table: rt-spoke1
└── Route: onprem-datacenter
    ├── Address Prefix: 192.168.0.0/16 (On-premises range)
    ├── Next Hop Type: Virtual Network Gateway
    └── Next Hop Address: (not required)
```

- Manual route management
- Use when BGP not available
- Requires updates for topology changes

## Azure Firewall Routing

### Firewall Subnet Routes

**Special Consideration**: Azure Firewall subnet typically needs minimal routing

```
Route Table: rt-azurefirewall (optional)
└── Route: optional-routes
    ├── Usually relies on system routes
    └── Specific routes for complex scenarios only
```

**Why minimal routing?**

- Firewall receives traffic from UDRs pointing to it
- Return traffic uses learned routes
- Symmetric routing handled automatically

### Firewall IP Configuration

Azure Firewall private IP address:

- Automatically assigned from AzureFirewallSubnet
- Static after creation
- Use this IP in spoke route tables
- Typically first usable IP in subnet (e.g., 10.0.1.4)

## Advanced Routing Scenarios

### Forced Tunneling

**Purpose**: Force all internet traffic through on-premises

**Configuration**:

```
Route Table: rt-azurefirewall
└── Route: force-tunnel
    ├── Address Prefix: 0.0.0.0/0
    ├── Next Hop Type: Virtual Network Gateway
    └── Purpose: Send internet via on-premises
```

**Use case**:

- Compliance requirements
- Centralized internet inspection on-premises
- Hybrid security architecture

**Consideration**: Azure services may require internet access

### Route Aggregation

**Purpose**: Simplify routing with summary routes

**Example**:

Instead of:

```
10.1.0.0/24 → Firewall
10.2.0.0/24 → Firewall
10.3.0.0/24 → Firewall
```

Use:

```
10.0.0.0/8 → Firewall
```

**Benefits**:

- Fewer routes to manage
- Reduced route table complexity
- Easier maintenance

**Requirement**: Contiguous address space

### Multi-NIC and Multiple Route Tables

- Different subnets can have different route tables
- Allows segmented routing within a VNet
- Complex but flexible for specific requirements

**Example**:

```
VNet: spoke1
├── subnet-web
│   └── Route Table: rt-web (routes to internet)
└── subnet-data
    └── Route Table: rt-data (no internet access)
```

## Route Table Deployment Strategy

### Hub VNet Route Tables

#### Gateway Subnet

```
Route Table: Generally none required
- BGP handles on-premises routes
- Exception: Forced tunneling scenarios
```

#### Azure Firewall Subnet

```
Route Table: Usually not required
- System routes typically sufficient
- Add specific routes only if needed
```

#### Management Subnet

```
Route Table: rt-hub-management
└── Route: internet-via-firewall
    ├── Address Prefix: 0.0.0.0/0
    ├── Next Hop Type: Virtual Appliance
    └── Next Hop Address: 10.0.1.4
```

### Spoke VNet Route Tables

#### Standard Spoke Route Table

```
Route Table: rt-spoke-standard
├── Route: other-spokes
│   ├── Address Prefix: 10.0.0.0/8 (All spoke ranges)
│   ├── Next Hop Type: Virtual Appliance
│   └── Next Hop Address: 10.0.1.4
├── Route: internet
│   ├── Address Prefix: 0.0.0.0/0
│   ├── Next Hop Type: Virtual Appliance
│   └── Next Hop Address: 10.0.1.4
└── BGP Route Propagation: Enabled (for on-premises)
```

**Applied to**: All application subnets in spoke

## Routing Best Practices

### 1. Plan Routes Before Deployment

- Document required traffic flows
- Map source, destination, and next hop
- Consider return path (symmetric routing)
- Validate against security requirements

### 2. Use Descriptive Naming

**Route Table Naming**:

- `rt-{purpose}-{location}-{environment}`
- Examples: `rt-spoke1-westeurope-prod`, `rt-firewall-eastus-dev`

**Route Naming**:

- `route-{destination}-{purpose}`
- Examples: `route-spoke2-traffic`, `route-internet-egress`

### 3. Minimize Route Complexity

- Use route aggregation where possible
- Avoid unnecessary specific routes
- Keep route tables simple and maintainable
- Document exceptions and special cases

### 4. Implement Consistent Patterns

- Standardize route table configurations
- Use same approach across all spokes
- Document standard patterns
- Automate deployment with IaC

### 5. Enable BGP Propagation Appropriately

**Enable when**:

- Spoke needs on-premises access
- VPN/ExpressRoute gateway present in hub
- Dynamic route updates desired

**Disable when**:

- No gateway connectivity needed
- Static routing preferred
- Want to control specific routes manually

## Monitoring and Troubleshooting

### Effective Routes

**View effective routes** for a network interface:

- Shows all routes (UDR + System + BGP)
- Indicates which route is active
- Essential for troubleshooting

**Azure Portal**:
Network Interface → Effective Routes

**Azure CLI**:

```bash
az network nic show-effective-route-table \
  --resource-group rg-spoke1 \
  --name nic-vm1
```

### Next Hop Analysis

**Network Watcher → Next Hop**:

- Tests where traffic will route
- Validates UDR configuration
- Identifies routing issues

**Parameters**:

- Source VM
- Destination IP
- Shows next hop type and IP

### Common Routing Issues

#### Issue: Spoke-to-Spoke Traffic Not Working

**Symptoms**:

- VMs in different spokes cannot communicate
- Connectivity works within same spoke

**Diagnosis**:

1. Check effective routes on source VM NIC
1. Verify route table has destination spoke range
1. Confirm next hop points to firewall
1. Verify firewall rules allow traffic

**Resolution**:

- Add UDR for destination spoke
- Configure firewall to allow traffic
- Check NSG rules

#### Issue: Internet Access Not Working

**Symptoms**:

- VMs cannot reach internet
- Works from some subnets but not others

**Diagnosis**:

1. Check default route (0.0.0.0/0)
1. Verify next hop configuration
1. Check firewall NAT rules
1. Validate firewall outbound rules

**Resolution**:

- Add default route to route table
- Configure firewall SNAT
- Verify route table association

#### Issue: Asymmetric Routing

**Symptoms**:

- Traffic works in one direction only
- Inconsistent connectivity

**Diagnosis**:

- Check return path routing
- Verify both directions use same path
- Review firewall and NVA configuration

**Resolution**:

- Ensure symmetric routing
- Adjust route tables for both directions
- Configure firewall to handle return traffic

## Route Table Limits

|Resource                     |Default Limit|Maximum Limit|
|-----------------------------|-------------|-------------|
|Route tables per subscription|200          |800          |
|Routes per route table       |400          |400          |
|User-defined routes per VNet |400          |400          |
|BGP routes via VNet gateway  |4000         |10,000       |

**Planning Considerations**:

- Plan route consolidation for large deployments
- Use summary routes when possible
- Monitor route count in complex environments

## Infrastructure as Code Examples

### Bicep - Route Table

```bicep
resource routeTable 'Microsoft.Network/routeTables@2023-04-01' = {
  name: 'rt-spoke1-prod'
  location: location
  properties: {
    disableBgpRoutePropagation: false
    routes: [
      {
        name: 'route-to-firewall'
        properties: {
          addressPrefix: '0.0.0.0/0'
          nextHopType: 'VirtualAppliance'
          nextHopIpAddress: '10.0.1.4'
        }
      }
      {
        name: 'route-to-spoke2'
        properties: {
          addressPrefix: '10.2.0.0/16'
          nextHopType: 'VirtualAppliance'
          nextHopIpAddress: '10.0.1.4'
        }
      }
    ]
  }
}
```

### Terraform - Route Table

```hcl
resource "azurerm_route_table" "spoke1" {
  name                          = "rt-spoke1-prod"
  location                      = var.location
  resource_group_name           = var.resource_group_name
  disable_bgp_route_propagation = false

  route {
    name                   = "route-to-firewall"
    address_prefix         = "0.0.0.0/0"
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = "10.0.1.4"
  }

  route {
    name                   = "route-to-spoke2"
    address_prefix         = "10.2.0.0/16"
    next_hop_type          = "VirtualAppliance"
    next_hop_in_ip_address = "10.0.1.4"
  }
}

resource "azurerm_subnet_route_table_association" "spoke1_web" {
  subnet_id      = azurerm_subnet.web.id
  route_table_id = azurerm_route_table.spoke1.id
}
```

## Performance Considerations

### Routing Latency

- UDRs add minimal latency (~microseconds)
- Firewall/NVA adds more latency (milliseconds)
- Plan for additional hops in latency budget
- Monitor end-to-end latency

### Throughput Impact

- Azure Firewall throughput limits:
  - Standard: 30 Gbps
  - Premium: 100 Gbps
- NVA throughput varies by instance size
- Plan capacity for aggregate spoke traffic

## Exam Relevance

### AZ-700 Objectives

- Configure user-defined routes
- Configure routing in hub-spoke topology
- Troubleshoot routing issues
- Monitor routing with Network Watcher

### AZ-305 Objectives

- Design routing solutions
- Design for network traffic flow
- Recommend routing architecture
- Design for security requirements

## Summary

Effective routing in hub-spoke architectures requires:

- **User-Defined Routes** to control traffic flow
- **Route tables** on spoke subnets
- **Proper next hop configuration** to firewall/NVA
- **BGP propagation** for on-premises connectivity
- **Monitoring and troubleshooting** tools

Key principles:

- Spoke-to-spoke must route through hub
- Internet egress centralized to hub
- On-premises access via hub gateway
- Symmetric routing essential

## Next Steps

- **Flow 004**: Security Implementation (NSGs and Azure Firewall)
- **Task 005**: Configure Route Tables (hands-on)
- **Lab 01**: Basic Hub-Spoke with Routing

-----

*Part of the Learning Azure Networking Topology: Hub-Spoke series*

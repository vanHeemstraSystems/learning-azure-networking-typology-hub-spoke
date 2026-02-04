# Learning Azure Networking Topology: Hub & Spoke

A comprehensive learning resource for understanding and implementing Azure hub-spoke network topology patterns, created by Willem van Heemstra.

## Overview

This repository provides structured learning materials for mastering Azure hub-spoke networking architecture, a fundamental pattern for enterprise cloud deployments. The hub-spoke topology centralizes shared services in a hub virtual network while isolating workloads in spoke virtual networks, enabling efficient management, security, and connectivity.

## Learning Objectives

- Understand the fundamental concepts of hub-spoke network topology
- Design and implement hub-spoke architectures in Azure
- Configure Azure Virtual Network peering and routing
- Implement network security using NSGs and Azure Firewall
- Integrate hybrid connectivity using VPN Gateway and ExpressRoute
- Apply best practices for scalability and security
- Troubleshoot common networking issues

## Target Audience

- Cloud Engineers preparing for AZ-700 and AZ-305 certifications
- Network Engineers transitioning to Azure
- Solutions Architects designing enterprise Azure deployments
- DevOps Engineers implementing infrastructure as code

## Directory Structure

```
.
├── README.md
├── flows/
│   ├── 001-hub-spoke-overview.md
│   ├── 002-virtual-network-peering.md
│   ├── 003-routing-configuration.md
│   ├── 004-security-implementation.md
│   ├── 005-hybrid-connectivity.md
│   └── 006-advanced-patterns.md
├── stories/
│   ├── story-001-basic-hub-spoke.md
│   ├── story-002-multi-region-hub-spoke.md
│   ├── story-003-secured-hub-spoke.md
│   └── story-004-hybrid-hub-spoke.md
├── tasks/
│   ├── task-001-create-hub-vnet.md
│   ├── task-002-create-spoke-vnets.md
│   ├── task-003-configure-peering.md
│   ├── task-004-deploy-azure-firewall.md
│   ├── task-005-configure-route-tables.md
│   ├── task-006-implement-nsgs.md
│   ├── task-007-setup-vpn-gateway.md
│   └── task-008-configure-dns.md
├── diagrams/
│   ├── hub-spoke-basic.png
│   ├── hub-spoke-with-firewall.png
│   ├── multi-region-hub-spoke.png
│   └── hybrid-hub-spoke.png
├── templates/
│   ├── bicep/
│   │   ├── hub-vnet.bicep
│   │   ├── spoke-vnet.bicep
│   │   ├── vnet-peering.bicep
│   │   ├── azure-firewall.bicep
│   │   └── main.bicep
│   └── terraform/
│       ├── hub-vnet.tf
│       ├── spoke-vnet.tf
│       ├── vnet-peering.tf
│       ├── azure-firewall.tf
│       └── variables.tf
├── scripts/
│   ├── deploy-hub-spoke.sh
│   ├── validate-connectivity.ps1
│   └── cleanup-resources.sh
├── labs/
│   ├── lab-01-basic-hub-spoke.md
│   ├── lab-02-add-azure-firewall.md
│   ├── lab-03-hybrid-connectivity.md
│   └── lab-04-multi-region.md
├── references/
│   ├── azure-networking-limits.md
│   ├── troubleshooting-guide.md
│   ├── best-practices.md
│   └── additional-resources.md
└── notes/
    ├── exam-tips-az700.md
    ├── exam-tips-az305.md
    └── gotchas-and-lessons-learned.md
```

## Content Structure

### Flows

Sequential learning modules that build understanding progressively:

- **001-hub-spoke-overview.md**: Introduction to hub-spoke topology, benefits, and use cases
- **002-virtual-network-peering.md**: VNet peering concepts, configuration, and limitations
- **003-routing-configuration.md**: User-defined routes, route tables, and traffic flow
- **004-security-implementation.md**: NSGs, Azure Firewall, and network security
- **005-hybrid-connectivity.md**: VPN Gateway and ExpressRoute integration
- **006-advanced-patterns.md**: Multi-region, Virtual WAN, and complex scenarios

### Stories

Real-world scenarios with complete implementation examples:

- **story-001-basic-hub-spoke.md**: Simple hub with 2-3 spokes for dev/test/prod
- **story-002-multi-region-hub-spoke.md**: Regional hubs with global peering
- **story-003-secured-hub-spoke.md**: Hub-spoke with Azure Firewall and forced tunneling
- **story-004-hybrid-hub-spoke.md**: Hub-spoke connecting to on-premises datacenter

### Tasks

Discrete, actionable steps for specific implementations:

- Creating and configuring hub and spoke VNets
- Establishing VNet peering relationships
- Deploying and configuring Azure Firewall
- Setting up route tables and custom routes
- Implementing network security groups
- Configuring VPN Gateway for hybrid connectivity
- DNS configuration and private DNS zones

### Diagrams

Visual representations of architectures and traffic flows

### Templates

Infrastructure as Code for quick deployment:

- **Bicep templates**: Azure-native IaC for hub-spoke deployment
- **Terraform templates**: Multi-cloud compatible configurations

### Scripts

Automation scripts for deployment, validation, and cleanup

### Labs

Hands-on exercises with step-by-step instructions

### References

- Azure networking service limits and quotas
- Troubleshooting connectivity issues
- Security and performance best practices
- Links to Microsoft documentation and community resources

### Notes

- AZ-700 and AZ-305 exam preparation tips
- Common pitfalls and gotchas
- Lessons learned from real implementations

## Getting Started

1. **Start with Flows**: Read through the flow documents in sequence to build foundational understanding
1. **Review Stories**: Study real-world scenarios that align with your use case
1. **Practice with Labs**: Complete hands-on labs to gain practical experience
1. **Deploy Templates**: Use provided IaC templates to deploy test environments
1. **Reference as Needed**: Use reference materials for troubleshooting and best practices

## Prerequisites

- Active Azure subscription
- Basic understanding of networking concepts (IP addressing, routing, DNS)
- Familiarity with Azure portal or Azure CLI
- (Optional) Terraform or Bicep knowledge for IaC deployment

## Related Learning Resources

- learning-azure-networking
- learning-azure-security
- learning-terraform
- learning-bicep

## Certification Alignment

This repository supports preparation for:

- **AZ-700**: Designing and Implementing Microsoft Azure Networking Solutions
- **AZ-305**: Designing Microsoft Azure Infrastructure Solutions

Relevant exam objectives are tagged in each learning module.

## Contributing

This is a personal learning repository, but suggestions and corrections are welcome via issues.

## Author

**Willem van Heemstra**  
Cloud Engineer | Azure Specialist | Security Enthusiast

## License

This learning resource is provided as-is for educational purposes.

## Acknowledgments

- Microsoft Azure documentation
- Azure architecture patterns and best practices
- Community contributions and real-world implementations

-----

*Last updated: February 2026*

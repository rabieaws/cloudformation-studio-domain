# Requirements Document

## Introduction

This feature enhances the SageMaker CloudFormation template to dynamically calculate subnet CIDR blocks based on the user-provided VPC CIDR parameter. Currently, the template has hardcoded subnet CIDRs (10.0.0.0/24 and 10.0.1.0/24) which fail when users provide different VPC CIDR blocks. The solution will use CloudFormation intrinsic functions or custom Lambda resources to automatically derive appropriate subnet CIDR blocks from any valid VPC CIDR input.

## Glossary

- **CloudFormation Template**: The AWS infrastructure-as-code YAML file that defines the SageMaker domain and networking resources
- **VPC CIDR**: The IP address range for the Virtual Private Cloud in CIDR notation (e.g., 10.0.0.0/16)
- **Subnet CIDR**: The IP address range for a subnet within the VPC, must be a subset of the VPC CIDR
- **CIDR Block**: Classless Inter-Domain Routing notation for IP address ranges (e.g., 192.168.0.0/24)
- **Fn::Cidr Function**: CloudFormation intrinsic function that generates CIDR blocks from a parent CIDR

## Requirements

### Requirement 1

**User Story:** As a DevOps engineer, I want to provide any valid VPC CIDR block as input, so that the template automatically creates compatible subnets without manual CIDR calculation

#### Acceptance Criteria

1. WHEN a user provides a VPC CIDR parameter, THE CloudFormation Template SHALL calculate subnet CIDR blocks that are valid subsets of the provided VPC CIDR
2. THE CloudFormation Template SHALL create exactly two private subnets with non-overlapping CIDR blocks
3. THE CloudFormation Template SHALL ensure each subnet CIDR block is appropriately sized for the SageMaker workload requirements
4. THE CloudFormation Template SHALL validate that calculated subnet CIDRs do not exceed the boundaries of the parent VPC CIDR
5. WHEN the VPC CIDR is modified, THE CloudFormation Template SHALL recalculate subnet CIDRs automatically during stack update

### Requirement 2

**User Story:** As a cloud architect, I want the subnet sizing to be consistent and predictable, so that I can plan IP address allocation across multiple environments

#### Acceptance Criteria

1. THE CloudFormation Template SHALL use a consistent algorithm to derive subnet CIDR blocks from the VPC CIDR
2. THE CloudFormation Template SHALL create subnets with a /24 netmask when the VPC CIDR allows sufficient address space
3. WHERE the VPC CIDR is smaller than /22, THE CloudFormation Template SHALL adjust subnet sizes to fit within available address space
4. THE CloudFormation Template SHALL document the subnet sizing logic in template comments or description

### Requirement 3

**User Story:** As a template user, I want clear error messages if my VPC CIDR is incompatible, so that I can quickly identify and fix configuration issues

#### Acceptance Criteria

1. WHEN a user provides a VPC CIDR that is too small to accommodate two subnets, THE CloudFormation Template SHALL fail validation with a descriptive error message
2. THE CloudFormation Template SHALL validate the VPC CIDR parameter format before attempting subnet calculation
3. IF subnet calculation fails, THEN THE CloudFormation Template SHALL provide an error message indicating the minimum required VPC CIDR size
4. THE CloudFormation Template SHALL accept VPC CIDR blocks ranging from /16 to /28 netmask

### Requirement 4

**User Story:** As a security engineer, I want the subnet allocation to maintain network isolation, so that the two private subnets remain properly segmented

#### Acceptance Criteria

1. THE CloudFormation Template SHALL ensure the two calculated subnet CIDR blocks do not overlap
2. THE CloudFormation Template SHALL allocate subnets in different availability zones as currently implemented
3. THE CloudFormation Template SHALL maintain all existing VPC endpoint and security group configurations
4. THE CloudFormation Template SHALL preserve the private subnet routing table associations

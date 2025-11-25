# Design Document

## Overview

This design implements dynamic subnet CIDR calculation for the SageMaker CloudFormation template using the native `Fn::Cidr` intrinsic function. This approach eliminates hardcoded subnet CIDRs and allows the template to work with any valid VPC CIDR block provided by the user.

The solution leverages CloudFormation's built-in CIDR calculation capabilities to automatically generate two non-overlapping /24 subnet CIDR blocks from the user-provided VPC CIDR parameter. This approach is simple, maintainable, and requires no custom Lambda functions or external dependencies.

## Architecture

### Current State
- VPC CIDR: Parameterized (user input)
- Subnet 1 CIDR: Hardcoded to `10.0.0.0/24`
- Subnet 2 CIDR: Hardcoded to `10.0.1.0/24`
- Problem: Template fails when VPC CIDR is not `10.0.0.0/16`

### Target State
- VPC CIDR: Parameterized (user input)
- Subnet 1 CIDR: Dynamically calculated using `Fn::Cidr`
- Subnet 2 CIDR: Dynamically calculated using `Fn::Cidr`
- Result: Template works with any valid VPC CIDR input

### Design Decision: Fn::Cidr vs Custom Lambda

**Selected Approach: Fn::Cidr Intrinsic Function**

Rationale:
- Native CloudFormation functionality (no custom code)
- No additional IAM roles or Lambda resources required
- Simpler to maintain and understand
- Faster stack creation (no Lambda cold starts)
- Widely documented and supported

**Alternative Considered: Custom Lambda Resource**
- Rejected due to added complexity
- Would require additional IAM permissions
- Increases template size and maintenance burden
- Provides no significant advantage over Fn::Cidr

## Components and Interfaces

### Modified Resources

#### 1. SageMakerPrivateSubnet1
**Current Implementation:**
```yaml
SageMakerPrivateSubnet1:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref SageMakerVPC
    CidrBlock: 10.0.0.0/24
    AvailabilityZone: !Select [0, !GetAZs !Ref 'AWS::Region']
```

**New Implementation:**
```yaml
SageMakerPrivateSubnet1:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref SageMakerVPC
    CidrBlock: !Select [0, !Cidr [!Ref VPCCidr, 2, 8]]
    AvailabilityZone: !Select [0, !GetAZs !Ref 'AWS::Region']
```

#### 2. SageMakerPrivateSubnet2
**Current Implementation:**
```yaml
SageMakerPrivateSubnet2:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref SageMakerVPC
    CidrBlock: 10.0.1.0/24
    AvailabilityZone: !Select [1, !GetAZs !Ref 'AWS::Region']
```

**New Implementation:**
```yaml
SageMakerPrivateSubnet2:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref SageMakerVPC
    CidrBlock: !Select [1, !Cidr [!Ref VPCCidr, 2, 8]]
    AvailabilityZone: !Select [1, !GetAZs !Ref 'AWS::Region']
```

### Fn::Cidr Function Explanation

**Syntax:** `!Cidr [ipBlock, count, cidrBits]`

**Parameters:**
- `ipBlock`: The VPC CIDR block (e.g., 10.0.0.0/16)
- `count`: Number of subnets to generate (2 in our case)
- `cidrBits`: Number of subnet bits for the CIDR (8 = /24 subnets)

**Example Calculations:**

For VPC CIDR `10.0.0.0/16`:
- `!Cidr [10.0.0.0/16, 2, 8]` generates:
  - `10.0.0.0/24` (first subnet)
  - `10.0.1.0/24` (second subnet)

For VPC CIDR `192.168.0.0/16`:
- `!Cidr [192.168.0.0/16, 2, 8]` generates:
  - `192.168.0.0/24` (first subnet)
  - `192.168.1.0/24` (second subnet)

For VPC CIDR `172.31.0.0/20`:
- `!Cidr [172.31.0.0/20, 2, 8]` generates:
  - `172.31.0.0/28` (first subnet)
  - `172.31.0.16/28` (second subnet)

**Note:** The `cidrBits` parameter of 8 creates /24 subnets when the VPC is /16. For smaller VPCs, CloudFormation automatically adjusts to fit within the available address space.

## Data Models

No data model changes required. The modification only affects infrastructure resource definitions.

## Error Handling

### VPC CIDR Validation

The existing parameter validation remains in place:
```yaml
VPCCidr:
  Type: String
  Description: "Enter a valid CIDR block (e.g., 10.0.0.0/16)"
  AllowedPattern: ^([0-9]{1,3}\.){3}[0-9]{1,3}/(3[0-2]|[1-2]?[0-9])$
  ConstraintDescription: "Must be a valid CIDR block in the format x.x.x.x/x"
  Default: "10.0.0.0/16"
```

### Subnet Size Constraints

CloudFormation's `Fn::Cidr` function automatically handles:
- VPC CIDR blocks that are too small (will fail with descriptive error)
- Subnet count exceeding available address space
- Invalid CIDR notation

### Expected Error Scenarios

1. **VPC CIDR too small for requested subnets:**
   - Error: "The CIDR block does not have enough space for the requested subnets"
   - Solution: User must provide a larger VPC CIDR

2. **Invalid CIDR format:**
   - Caught by parameter validation regex
   - Error: "Must be a valid CIDR block in the format x.x.x.x/x"

## Testing Strategy

### Unit Testing (Manual Validation)

Test the template with various VPC CIDR inputs:

1. **Standard /16 VPC:**
   - Input: `10.0.0.0/16`
   - Expected Subnets: `10.0.0.0/24`, `10.0.1.0/24`

2. **Different /16 VPC:**
   - Input: `192.168.0.0/16`
   - Expected Subnets: `192.168.0.0/24`, `192.168.1.0/24`

3. **Smaller /20 VPC:**
   - Input: `172.31.0.0/20`
   - Expected Subnets: `172.31.0.0/28`, `172.31.0.16/28`

4. **Large /24 VPC:**
   - Input: `10.10.10.0/24`
   - Expected Subnets: `10.10.10.0/26`, `10.10.10.64/26`

5. **Edge Case - /28 VPC:**
   - Input: `10.0.0.0/28`
   - Expected: Should fail or create very small subnets

### Integration Testing

1. Deploy the CloudFormation stack with default VPC CIDR
2. Verify SageMaker domain creation succeeds
3. Verify VPC endpoints are properly configured
4. Update stack with different VPC CIDR
5. Verify subnets are recalculated correctly

### Validation Checklist

- [ ] Template validates successfully with `aws cloudformation validate-template`
- [ ] Stack creates successfully with default parameters
- [ ] Stack creates successfully with custom VPC CIDR (192.168.0.0/16)
- [ ] Subnets are in different availability zones
- [ ] VPC endpoints connect to both subnets
- [ ] SageMaker domain is accessible
- [ ] No hardcoded IP addresses remain in template

## Implementation Notes

### Backward Compatibility

This change modifies subnet CIDR allocation, which means:
- **New stacks:** Will work with any VPC CIDR
- **Existing stacks:** Updating will cause subnet replacement (destructive change)
- **Recommendation:** For existing stacks, consider creating a new stack rather than updating

### Subnet Sizing Considerations

The `cidrBits` value of 8 is chosen to create /24 subnets (256 IP addresses each) which provides:
- 251 usable IP addresses per subnet (AWS reserves 5)
- Sufficient capacity for typical SageMaker workloads
- Room for growth and additional resources

If larger subnets are needed, the `cidrBits` parameter can be adjusted:
- `cidrBits: 7` → /23 subnets (512 IPs)
- `cidrBits: 6` → /22 subnets (1024 IPs)
- `cidrBits: 9` → /25 subnets (128 IPs)

### Documentation Updates

The template description should be updated to reflect the dynamic subnet calculation capability.

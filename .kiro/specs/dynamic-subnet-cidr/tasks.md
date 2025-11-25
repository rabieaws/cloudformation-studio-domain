# Implementation Plan

- [ ] 1. Update subnet CIDR blocks to use dynamic calculation
  - [ ] 1.1 Replace hardcoded CIDR in SageMakerPrivateSubnet1 with Fn::Cidr function
    - Modify the `CidrBlock` property in `sagemaker-domain-with-vpc.yaml` (line ~152) from `10.0.0.0/24` to `!Select [0, !Cidr [!Ref VPCCidr, 2, 8]]`
    - Preserve all other subnet properties (VpcId, AvailabilityZone, Tags)
    - _Requirements: 1.1, 1.5, 2.1_
  - [ ] 1.2 Replace hardcoded CIDR in SageMakerPrivateSubnet2 with Fn::Cidr function
    - Modify the `CidrBlock` property in `sagemaker-domain-with-vpc.yaml` (line ~164) from `10.0.1.0/24` to `!Select [1, !Cidr [!Ref VPCCidr, 2, 8]]`
    - Preserve all other subnet properties (VpcId, AvailabilityZone, Tags)
    - _Requirements: 1.1, 1.5, 2.1_
  - [ ] 1.3 Add inline comments explaining the Fn::Cidr function parameters
    - Add comments above the SageMakerPrivateSubnet1 CidrBlock explaining the Fn::Cidr function
    - Document what the count parameter (2) represents: number of subnets to create
    - Document what the cidrBits parameter (8) represents: creates /24 subnets from /16 VPC
    - Add example: "For VPC 10.0.0.0/16, generates 10.0.0.0/24 and 10.0.1.0/24"
    - _Requirements: 2.4_

- [ ] 2. Update template description and documentation
  - [ ] 2.1 Update the CloudFormation template Description field
    - Modify the Description at the top of `sagemaker-domain-with-vpc.yaml` (line ~2)
    - Add mention that subnet CIDRs are dynamically calculated from the VPC CIDR parameter
    - Note that template works with any valid VPC CIDR input
    - _Requirements: 2.4_
  - [ ] 2.2 Enhance VPCCidr parameter description
    - Update the VPCCidr parameter Description field (line ~20)
    - Clarify that subnets will be automatically calculated as /24 blocks (or smaller if VPC is smaller)
    - Add note about minimum VPC size requirements (recommend /22 or larger for two /24 subnets)
    - _Requirements: 3.1, 3.3, 3.4_

- [x] 3. Validate template changes






  - [-] 3.1 Test template syntax validation




    - Run `aws cloudformation validate-template --template-body file://sagemaker-domain-with-vpc.yaml` command
    - Verify no syntax errors are present
    - _Requirements: 3.2_
  - [ ]* 3.2 Create test deployment with default VPC CIDR
    - Deploy stack with default 10.0.0.0/16 VPC CIDR
    - Verify subnets are created with expected CIDR blocks (10.0.0.0/24, 10.0.1.0/24)
    - Verify SageMaker domain creates successfully
    - _Requirements: 1.1, 1.2, 1.3, 4.1, 4.2, 4.3_
  - [ ]* 3.3 Test with alternative VPC CIDR blocks
    - Test with 192.168.0.0/16 and verify subnet calculation
    - Test with 172.31.0.0/20 and verify subnet calculation
    - Test with 10.10.0.0/24 and verify subnet calculation
    - _Requirements: 1.1, 1.5, 2.1, 2.3, 3.4_

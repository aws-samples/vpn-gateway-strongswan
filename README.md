# VPN Gateway Stack Using strongSwan

CloudFormation template to deploy the open source [strongSwan VPN](https://www.strongswan.org/) solution to act as a VPN gateway in support of site-to-site VPN connections.

The open source [Quagga](https://en.wikipedia.org/wiki/Quagga_(software) ) software suite is used to provide [Border Gateway Protocol (BGP)](https://searchnetworking.techtarget.com/definition/BGP-Border-Gateway-Protocol) support to automatically propagate routing information across a site-to-site VPN connection.

* [Use Cases](#use-cases)
* [CloudFormation Features Demonstrated](#cloudformation-features-demonstrated)
* [Usage](#usage)
* [Parameters](#parameters)
* [Troubleshooting](#troubleshooting)
* [Advanced Usage](#advanced-usage)
* [Contributing](#contributing)
* [License](#license)

## Use Cases

* **Demonstration and Lab Environments Representing On-Premises VPN Gateways:** This stack can be useful to help demonstrate how to integrate an on-premises VPN gateway with AWS networks via Virtual Private Gateways (VPGs) and Transit Gateways (TGWs).  

* **Both Ends of a Site-to-Site VPN Connection:** This stack can also be used on both ends of a site-to-site VPN connection in scenarios where VPGs and TGWs are not applicable. For example, until TGWs support inter-region routing, you can demonstrate how to fill the gap by using a site-to-site VPN using this stack on both ends.  See the "Transit VPC" section of [Multiple Region Multi-VPC Connectivity](https://aws.amazon.com/answers/networking/aws-multiple-region-multi-vpc-connectivity/) for the scenario.

## CloudFormation Features Demonstrated

Even if you're not interested in deploying a VPN gateway, but you're interested in better understanding what you can do with CloudFormation in support of Infrastructure as Code (IaC), you might find value in reviewing how the template makes use of these generally applicable capabilities:

* [`Fn::ImportValue`](https://docs.aws.amazon.com/en_pv/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-importvalue.html) to import data exported from other CloudFormation stacks.
* [`AWS::CloudFormation::Init`](https://docs.aws.amazon.com/en_pv/AWSCloudFormation/latest/UserGuide/aws-resource-init.html) to completely automate the build out of the VPN gateway stack and BGP support upon first boot.
* [`AWS::CloudFormation::WaitCondition`](https://docs.aws.amazon.com/en_pv/AWSCloudFormation/latest/UserGuide/aws-properties-waitcondition.html) to force the stack creation process to wait until the first boot build out is complete.
* AWS CloudWatch Logs integration via the [CloudWatch Logs Agent](https://docs.aws.amazon.com/en_pv/AmazonCloudWatch/latest/logs/CWL_GettingStarted.html) in which OS, VPN gateway, and BGP log files are written to a series of log streams in a CloudWatch Logs log group.
* CloudWatch monitoring integration for [monitoring EC2 memory and disk metrics](https://docs.aws.amazon.com/en_pv/AWSEC2/latest/UserGuide/mon-scripts.html).
* Use of [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/en_pv/systems-manager/latest/userguide/session-manager.html) to enable secure terminal access to the OS instance without the need to establish Internet accessible bastion hosts and port 22 access to the VPN gateway.
* Using [Systems Manager Parameter Store](https://aws.amazon.com/blogs/compute/query-for-the-latest-amazon-linux-ami-ids-using-aws-systems-manager-parameter-store/) to query for latest Amazon Linuix 2 Amazon Machine Image (AMI) images.

## Usage

### 1. Obtain Elastic IP Address

Decoupling creation and management of the Elastic IP Address (EIP) from the creation and management of the VPN gateway is useful so that you can replace the VPN gateway stack at will without needing to reconfigure the remote end of the site-to-site VPN connection.  By preserving the local EIP and attaching to an replacement stack's EC2 instance, the remote end of the connection is unaffected.

Your first task is to use the following EIP template to create a new EIP that will be used in subsequent steps to help configure the remote end of the VPN connection and drive the configuration of the local VPN gateway stack:

[https://github.com/ckamps/infra-aws-elastic-ip](https://github.com/ckamps/infra-aws-elastic-ip)

The VPN gateway template that you will be using in a subsequent step is configured to import an Elastic IP allocation ID from an EIP resource created via this EIP template:

### 2. Arrive at VPN Tunnel Configuration Settings

When using either AWS VPGs or TGWs for the remote end of the site-to-site VPN connection, a site-to-site VPN connection resource will be established in AWS on the remote site. Within the site-to-site VPN connection resource on the remote site, you can download a VPN configuration file that will provide you with much of the data required to deploy the local VPN gateway. See `VPC -> Site-to-Site VPN Connections`, select the connection of interest, click `Download` and select the `Generic` option for `Vendor` and download the configuration file.

Review this file in preparation for using some of the data in the next step.

### 3. Deploy VPN Gateway Stack

In this step you'll create a CloudFormation stack using the [`vpn-gateway-strongswan.yml`](./vpn-gateway-strongswan.yml) template and configuration data obtained from the remote site's Site-to-Site VPN Connection resource.

* Use the CloudFormation template to deploy a VPN gateway stack in a public subnet based on the parameters described below.
* Wait for creation of the stack to complete. Since the template uses a wait condition, the stack won't complete until strongSwan and other components have been configured and started.
* Wait for several minutes after stack creation completes. Then monitor the Site-to-Site VPN Connection on the remote site to confirm that the two VPN tunnels have progressed from the `DOWN` state to the `UP` state.  If the VPN gateway configuration is correct, Tunnel 1 will come up first followed several minutes later by Tunnel 2.

If the tunnels have not come up 3-5 minutes after creation of the VPN gateway stack completed, then see the Troubleshooting section below.

### 4. Ensure Routing and EC2 Security Groups Are in Place

On both sides of the site-to-site VPN connection, ensure that the appropriate routing and security group configurations are in place to enable proper routing of traffic. For example:

* In support of testing, ensure that testing EC2 instances can receive ICMP or ping traffic.
* Ensure VPC route tables associated with subnets route traffic destined for the other site to the local VPN gateway instance.
* If using Transit Gateway on the remote site, ensure that VPC route tables are configured to route traffic destined for the other site to the Transit Gateway. (Although the built-in BGP support in this stack will ensure that both the local VPN gateway's route information and the remote Transit Gateway's route table will be automatically configuired, you still need to ensure that the VPC route tables in both sites are properly configured).

### 5. Test

* Deploy an Amazon Linux EC2 instance to one of the local subnets.  
    * Ensure ICMP is allowed as inbound traffic.
    * Set it up for SSH access in one of two ways:
      * Systems Manager Session Manager: No SSH and publicly accesible IP address required. Instead, create an IAM role for EC2 that includes the `AmazonSSMManagedInstanceCore` policy and attach it to the EC2 instance via the `Actions -> Instance Settings -> Attach/Replace IAM Role`.
      * SSH: Ensure that the security group allows for SSH inbound access and that the instance has a publicly accessible IP address.
* Deploy another EC2 instance in the remote site with the same configuration as above.
* Validate that route tables and security groups are properly configured.
* Use `ping` on one of the two ends to validate routing and connectivity between the instances.
* Use `# tcpdump -eni any icmp` to on the target instance to monitor traffic.

## Parameters

|Parameter|Required|Description|Default|
|---------|--------|-----------|-------|
|**System Classification and Environment**| | | |
|`pOrg`|Optional|Per AWS resource naming standards, the business organization to apply to resources such as IAM roles.|`acme`|
|`pSystem`|Optional|Per AWS resource naming standards, system ID to apply to AWS resources.|`infra`|
|`pApp`|Optional|Per AWS resource naming standards, app ID to apply to AWS resources.|`vpngw`|
|`pEnvPurpose`|Optional|Per AWS resource naming standards, qualifier as to purpose of this particular instance of the stack. For example, "dev1", "test", "1", etc.|None|
|**VPN Tunnel 1**| | | |
|`pTunnel1Psk`|Required|See the remote site's configuration for the "IPSec Tunnel #1" section and "Pre-Shared Key" value.|None|
|`pTunnel1RemoteExternalIpAddress`|Required|See the remote site's configuration for the "IPSec Tunnel #1" secton, "Outside IP Addresses" section and "Virtual Private Gateway" value.|None|
|`pTunnel1RemoteInsideCidr`|Required|See the remote site's configuration for the "IPSec Tunnel #1" secton, "Inside IP Addresses" section and "Virtual Private Gateway" value.|None|
|`pTunnel1LocalInsideCidr`|Required|See the remote site's configuration for the "IPSec Tunnel #1" secton, "Inside IP Addresses" section and "Customer Gateway" value.|None|
|`pTunnel1BgpAsn`|Optional|See the remote site's configuration for the "BGP Configuration Options" and the "Virtual Private  Gateway ASN" value.|`64512`|
|`pTunnel1BgpNeighborIpAddress`|Required|See the remote site's configuration for the "BGP Configuration Options" and the "Neighbor IP Address" value.|None|
|**VPN Tunnel 2**| | | |
|`pTunnel2Psk`|Required|See Tunnel 1.|None|
|`pTunnel2RemoteExternalIpAddress`|Required|See Tunnel 1.|None|
|`pTunnel2RemoteInsideCidr`|Required|See Tunnel 1.|None|
|`pTunnel2LocalInsideCidr`|Required|See Tunnel 1.|None|
|`pTunnel2BgpAsn`|Optional|See Tunnel 1.|`64512`|
|`pTunnel2BgpNeighborIpAddress`|Required|See Tunnel 1.|None|
|**Local Network**| | | |
|`pVpcId`|Required|The VPC in which the VPN gateway is to be deployed.|None|
|`pVpcCidr`|Required|The CIDR block of the local VPC. Used to advertise via BGP routing information to the remote site.|None|
|`pPublicSubnetId`|Required|The publicly accessible subnet in which the VPN gateway is to be deployed.|None|
|`pEipStackName`|Required|The name of the CloudFormation stack that was used to configure an Elastic IP address. See [https://github.com/ckamps/infra-aws-elastic-ip](https://github.com/ckamps/infra-aws-elastic-ip)|None|
|`pLocalBgpAsn`|Optional|The BGP Autonomous System Number (ASN) used to represent the local end of the site-to-site VPN connection.|`65000`|
|`pTunnel1BgpNeighborIpAddress`|Required|See the remote site's configuration.|None|
|**EC2 Instance**| | | |
|`pAmiId`|Optional|The ID of the AMI to use for the VPN gateway. By default this Systems Manager Parameter Store key is used to lookup the latest version of the referenced AMI for use in the current region.|`/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-ebs`|
|`pInstanceType`|Optional|The EC2 instance type to use for the VPN gateway.|`t3a.micro`|

## Troubleshooting

### VPN Gateway Stack Fails On Creation

TBD

### Tunnels Don't Come Up

It's likely that one or more of the tunnel related stack parameters is incorrect. Double check the settings.  You can delete and recreate the VPN gateway stack without needing to delete and recreate the remote site's VPN resources.

You can also inspect the VPN gateway's logs via CloudWatch Logs.  In CloudWatch Logs, look for a log group that is named based on the system classification parameters described above. For example: `/infra/vpngw/ec2/...`. If you see any of the following log files not present: `charon.log`, `zebra.log`, `bgpd.log`, then you should access the gateway instance and check the `systemctl` messages to see why a service did not start.

### Can't Ping Across the VPN Connection

Things to double check:
* EC2 instance security groups.
* Route tables.

Consider using `tcpdump` on the VPN gateway EC2 instance to see if traffic is being routed through the gateway.

## Advanced Usage

### Updating the VPN Gateway Stack

If you need to change resources that are configured outside of the `UserData` and `Metadata` sections of the `AWS::EC2::LaunchTemplate`, you should be able to update either the template or stack parameters and update the stack in place.

Until the `AWS::EC2::LaunchTemplate` is modified to support stack updates (see `TODO.md`), any changes in the `UserData` and `Metadata` sections of that resource require replacement of the stack.

### Replacing a VPN Gateway Stack

Since the Elastic IP Address resource is managed via a distinct CloudFormation stack, you can delete a VPN gateway stack without also deleting the associated EIP address. If you are using the VPN gateway stack to set up a site-to-site VPN with AWS VPG or TGW resources, you can simply delete the existing VPN gateway stack and create a new stack with the same parameters.  The remote side of the site-to-site VPN connection will automatically reconnect once the new VPN gateway has been established.

After deploying the new VPN gateway stack, you will need to ensure that any local routing table entries are updated to point to the new VPN gateway EC2 instance.

## Contributing

See [TODO.md](./TODO.md) for enhancement ideas.

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the contribution process.

File an issue in GitHub, ensure your changes pass `cfn-lint` tests and functionally work, before submitting a Pull Request (PR) for consideration.

## License

This project is licensed under the [Apache-2.0 License](./LICENSE).
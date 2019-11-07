# VPN Gateway Stack Using strongSwan

An [AWS CloudFormation](https://aws.amazon.com/cloudformation/) template that can be used to automate deployment of the open source [strongSwan VPN solution](https://www.strongswan.org/) as a VPN gateway in support of several different [site-to-site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html) topologies.  The open source [Quagga](https://en.wikipedia.org/wiki/Quagga_(software) ) software suite complements the role of strongSwan by providing [Border Gateway Protocol (BGP)](https://searchnetworking.techtarget.com/definition/BGP-Border-Gateway-Protocol) support to automatically propagate routing information across site-to-site VPN connections.

Even if you don’t have a need to demonstrate integration with AWS site-to-site VPN capabilities, you might find value in reviewing the Infrastructure as Code (IaC) techniques demonstrated by the example AWS CloudFormation template including its built-in integrations with other AWS services to support logging, resource monitoring, and secure remote terminal access.

* [Use Cases](#use-cases)
* [Integration with AWS Services](#cloudformation-features-demonstrated)
* [Usage](#usage)
* [Parameters](#parameters)
* [Troubleshooting](#troubleshooting)
* [Advanced Usage](#advanced-usage)
* [Contributing](#contributing)
* [License](#license)

## Use Cases

The example template can be useful for experimenting, testing, and demonstrating integration scenarios with the AWS Site-to-Site VPN feature and more formally implementing site-to-site VPN connections where the built-in AWS VPN services might not apply.

### Demonstration and Lab Environments

When you don’t have ready access to either real on-premises VPN hardware or software appliances, this example can be useful in demonstrating how to integrate an on-premises network with AWS networks via AWS site-to-site VPN connections and either AWS Virtual Private Gateways (VPGs) or AWS Transit Gateways (TGWs).

**Site-to-Site VPN with AWS Transit Gateway**

See [Transit Gateway Example: Centralized Router](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-centralized-router.html) for an overview of this topology.

<img src="images/site-to-site-vpn-scenarios-tgw.png" alt="Site-to-Site VPN with AWS Transit Gateway" width="800"/>

**Site-to-Site VPN with AWS Virtual Private Gateway**

See [AWS Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/SetUpVPNConnections.html) for details on this configuration.

<img src="images/site-to-site-vpn-scenarios-vgw.png" alt="Site-to-Site VPN with AWS Transit Gateway" width="800"/>

### Both Ends of a DIY Site-to-Site VPN Connection

The example template can also be used on both ends of a site-to-site VPN connection in scenarios where VPGs and TGWs are not applicable. Normally, you would use either VPC Peering or AWS Transit Gateway when you control the environments on both ends of a site-to-site VPN connection, but there may be circumstances in which you want to manage the VPN gateway on both ends.

<img src="images/site-to-site-vpn-scenarios-diy.png" alt="Site-to-Site VPN with AWS Transit Gateway" width="800"/>

## Integration with AWS Services

The example AWS CloudFormation template automatically builds a stack that demonstrates use of the following AWS services, features, and best practices:

* AWS CloudFormation features including the [`AWS::CloudFormation::Init`](https://docs.aws.amazon.com/en_pv/AWSCloudFormation/latest/UserGuide/aws-resource-init.html) feature to completely automate the build out of the VPN gateway stack and BGP support upon first boot and the [`AWS::CloudFormation::WaitCondition`](https://docs.aws.amazon.com/en_pv/AWSCloudFormation/latest/UserGuide/aws-properties-waitcondition.html) feature to force the stack creation process to wait until the first boot build out is complete.

*	[Amazon EC2](https://aws.amazon.com/ec2/) provides the compute platform in which to deploy the strongSwan VPN gateway.

*	[Amazon CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) integration via the [CloudWatch Logs Agent](https://docs.aws.amazon.com/en_pv/AmazonCloudWatch/latest/logs/CWL_GettingStarted.html) in which OS, VPN gateway, and BGP log files are written to a series of log streams in a CloudWatch Logs log group.

* [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/) integration for [monitoring EC2 memory and disk metrics](https://docs.aws.amazon.com/en_pv/AWSEC2/latest/UserGuide/mon-scripts.html).

* [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/en_pv/systems-manager/latest/userguide/session-manager.html) to enable secure terminal access to the OS instance without the need to establish Internet accessible bastion hosts and port 22 access to the VPN gateway.

* [Systems Manager Parameter Store](https://aws.amazon.com/blogs/compute/query-for-the-latest-amazon-linux-ami-ids-using-aws-systems-manager-parameter-store/) to query for latest Amazon Linux 2 Amazon Machine Image (AMI) images.

* Standardized naming of cloud resources to help distinguish from other resources, identify ownership, and potentially aid in access control.

## Usage

The following instructions are primarily oriented toward the first use case and two deployment topologies described above: Site-to-Site VPN with AWS Transit Gateway and Site-to-Site VPN with Virtual Private Gateway.  If you're interested in demonstrating a DIY solution for both ends of a site-to-site VPN connection, you should be able to easily extend these instructions.

### 1. Determine Deployment Location: Public or Private Subnet

Since VPN connections typically occur over the public Internet, you'll need to have at least one public IP address to represent the local side of the VPN tunnels.  You have several options to associate a public IP address. In either case, decoupling the creation and management of the public IP address from the creation and management of the VPN gateway enables you to replace the VPN gateway stack including the associated strongSwan EC2 instance without needing to reconfigure the remote end of the site-to-site VPN connection.

#### Option 1: Deploy VPN Gateway in Public Subnet and Use Elastic IP Address

Before deploying this stack, create an EIP and obtain its allocation ID so that you can pass it as a parameter to the CloudFormation stack through which the VPN gateway will be created.  When deploying this stack, you set the parameter `pUseElasticIp` to `true` and supply a value for the `pEipAllocationId` parameter.

#### Option 2: Deploy VPN gateway in Private Subnet and Use a NAT Gateway

In this case, you discover the public IP address of the NAT Gateway and use it when configuring the remote side of the VPN connection.  When deploying this stack, you accept the default `false` setting for the `pUseElasticIp` parameter. Since the local side of the site-to-site VPN initiates the connection, the local strongSwan VON gateway will initiate the connection through the NAT Gateways public IP address.

### 2. Determine VPN Tunnel Configuration Settings

When using either AWS VPGs or TGWs for the remote end of the site-to-site VPN connection, a site-to-site VPN connection resource will be established in AWS on the remote site. Within the site-to-site VPN connection resource on the remote site, you can download a VPN configuration file that will provide you with much of the data required to deploy the local VPN gateway. In the AWS management console, see `VPC -> Site-to-Site VPN Connections`, select the connection of interest, click `Download` and select the `Generic` option for `Vendor` and download the configuration file.

Review the data in this file in preparation for passing it as parameters to the CloudFormation stack in the next step.

### 3. Deploy VPN Gateway Stack

In this step you'll create a CloudFormation stack using the [`vpn-gateway-strongswan.yml`](./vpn-gateway-strongswan.yml) template and configuration data obtained from the remote site's Site-to-Site VPN Connection resource.

* Use the CloudFormation template to deploy a VPN gateway stack in a public subnet based on the [Parameters](#parameters) described below.
* Wait for creation of the stack to complete. Since the template uses a wait condition, the stack won't complete until strongSwan and other components have been configured and started.
* Wait for several minutes after stack creation completes. Then monitor the Site-to-Site VPN Connection on the remote site to confirm that the two VPN tunnels have progressed from the `DOWN` state to the `UP` state.  If the VPN gateway configuration is correct, Tunnel 1 will come up first followed several minutes later by Tunnel 2.

If the tunnels have not come up 3-5 minutes after creation of the VPN gateway stack completed, then see the Troubleshooting section below.

### 4. Ensure Routing Tables and EC2 Security Groups Are in Place

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
|`pOrg`|Optional|As an example of using resurce naming standards, include the business organization in the names of resources including, for example, IAM roles.|`acme`|
|`pSystem`|Optional|As an example of using resurce naming standards, include a system identifier in the names of resources including, for example, IAM roles..|`infra`|
|`pApp`|Optional|As an example of using resurce naming standards, include an application identifier in the names of resources including, for example, IAM roles.|`vpngw`|
|`pEnvPurpose`|Optional|As an example of using resurce naming standards, include a purpose for this particulart instance of the stack in the names of resources including, for example, IAM roles.. For example, "dev1", "test", "1", etc.|None|
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
|`pSubnetId`|Required|The subnet in which the VPN gateway is to be deployed.|None|
|`pUseElasticIp`|Optional|Use elastic IP address?|`false`|
|`pEipAllocationId`|Conditional|The allocation ID of the Elastic IP address that is to be associated with the VPN gateway. Required when `pUseElasticIP` = `true`|None|
|`pLocalBgpAsn`|Optional|The BGP Autonomous System Number (ASN) used to represent the local end of the site-to-site VPN connection.|`65000`|
|`pTunnel1BgpNeighborIpAddress`|Required|See the remote site's configuration.|None|
|**EC2 Instance**| | | |
|`pAmiId`|Optional|The ID of the AMI to use for the VPN gateway. By default this Systems Manager Parameter Store key is used to lookup the latest version of the referenced AMI for use in the current region.|`/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-ebs`|
|`pInstanceType`|Optional|The EC2 instance type to use for the VPN gateway.|`t3a.micro`|

## Troubleshooting

### VPN Gateway Stack Fails On Creation

Double check your parameter settings against both your local network configuration and the configuration of the site-to-site tunnels.  If you're using an Elastic IP address, ensure that the allocation ID is correct.

### Tunnels Don't Come Up

Ensure that you've waited 5 minutes or so to give the tunnels time to establish.

It's likely that one or more of the tunnel related stack parameters is incorrect. Double check the settings.  You can delete and recreate the VPN gateway stack without needing to delete and recreate the remote site's VPN resources.

You can also inspect the VPN gateway's logs via CloudWatch Logs.  In CloudWatch Logs, look for a log group that is named based on the system classification parameters described above. For example: `/infra/vpngw/ec2/...`. 

If any of the following log files are not present: `charon.log`, `zebra.log`, `bgpd.log`, start a terminal session with the gateway instance and use the `systemctl status <service name>` command to understand why a service did not start.

Log files in order of importance are:

* `cf-init.log` - Look for successful execution of the configuration sets from the `AWS::CloudFormation::Init` section of the CloudFormation template.
* `charon.log` - If initialization looks ok, check the content of this log file to monitor the establishment of the VPN tunnels.

### Can't Ping Across the VPN Connection

Verify correctness of the following configurations on both sides of the site-to-site VPN connection:
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

### Masking Source IP Addresses

If you'd like the VPC in which the strongSwan VPN gateway is running to forward traffice from the VPN connection to either other VPCs via VPC Peering or onward via gateways such an Internet Gateway to NAT Gateway, you'll need to configure the VPN gateway to mask the original source IP address by using the VPN gateway's IP address.

You can implement source network IP masking via an `iptables` command.  For example, the following command when run on the strongSwan VPN gatewy will mask the source IP address only for traffic whose destination IP address does not match the specified network. e.g. It will mask traffic destined for the Internet, but not for the local network. Presence of the `!` argument prior to the `-d` argument ensures that the all destinations other than the stated network will be subject to the masking rule.

```
$ sudo /sbin/iptables -t nat -A POSTROUTING -o eth0 ! -d 10.0.0.0/16 -j MASQUERADE
```
Alternatively, you can choose to mask all traffic:

```
$ sudo /sbin/iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Other useful `iptables` commands include:

```
# List the current rules:

# With packet counts:

$ sudo iptables -t nat -L -v

Chain PREROUTING (policy ACCEPT 12 packets, 1001 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 1 packets, 93 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 234 packets, 16318 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain POSTROUTING (policy ACCEPT 152 packets, 11245 bytes)
 pkts bytes target     prot opt in     out     source               destination
   93  5981 MASQUERADE  all  --  any    eth0    anywhere            !10.0.0.0/16

# With line numbers:

$ sudo iptables -t nat -L --line-numbers

Chain PREROUTING (policy ACCEPT)
num  target     prot opt source               destination

Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
num  target     prot opt source               destination
1    MASQUERADE  all  --  anywhere            !10.0.0.0/16

# Delete a specific rule based on line number:

$ sudo iptables -t nat -D POSTROUTING 1
```
See [TODO.md](./TODO.md) for an entry calling for the optional automation of source IP masking.

## Contributing

See [TODO.md](./TODO.md) for enhancement ideas.

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the contribution process.

File an issue in GitHub, ensure your changes pass `cfn-lint` tests and functionally work, before submitting a Pull Request (PR) for consideration.

## License

This project is licensed under the [Apache-2.0 License](./LICENSE).
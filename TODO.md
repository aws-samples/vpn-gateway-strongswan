# To Do Items

In order of importance:

* Validate that the gateway function properly after EC2 instance is restarted. e.g. that all settings are persistent and system services come up automatically upon OS restart.

* Provide a parameter to automatically configure source IP address translation for traffic leaving the strongSwan VPN gateway so that remotely sourced traffic can be routed beyond the local VPC via, for example, either an Internet Gateway or NAT Gateway. For example, when attempting to route remote traffic onward, the following commands can be used to perform source IP address translation so that the traffic emanating from the strongSwan instance looks like it is originating from that instance and will be routable outside the VPC.

```
# /sbin/iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

or

# iptables -t nat -A POSTROUTING -s 10.0.4.0/18 -o eth0 -j MASQUERADE
```

Another option is to insert a Transit Gateway to handle transitive routing without needing to perform source IP translation on the local VPN gateway.

* Consider enhancing cfn-init handling to support updates to the stack.

* Automated testing: An initial level of post stack creation automated testing.

* See if certificate-based VPN authentication is feasible with Strongswan.

* Provide same automation via [AWS Cloud Development Kit (CDK)](https://aws.amazon.com/cdk/).

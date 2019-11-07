# To Do Items

In order of importance:

* Validate that the gateway function properly after EC2 instance is restarted. e.g. that all settings are persistent and system services come up automatically upon OS restart.

* Provide a parameter to automatically configure source IP address translation for traffic leaving the strongSwan VPN gateway so that remotely sourced traffic can be routed beyond the local VPC via, for example, either an Internet Gateway or NAT Gateway. See the Advanced Usage section of the `README.md` for manual configuration instructions.

See https://fedoraproject.org/wiki/How_to_edit_iptables_rules for examples of how to persist iptables rules.

Automating and persisting this configuration will require installation of the following package and enablement of the `iptables` service:

```
$ sudo yum install iptables-services

$ sudo systemctl enable iptables.service
```

* Consider enhancing cfn-init handling to support updates to the stack.

* Automated testing: An initial level of post stack creation automated testing.

* See if certificate-based VPN authentication is feasible with Strongswan.

* Provide same automation via [AWS Cloud Development Kit (CDK)](https://aws.amazon.com/cdk/).

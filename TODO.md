# To Do Items

In order of importance:

* Validate that the gateway function properly after EC2 instance is restarted. e.g. that all settings are persistent and system services come up automatically upon OS restart.

* Consider enhancing cfn-init handling to support updates to the stack.

* Automated testing: An initial level of post stack creation automated testing.

* See if certificate-based VPN authentication is feasible with Strongswan.

* Provide same automation via [AWS Cloud Development Kit (CDK)](https://aws.amazon.com/cdk/).

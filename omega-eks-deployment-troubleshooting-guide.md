can yoiu explain to me about the brokers again and ycpi being a frontend. Are these brokers just DNS records forwarding to actual brokers in AWS? 

YCPI Routing Rules in Edgeman
Onboarding request

YCPI-6285: Onboarding request to proxy requests from on-prem to Amazon MSK
Done
 

Overview
Edgeman is the self-service UI where properties who are onboarded to YCPI can manage their routing rules. They update their own rules in the UI and wait for them to be deployed.

Since all traffic from on-prem flows through YCPI, we have rules in YCPI for managing those routes. Our rules can be found under the SRE_US property.

https://edgeman.ouryahoo.com/edgeman/showRevs/269/

Here’s a screenshot of our current rules on Nov 19, 2025. Note that these may change, but this should provide a good foundation for understanding the routing configuration.


Understanding the Routing Configuration
YCPI created 6 DNS records in the yahooinc domain: 1 for each Kafka broker in the dev cluster, and 1 for each Kafka broker in the prod cluster. 

broker[1,2,3]-public-<cluster name>.yahooinc.com

We use those records as the hosts config and use a “partial blind tunnel” to route the traffic to our actual brokers. Note the P: prefix on the tunnel_route. This is intentional and is used for the “partial” aspect of the blind tunnel route.

MSD/WAKL for YCPI IP Access
Overview
We use MSD (Micro Segmentation Daemon) and WAKL (Wide Area Key List) to dynamically manage IP prefix lists that allow YCPI (Yahoo Cloud Private Infrastructure) IP ranges to access the MSK cluster.

Although our MSD policies are managed by Terraform, you can locate the MSD policies in the Athenz UI under our domain and the Microsegmentation tab. Screenshot taken on Nov 19, 2025, this is just an example.


Architecture
Prefix Lists: AWS EC2 Managed Prefix Lists store YCPI IP ranges

Security Groups: Reference prefix lists for ingress rules

WAKL Lambda: Updates prefix lists hourly with current YCPI IPs

MSD Policies: Define transport policies for network access

Components
Prefix Lists
Count: 2 prefix lists (to handle large IP ranges)

Max Entries: 250 IPs per prefix list

Naming: msk-access-prefix-list-{0,1}

Security Groups
Count: 2 security groups (one per prefix list)

Naming: msk-access-security-group-{0,1}

Port: 9196 (MSK public access port)

Protocol: TCP

WAKL Module
Terraform Module: terraform-modules__yahoo/wakl/aws

Configuration:

Frequency: rate(1 hour) - Updates every hour

Athenz Domain: sre.tooling.{environment}

Athenz Service: bpi-msk-access

OTEL Endpoint: otel.dev.ui:4443

Prefix List IDs: Comma-separated list of region:prefix-list-id

Function:

Fetches YCPI IP ranges from MSD

Updates AWS EC2 Managed Prefix Lists

Prefix lists are used in the Security Groups that the Kafka brokers run in for allowing access to the public SCRAM port on 9096

Runs as Lambda function on schedule

MSD Transport Policies
Module: terraform-modules__yahoo/msd-policy/athenz

Policy Configuration:



transport_policies = [
  {
    identifier = "bpi-msk-access-{env}-allow-from-ycpi"
    direction  = "ingress"
    remote_services = [
      "ycpi.ycpi-networks-access"
    ]
    destination_ports = ["9196"]
    source_ports      = ["1024-65535"]
    protocol          = "TCP"
    conditions = [
      {
        hosts             = "*"
        scope             = ["aws"]
        enforcement_state = "enforce"
      }
    ]
  }
]
How It Works
WAKL Lambda runs hourly:

Queries MSD for YCPI IP ranges

Updates AWS EC2 Managed Prefix Lists

Prefix lists automatically update security groups

Security Groups attached to MSK cluster:

Allow ingress on port 9196

Source: Prefix lists (dynamically updated)

MSD Policies enforce network segmentation:

Define allowed traffic patterns

Enforced at network level

Monitoring
WAKL Lambda: CloudWatch Logs for execution status

Prefix Lists: AWS Console shows current IP entries

Security Groups: Ingress rules show prefix list references

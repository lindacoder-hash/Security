IPsec protocol suite designed to secure IP communications by authenticating/encrpyting each IP packet in a data stream.
Used for VPN over untrusted networks like the internet.

## 1. On-prem client shares public IP of their Linux server, let me say 20.187.198.32

## 2. Create a customer gateway with the IP above with routing as static

```python
import boto3

ec2 = boto3.client('ec2')

customer_gateway = ec2.create_customer_gateway(
    BgpAsn=65000,
    PublicIp='20.187.198.32',
    Type='ipsec.1',
    TagSpecifications=[
        {
            'ResourceType': 'customer-gateway',
            'Tags': [{'Key': 'Name', 'Value': 'OnPrem-CGW'}]
        }
    ]
)
customer_gateway_id = customer_gateway['CustomerGateway']['CustomerGatewayId']
```

## 3. Create a virtual private gateway and give it a name tag

```python
vgw = ec2.create_vpn_gateway(
    Type='ipsec.1',
    TagSpecifications=[
        {
            'ResourceType': 'vpn-gateway',
            'Tags': [{'Key': 'Name', 'Value': 'My-VGW'}]
        }
    ]
)
vgw_id = vgw['VpnGateway']['VpnGatewayId']
```

## 4. Attach the virtual private gateway to a VPC

```python
vpc_id = 'vpc-xxxxxxxxxxxxxxxxx'  # replace with your actual VPC ID

ec2.attach_vpn_gateway(
    VpnGatewayId=vgw_id,
    VpcId=vpc_id
)
```

## 5. Go to Site-to-Site and create a VPN connection

    - Choose the VGW created in step 4
    - Choose CGW in step 2
    - Routing should be static
    - Prefixes, add 2 , the CIDR block from customer's side and yours (192.31.0.0/16 and  10.0.0.1/16)
    -  Local IPv4 network should be customer's cidr block (192.31.0.0/16 )
    - Remote IPv4 network CIDR (10.0.0.1/16)

```python
vpc_id = 'vpc-xxxxxxxxxxxxxxxxx'  # replace with your actual VPC ID

ec2.attach_vpn_gateway(
    VpnGatewayId=vgw_id,
    VpcId=vpc_id
)

vpn_connection = ec2.create_vpn_connection(
    Type='ipsec.1',
    CustomerGatewayId=customer_gateway_id,
    VpnGatewayId=vgw_id,
    Options={
        'StaticRoutesOnly': True
    },
    TagSpecifications=[
        {
            'ResourceType': 'vpn-connection',
            'Tags': [{'Key': 'Name', 'Value': 'OnPrem-VPN'}]
        }
    ]
)
vpn_connection_id = vpn_connection['VpnConnection']['VpnConnectionId']

# Add customer's CIDR
ec2.create_vpn_connection_route(
    DestinationCidrBlock='192.31.0.0/16',
    VpnConnectionId=vpn_connection_id
)

# Add your AWS CIDR block
ec2.create_vpn_connection_route(
    DestinationCidrBlock='10.0.0.1/16',
    VpnConnectionId=vpn_connection_id
)
```

## 6. Enable Route Propagation to the VGW

```python
# Replace with your actual Route Table ID and VGW ID
route_table_id = 'rtb-xxxxxxxxxxxxxxxxx'  # Main route table ID
vgw_id = vgw_id  # From Step 2

ec2.enable_vgw_route_propagation(
    RouteTableId=route_table_id,
    GatewayId=vgw_id
)
```

## 7: Download the VPN Configuration File

- Go to **VPC Dashboard > Site-to-Site VPN Connections**
- Click your **VPN Connection ID**
- Click the **“Download Configuration”** button at the top
- Choose your vendor (Openswan) or generic **“Generic”**, which looks like file below.

```txt

=======================================================================
                         IPSEC Tunnel #1
=======================================================================

This configuration assumes that you already have a default Openswan or Libreswan installation in place on your Linux server.

1) Open `/etc/sysctl.conf` and ensure these values exist:

    net.ipv4.ip_forward = 1
    net.ipv4.conf.default.rp_filter = 0
    net.ipv4.conf.default.accept_source_route = 0

2) Apply the changes by running:

    sysctl -p

3) Edit `/etc/ipsec.conf` and ensure this line is uncommented:

    #include /etc/ipsec.d/*.conf

4) Create and edit this file `/etc/ipsec.d/aws.conf`, append the config:

    conn Tunnel1
        authby=secret
        auto=start
        left=%defaultroute
        leftid=20.187.198.32                  # Your public IP
        right=AWS_TUNNEL_ENDPOINT             # Replace with actual AWS tunnel IP
        type=tunnel
        ikelifetime=8h
        keylife=1h
        phase2alg=aes128-sha1;modp1024
        ike=aes128-sha1;modp1024
        auth=esp  #remove this line
        keyingtries=%forever
        keyexchange=ike
        leftsubnet=192.31.0.0/16              # Customer CIDR block
        rightsubnet=10.0.0.0/16               # AWS VPC CIDR block
        dpddelay=10
        dpdtimeout=30
        dpdaction=restart_by_peer

5) Create this file `/etc/ipsec.d/aws.secrets` and append the shared secret:

    20.187.198.32 AWS_TUNNEL_ENDPOINT : PSK "YourSharedSecretHere"

```

Our customer follows the instructions in the file above.

## 8 Check if Tunnel 1 is up using the AWS CLI (awscli)

`aws ec2 describe-vpn-connections --vpn-connection-ids vpn-xxxxxxxxxxxxxxxxx --query "VpnConnections[0].VgwTelemetry[0]"`

```bash
{
    "OutsideIpAddress": "34.201.89.22",
    "Status": "UP",
    "LastStatusChange": "2025-06-30T17:48:29+00:00",
    "StatusMessage": "IPSEC IS UP",
    "AcceptedRouteCount": 1,s
    "TunnelInsideIpAddress": "169.254.10.6"
}

```

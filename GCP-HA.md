# 🌐 HA VPN

**HA VPN** is a high-availability (HA) Cloud VPN solution that lets you securely connect your on-premises network to your VPC network through an IPsec VPN connection.

Based on the topology and configuration, HA VPN can provide an SLA of **99.99% or 99.9% service availability**.

HA VPN uses an **external VPN gateway** resource in Google Cloud to provide information to Google Cloud about your **peer VPN gateway or gateways**.

When you create an HA VPN gateway, Google Cloud **automatically chooses two external IP addresses**, one for each of its interfaces.
Each IP address is automatically chosen from a unique address pool to support **high availability**.
Each HA VPN gateway interface supports **multiple tunnels**.

---

## 🧭 HA VPN over Cloud Interconnect

One option for using HA VPN is to use it **over Cloud Interconnect**.

With this setup:

- You get the **security of IPsec encryption** from Cloud VPN.
- You benefit from the **increased capacity** of Cloud Interconnect.
- Your traffic **does not go over the public internet**.

If you're using **Partner Interconnect**, you must add **IPsec encryption** to your Cloud Interconnect traffic to meet **data security and compliance** requirements when connecting to **third-party providers**.

---

## 🔗 Interconnect Options

```txt
🔗 1. Dedicated Interconnect
🧾 What it is:
You get physical fiber connections directly from Google at Google’s colocation facilities.

✅ You need your own router in a supported colocation facility and a network engineer/team to manage BGP, hardware, and failover

🤝 2. Partner Interconnect
🧾 What it is:
Google works with a third-party network provider to help you connect to Google Cloud without owning or placing equipment in Google colocation facilities.

✅ You need a local provider from Google’s Partner list (e.g., Equinix, Megaport), less infrastructure; setup is more flexible and just set up a VLAN attachment from your location to Google
```

---

## 🧵 VLAN Attachment

**GCP VLAN attachment** connects your **on-prem router** (via Partner or Dedicated Interconnect) to your **VPC** using a **VLAN ID**.

---

## 🔁 Dynamic Border Gateway Protocol (BGP) Routing

**Dynamic Border Gateway Protocol (BGP) routing** is a protocol used to **automatically exchange routing information** between different networks—especially across large, complex, or multi-provider environments like the internet or hybrid cloud setups.

---

### 🛠️ How BGP Works in Cloud VPN (e.g., AWS/GCP)

Let’s say you're using a Site-to-Site VPN between:

- Your on-premises router (**AS 65000**)
- And Google Cloud or AWS (**AS 16550 or 7224**)

Instead of manually defining which CIDRs are reachable:

✅ You enable BGP, and both ends automatically exchange:
🔧 Automatic failover (e.g., VPN tunnel 1 goes down, tunnel 2 takes over)ssssssssssssssssssss

---

### 🔁 Redundancy with HA VPN

If **two peer devices** are required, each peer device must be connected to a **different HA VPN gateway interface**.

If the peer side is another cloud provider like AWS, **VPN connections must be configured with adequate redundancy** on the AWS side as well.

Your **peer VPN gateway device must support dynamic BGP routing**.

---

![alt text](image.png)

# HANDS-ON

# ✅ **GCP CLI — Step-by-step**

---

### **1. Create Cloud Router with ASN 65000 and advertise all subnets**

```bash
gcloud compute routers create my-cloud-router \
  --network=my-vpc \
  --region=europe-west1 \
  --asn=65000 \
  --advertise-mode=DEFAULT
```

> Replace `my-vpc` and `europe-west1` with your actual VPC name and region.

---

### **2. Create HA VPN Gateway**

```bash
gcloud compute vpn-gateways create my-ha-vpn-gateway \
  --network=my-vpc \
  --region=europe-west1 \
  --stack-type=IPV4_ONLY
```

---

### **3. Get External IPs (interface0, interface1)**

```bash
gcloud compute vpn-gateways describe my-ha-vpn-gateway \
  --region=europe-west1 \
  --format="get(vpnInterfaces)"
```

You’ll see external IPs like:

- Interface 0: `35.242.105.234`
- Interface 1: `34.156.200.546`

---

# ✅ **AWS CLI — Step-by-step**

---

### **4. Create 2 Customer Gateways (for the 2 IPs from GCP)**

```bash
aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 35.242.105.234 \
  --bgp-asn 65000 \
  --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=GCP-CGW1}]'

aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 34.156.200.546 \
  --bgp-asn 65000 \
  --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=GCP-CGW2}]'
```

---

### **5. Create a Virtual Private Gateway and attach to VPC with ASN 65001**

```bash
aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --amazon-side-asn 65001 \
  --tag-specifications 'ResourceType=vpn-gateway,Tags=[{Key=Name,Value=AWS-VGW}]'

aws ec2 attach-vpn-gateway \
  --vpn-gateway-id vgw-xxxxxxxx \
  --vpc-id vpc-xxxxxxxx
```

Enable route propagation (via console or route table association).

```bash
aws ec2 enable-vgw-route-propagation \
    --route-table-id ROUTE_TABLE_ID \
    --gateway-id VPN_GATEWAY_ID
```

---

### **6. Create 2 Site-to-Site VPN Connections (Dynamic Routing)**

```bash
aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-xxxxxxxx \
  --vpn-gateway-id vgw-xxxxxxxx \
  --options '{"TunnelInsideIpVersion": "ipv4", "StaticRoutesOnly": false}' \
  --tag-specifications 'ResourceType=vpn-connection,Tags=[{Key=Name,Value=VPN-to-GCP-CGW1}]'

aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id cgw-yyyyyyyy \
  --vpn-gateway-id vgw-xxxxxxxx \
  --options '{"TunnelInsideIpVersion": "ipv4", "StaticRoutesOnly": false}' \
  --tag-specifications 'ResourceType=vpn-connection,Tags=[{Key=Name,Value=VPN-to-GCP-CGW2}]'
```

Download 2 configuration files of 2 vpn connection created above with settings below

- Vendor: **Generic**
- Platform: **Vendor Agnostic**
- Software: **IKEv1**

Looks like these below

```txt

=======================================================================
                         IPSEC Tunnel #1
=======================================================================
NOTE: If you customized tunnel options when creating or modifying your VPN connection,

Higher parameters are only available for VPNs of category "VPN," and not for "VPN-Classic".
The address of the external interface for your customer gateway must be a static address.
Your customer gateway may reside behind a device performing network address translation (NAT).
To ensure that NAT traversal (NAT-T) can function, you must adjust your firewall rules!
If not behind NAT, and you are not using an Accelerated VPN, we recommend disabling NAT.

  - IKE version                : IKEv1
  - Authentication Method      : Pre-Shared Key
  - Pre-Shared Key             : XwEc.Iwsf_Ue6HLkkZuKRrPyyyyyyyyyyyyyyy5JOsoUbIF
  - Authentication Algorithm   : sha1
  - Encryption Algorithm       : aes-128-cbc
  - Lifetime                   : 28800 seconds
  - Phase 1 Negotiation Mode   : main
  - Diffie-Hellman             : Group 2

#2: IPSec Configuration

Configure the IPSec SA as follows:
Category "VPN" connections in the GovCloud region have a minimum requirement of AES128.
Please note, you may use these additionally supported IPSec parameters for encryption.

NOTE: If you customized tunnel options when creating or modifying your VPN connection,

Higher parameters are only available for VPNs of category "VPN," and not for "VPN-Classic".

  - Protocol                   : esp
  - Authentication Algorithm   : hmac-sha1-96
  - Encryption Algorithm       : aes-128-cbc
  - Lifetime                   : 3600 seconds
  - Mode                       : tunnel
  - Perfect Forward Secrecy    : Diffie-Hellman Group 2



#3: Tunnel Interface Configuration

Your Customer Gateway must be configured with a tunnel interface that is
associated with the IPSec tunnel. All traffic transmitted to the tunnel
interface is encrypted and transmitted to the Virtual Private Gateway.

The Customer Gateway and Virtual Private Gateway each have two addresses that relate
to this IPSec tunnel. Each contains an outside address, upon which encrypted
traffic is exchanged. Each also contain an inside address associated with
the tunnel interface.

The Customer Gateway outside IP address was provided when the Customer Gateway
was created. Changing the IP address requires the creation of a new
Customer Gateway.

The Customer Gateway inside IP address should be configured on your tunnel
interface.

Outside IP Addresses:
  - Customer Gateway         : 34.157.97.82
  - Virtual Private Gateway  : 52.1.197.46

Inside IP Addresses:
  - Customer Gateway         : 169.254.232.42/30
  - Virtual Private Gateway  : 169.254.232.41/30

Configure your tunnel to fragment at the optimal size:
  - Tunnel interface MTU     : 1436 bytes



=======================================================================
                         IPSEC Tunnel #2
=======================================================================

NOTE: If you customized tunnel options when creating or modifying your VPN connection,

Higher parameters are only available for VPNs of category "VPN," and not for "VPN-Classic".
The address of the external interface for your customer gateway must be a static address.
Your customer gateway may reside behind a device performing network address translation (NAT).
To ensure that NAT traversal (NAT-T) can function, you must adjust your firewall rules!
If not behind NAT, and you are not using an Accelerated VPN, we recommend disabling NAT.

  - IKE version                : IKEv1
  - Authentication Method      : Pre-Shared Key
  - Pre-Shared Key             : XwEc.Iwsf_Ue6HLkkZuKRhhhhhhhhhhhhhhrP5JOsoUbIF
  - Authentication Algorithm   : sha1
  - Encryption Algorithm       : aes-128-cbc
  - Lifetime                   : 28800 seconds
  - Phase 1 Negotiation Mode   : main
  - Diffie-Hellman             : Group 2

#2: IPSec Configuration

Configure the IPSec SA as follows:
Category "VPN" connections in the GovCloud region have a minimum requirement of AES128.
Please note, you may use these additionally supported IPSec parameters for encryption.

NOTE: If you customized tunnel options when creating or modifying your VPN connection,

Higher parameters are only available for VPNs of category "VPN," and not for "VPN-Classic".

  - Protocol                   : esp
  - Authentication Algorithm   : hmac-sha1-96
  - Encryption Algorithm       : aes-128-cbc
  - Lifetime                   : 3600 seconds
  - Mode                       : tunnel
  - Perfect Forward Secrecy    : Diffie-Hellman Group 2


#3: Tunnel Interface Configuration

Your Customer Gateway must be configured with a tunnel interface that is
associated with the IPSec tunnel. All traffic transmitted to the tunnel
interface is encrypted and transmitted to the Virtual Private Gateway.

The Customer Gateway and Virtual Private Gateway each have two addresses that relate
to this IPSec tunnel. Each contains an outside address, upon which encrypted
traffic is exchanged. Each also contain an inside address associated with
the tunnel interface.

The Customer Gateway outside IP address was provided when the Customer Gateway
was created. Changing the IP address requires the creation of a new
Customer Gateway.

The Customer Gateway inside IP address should be configured on your tunnel
interface.

Outside IP Addresses:
  - Customer Gateway         : 39.157.97.82
  - Virtual Private Gateway  : 53.1.197.46

Inside IP Addresses:
  - Customer Gateway         : 163.254.232.42/30
  - Virtual Private Gateway  : 161.254.232.41/30

Configure your tunnel to fragment at the optimal size:
  - Tunnel interface MTU     : 1436 bytes


```

---

# ✅ **Back to GCP — Configure Peer VPN Gateway**

---

### **7. Create Peer VPN Gateway (with 4 interfaces)**

Assuming you downloaded the AWS VPN config and extracted IPs from there:
Copy the outside virtual private gateway IP
```txt
=======================================================================
                         IPSEC Tunnel #1
=======================================================================
Outside IP Addresses:
  - Virtual Private Gateway  : 52.1.197.46



=======================================================================
                         IPSEC Tunnel #2
=======================================================================
Outside IP Addresses:
  - Virtual Private Gateway  : 53.1.197.46

```

```bash


  gcloud compute external-vpn-gateways create aws-peer-gateway  \
    --interfaces 0=52.1.197.46,1= 53.1.197.46, \
        2=<OUTSIDE Virtual Private Gateway VALUE IN 2ND CONF FILE FOR TUNNEL 1>, \
        3=<OUTSIDE Virtual Private Gateway VALUE IN 2ND CONF FILE FOR TUNNEL 2>
```

Update the IPs from your AWS config file if different.

---

### **8. Create 4 VPN tunnels, link them to Cloud Router and match PSKs**

Do this for each interface pair:

```bash
gcloud compute vpn-tunnels create tunnel-1 \
  --peer-external-gateway=aws-peer-gateway \
  --peer-external-gateway-interface=0 \
  --region=europe-west1 \
  --router=my-cloud-router \
  --ike-version=1 \
  --shared-secret="XwEc.Iwsf_Ue6HLkkZuKRrPyyyyyyyyyyyyyyy5JOsoUbIF" \
  --vpn-gateway=my-ha-vpn-gateway \
  --vpn-gateway-interface=0

gcloud compute vpn-tunnels create tunnel-2 \
  --peer-external-gateway=aws-peer-gateway \
  --peer-external-gateway-interface=1 \
  --region=europe-west1 \
  --router=my-cloud-router \
  --ike-version=1 \
  --shared-secret="XwEc.Iwsf_Ue6HLkkZuKRhhhhhhhhhhhhhhrP5JOsoUbIF" \
  --vpn-gateway=my-ha-vpn-gateway \
  --vpn-gateway-interface=0

# Tunnel 3
gcloud compute vpn-tunnels create tunnel-3 \
  --peer-external-gateway=aws-peer-gateway \
  --peer-external-gateway-interface=2 \
  --region=europe-west1 \
  --router=my-cloud-router \
  --ike-version=1 \
  --shared-secret="psk-tunnel-3" \
  --vpn-gateway=my-ha-vpn-gateway \
  --vpn-gateway-interface=1

# Tunnel 4
gcloud compute vpn-tunnels create tunnel-4 \
  --peer-external-gateway=aws-peer-gateway \
  --peer-external-gateway-interface=3 \
  --region=europe-west1 \
  --router=my-cloud-router \
  --ike-version=1 \
  --shared-secret="psk-tunnel-4" \
  --vpn-gateway=my-ha-vpn-gateway \
  --vpn-gateway-interface=1
```

---

### **9. Configure BGP Sessions (on Cloud Router)**

The `ip-address` and `peer-ip-address` is the inside customer gateway and virtual private gateway IP respectively

```txt
Inside IP Addresses:
  - Customer Gateway         : 169.254.232.42/30
  - Virtual Private Gateway  : 169.254.232.41/30

```

For each tunnel:

```bash
# BGP for Tunnel 1
gcloud compute routers add-bgp-peer my-cloud-router \
  --region=europe-west1 \
  --interface=tunnel-1 \
  --peer-name=aws-peer-1 \
  --peer-asn=65001 \
  --advertised-route-priority=100 \
  --peer-ip-address=169.254.232.41 \
  --ip-address=169.254.232.42

# BGP for Tunnel 2
gcloud compute routers add-bgp-peer my-cloud-router \
  --region=europe-west1 \
  --interface=tunnel-2 \
  --peer-name=aws-peer-2 \
  --peer-asn=65001 \
  --advertised-route-priority=100 \
  --peer-ip-address=169.254.11.2 \
  --ip-address=169.254.11.1

# BGP for Tunnel 3
gcloud compute routers add-bgp-peer my-cloud-router \
  --region=europe-west1 \
  --interface=tunnel-3 \
  --peer-name=aws-peer-3 \
  --peer-asn=65001 \
  --advertised-route-priority=100 \
  --peer-ip-address=169.254.12.2 \
  --ip-address=169.254.12.1

# BGP for Tunnel 4
gcloud compute routers add-bgp-peer my-cloud-router \
  --region=europe-west1 \
  --interface=tunnel-4 \
  --peer-name=aws-peer-4 \
  --peer-asn=65001 \
  --advertised-route-priority=100 \
  --peer-ip-address=169.254.13.2 \
  --ip-address=169.254.13.1

```

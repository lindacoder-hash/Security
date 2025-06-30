# Security

- I helped the DevOps team in migrating from our opaque secrets set-up to External Secrets Operator (ESO),s which supports storing or
  retrieving secret data from external secret management systems such as AWS Secrets Manager or Parameter store.
- Deployed Kubernetes master nodes/worker nodes on private subnets.
- Control traffic to subnets using network ACLs to block access to ports, including 6443 for the Kubernetes API Server and TCP 10250 for
  Kubelet API etc.
- Setting up a virtual private network with OpenVPN to encrypt internet connections for staff, securing their connections to the company’s
  internal network.
- Setting up AWS Site-to-Site VPN to enable our VPC to securely connect with our client’s on-premise network.
- Implemented Transport Layer Security for ingress.
- Established OpenID Connect (OIDC) and RBAC configurations.
- Configured Service Accounts with appropriate IAM roles/policies for different applications.
- Rolled out a company-wide implementation of Passbolt (a source password manager) on our EC2 instances, enabling both technical and
  non-technical teams to monitor and manage credentials.
- Introduced Trivy, a vulnerability scanner for containers, to our Gitlab CI/CD pipelines. This step assists in identifying issues before our
  images are deployed to Amazon ECR.

# EKS CDK Quick Start (in Python)

> **DEVELOPER PREVIEW NOTE:** This project is currently available as a preview and should not be considered for production use at this time. 


This Quick Start is a reference architecture and implementation of how you can use the Cloud Development Kit (CDK) to orchestrate the Elastic Kubernetes Service (EKS) to quickly deploy a more complete and "production ready" Kubernetes environment on AWS.

## What does this Quick Start create for you:

1. An appropriate VPC with public and private subnets across three availability zones.
    1. This defaults to a /22 CIDR w/1024 IPs by default - though this can be adjusted in `cluster-bootstrap/cdk.json` with the parameters `vpc_cidr`, `vpc_cidr_mask_public` and `vpc_cidr_mask_private`.
    1. Alternatively, just flip `create_new_vpc` to `False` and then specify the name of your VPC under `existing_vpc_name` in `cluster-bootstrap/cdk.json` to use an existing VPC. CDK will automatically work out which subnets are public and which are private and deploy to the private ones.
        1. Note that if you do this you'll also have to tag your subnets as per https://aws.amazon.com/premiumsupport/knowledge-center/eks-vpc-subnet-discovery/
1. A new EKS cluster with:
    1. A dedicated new IAM role only used to create it and own it. 
        1. This is important as the role that creates the cluster is a permanent, and rather hidden, full admin role that doesn't appear in nor is subject to the `aws-auth` ConfigMap. So you want to treat it like the AWS Root Account for this cluster and only use it if required (e.g. to fix the aws-auth ConfigMap should it become corrupted).
    1. A new dedicated role to Administer it (which *IS* via the `aws-auth` ConfigMap) as per https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html
        1. Alternatively, you can specify an existing role ARN to make the administrator by flipping to `False` in `create_new_cluster_admin_role` and then putting the arn to use in `existing_admin_role_arn` in `cluster-bootstrap/cdk.json`.
            1. NOTE that if you bring an existing role that this role will get assigned to the Bastion by default as well (so that the Bastion can manage the cluster). This means that you need to:
                1. Allow ec2.amazonaws.com to sts:AssumeRole this role (your Bastion)
                1. Add the Managed Policy `AmazonSSMManagedInstanceCore` to this role (so that your Bastion can register with SSM via this role)
    1. A new Managed Node Group with 2 x m5.large instances spread across 3 Availability Zones.
        1. You can change the instance type and quantity by changing `eks_node_quantity` and/or `eks_node_instance_type` in `cluster-bootstrap/eks-cluster.py`.
        1. You can also have it be via Spot rather than OnDemand by setting `eks_node_spot` to True.
    3. All control plane logging to CloudWatch Logs enabled (defaulting to 1 month's retention within CloudWatch Logs).
1. Ingress and AWS Load Balancer integration
    1. The AWS Load Balancer Controller (https://kubernetes-sigs.github.io/aws-load-balancer-controller) to allow you to seamlessly use ALBs for Ingress and NLB for Services.
1. External DNS (https://github.com/kubernetes-sigs/external-dns) to allow you to automatically create/update Route53 entries to point your 'real' names at your Ingresses and Services.
1. Observability
    1. CloudWatch Container Insights - Metrics
    1. CloudWatch Container Insights - Logs
    1. (Optional) AWS managed ElasticSearch - Logs 
        1. If you flip `deploy_managed_elasticsearch` to True this creates a new managed Amazon Elasticsearch Domain behind a private VPC endpoint as well as a fluent-bit DaemonSet to ship all your container logs there - including enriching them with the Kubernetes metadata using the kubernetes fluent-bit filter.
        1. Note that this provisions a single node 10GB managed Elasticsearch Domain suitable for a proof of concept. To use this in production you'll likely need to edit the `es_capacity` section of `cluster-bootstrap/cdk.json` to scale this out from a capacity and availability perspective. For more information see https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/sizing-domains.html.
        1. Note that this provisions an Elasticsearch and Kibana that does not have a login/password configured. It is secured instead by network access controlled by it being in a private subnet and its security group. While this is acceptable for the creation of a Proof of Concept (POC) environment, for production use you'd want to consider implementing Cognito to control user access to Kibana - https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/fgac.html#fgac-walkthrough-iam
    1. (Optional) (Temporarily until the AWS Managed Prometheus/Grafana are available) a self-managed Prometheus and Grafana - Metrics
        1. If you flip `deploy_kube_prometheus_operator` to True this will deploy the kube-prometheus Operator (https://github.com/prometheus-operator/kube-prometheus) which deploys you a Prometheus on your cluster that will collect all your cluster metrics as well as a Grafana to visualize them.
        1. You can adjust the disk size of these in `cluster-bootstrap/cdk.json` with `prometheus_disk_size`, `alertmanager_disk_size` and `grafana_disk_size`. 
        1. AMP and AMG are not yet available in all regions nor are they supported by CloudFormation and the CDK yet - once they are we'll add that to this Quick Start and deprecate this limited self-managed option in lieu of that.
1. CSI Drivers
    1. The AWS EBS CSI Driver (https://github.com/kubernetes-sigs/aws-ebs-csi-driver). Note that new development on EBS functionality has moved out of the Kubernetes mainline to this externalized CSI driver.
    1. The AWS EFS CSI Driver (https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html). Note that new development on EFS functionality has moved out of the Kubernetes mainline to this externalized CSI driver.
1. Security and Governance
    1. (Optional) An OPA Gatekeeper to enforce preventative security and operational policies (https://github.com/open-policy-agent/gatekeeper). A set of example policies is provided as well - see `gatekeeper-policies/README.md`
    1. (Optional) The Calico Network Policy Provider (https://docs.aws.amazon.com/eks/latest/userguide/calico.html). This enforces any [NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) that you specify.
1. Autoscaling
    1. The cluster autoscaler (CA) (https://github.com/kubernetes/autoscaler). This will scale your EC2 instances to ensure you have enough capacity to launch all of your Pods as they are deployed/scaled.
    1. The metrics-server (required for the Horizontal Pod Autoscaler (HPA)) (https://github.com/kubernetes-sigs/metrics-server)
1. The AWS Systems Manager (SSM) agent. This allows for various management activities (e.g. Inventory, Patching, Session Manager, etc.) of your Instances/Nodes by AWS Systems Manager.

The items listed as (Optional) above are not enabled by default - but all of the add-ons are really optional and you control whether you'll get them with a parameter for each in  `cluster-bootstrap/cdk.json` that you flip to True/False.

### Why Cloud Development Kit (CDK)?

The Cloud Development Kit (CDK) is a tool where you can write infrastructure-as-code with 'actual' code (TypeScript, Python, C#, and Java). This takes these languages and 'compiles' them into a CloudFormation template for the AWS CloudFormation engine to then deploy and manage as stacks.

When you develop and deploy infrastructure with the CDK you don't edit the intermediate CloudFormation but, instead, let CDK regenerate it in response to changes in the upstream CDK code.

What makes CDK uniquely good when it comes to our EKS Quickstart is:

* It handles the IAM Roles for Service Accounts (IRSA) rather elegantly and creates the IAM Roles and Policies, creates the Kubernetes service accounts, and then maps them to each other.
* It has implemented custom CloudFormation resources with Lambda invoking kubectl and helm to deploy manifests and charts as part of the cluster provisioning.
    * Until we have [Managed Add-Ons](https://aws.amazon.com/blogs/containers/introducing-amazon-eks-add-ons/) for the common things with EKS like the above this can fill the gap and provision us a complete cluster with all the add-ons we need.

## Getting started

You can either deploy this from your machine or leverage CodeBuild. The advantage of using CodeBuild is it also sets up a 'GitOps' approach where when you merge changes to
the cluster-bootstrap folder it'll (re)run `cdk deploy` for you. This means that to update the cluster you just change this file and merge. 

###  Deploy from CodeBuild
To use the CodeBuild CloudFormation Template:

1. Generate a personal access token on GitHub - https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token 
1. Edit `cluster-codebuild/EKSCodeBuildStack.template.json` to change Location to your GitHub repo/path
1. Run `aws codebuild import-source-credentials --server-type GITHUB --auth-type PERSONAL_ACCESS_TOKEN --token <token_value>` to provide your token to CodeBuild
1. Deploy `cluster-codebuild/EKSCodeBuildStack.template.json`
1. Go to the CodeBuild console, click on the Build project that starts with `EKSCodeBuild`, and then click the Start build button.
1. (Optional) You can click the Tail logs button to follow along with the build process

**_NOTE:_** This also enables a GitOps pattern where changes to the cluster-bootstrap folder on the branch mentioned (main by default) will re-trigger this CodeBuild to do another `cdk deploy` via web hook.

### Deploy from your laptop

Alternatively, you can deploy from any machine (your laptop, a bastion EC2 instance, etc.).

There are some prerequisites you likely will need to install on the machine doing your environment bootstrapping including Node, Python, the AWS CLI, the CDK, fluxctl and Helm

#### Pre-requisites - Ubuntu 20.04.2 LTS (including via Windows 10's WSL)

Run `sudo ./ubuntu-prereqs.sh`

#### Pre-requisites - Mac

1. Install Homebrew (https://brew.sh/)
1. Run `./mac-prereqs.sh`

#### Deploy from CDK locally

1. Make sure that you have your AWS CLI configured with administrative access to the AWS account in question (e.g. an `aws s3 ls` works)
    1. This can be via setting your access key and secret in your .aws folder via `aws configure` or in your environment variables by copy and pasting from AWS SSO etc.
1. Run `cd quickstart-eks-cdk-python/cluster-bootstrap`
2. Run `sudo npm install --upgrade -g aws-cdk` to ensure your CDK is up to date
3. Run `pip3 install --upgrade -r requirements.txt` to install the required Python bits of the CDK
4. Run `export CDK_DEPLOY_REGION=ap-southeast-2` replacing ap-southeast-2 with your region of choice
5. Run `export CDK_DEPLOY_ACCOUNT=123456789123` replacing 123456789123 with your AWS account number
6. (Optional) If you want to make an existing IAM User or Role the cluster admin rather than creating a new one then edit `cluster-bootstrap/cdk.json` and comment out the current cluster_admin_role and uncomment the one beneath it and fill in the ARN of the User/Role you'd like there.
7. (Only required the first time you use the CDK in this account) Run `cdk bootstrap` to create the S3 bucket where it puts the CDK puts its artifacts
8. (Only required the first time Elasticsearch in VPC mode is used in this account) Run `aws iam create-service-linked-role --aws-service-name es.amazonaws.com`
9. Run `cdk deploy --require-approval never`

### (Optional) Deploy Open Policy Agent (OPA) Gatekeeper and the policies via Flux v2

The `gatekeeper` folder contains the manifests to deploy [Gatekeeper's Helm Chart](https://github.com/open-policy-agent/gatekeeper/tree/master/charts/gatekeeper) as well as an example set of policies to help achieve a much better security and noisy neighbor situation - especially if doing multi-tenancy. See more about these example policies in `gatekeeper/README.md`.

This also serves as an example of how to use Flux v2 with EKS.

In order to deploy Gatekeeper and the example policies:
1. Ensure you are on a system where `kubectl` is installed and working against the cluster (like the bastion)
1. [Install the Flux v2 CLI](https://fluxcd.io/docs/installation/#install-the-flux-cli)
1. Run `flux install` to install Flux onto the cluster
1. Change directory into the root of quickstart-eks-cdk-python
1. Run `kubectl apply -f gatekeeper/gatekeeper-sync.yaml` to install the Gatekeeper Helm Chart w/Flux (as well as enable future GitOps if the main branch of the repo is updated)
1. Run `flux get all` to see the progress of getting and installing the Gatekeeper Helm Chart
1. Run `flux create source git gatekeeper --url=https://github.com/aws-quickstart/quickstart-eks-cdk-python --branch=main` to add this repo to Flux as a source
    1. Alternatively, and perhaps advisably, specify the URL of your git repo you've forked/cloned the projet to instead - as it will trigger GitOps actios going forward when this changes!
1. Run `kubectl apply -f gatekeeper/policies/policies-sync.yaml` to install the policies with Flux (as well as enable future GitOps if the main branch of the repo is updated)
1. Run `flux get all` to see all of the Flux items and their reconciliation statuses

If you want to change any of the Gatekeeper Constraints or ConstraitTemplates you just change the YAML and then push/merge it to the repo and branch you indicated above and Flux will them deploy those changes to your cluster via GitOps.

## Deploy and set up a Bastion based on an EC2 instance accessed securely via Systems Manager's Session Manager

If you set `deploy_bastion` to `True` in `cluster-bootstrap/cdk.json` then the template will deploy an EC2 instance with all the tools to manage your cluster.

To access this bastion:
1. Go to the Systems Manager Server in the AWS Console
1. Go to Managed Instances on the left hand navigation pane
1. Select the instance with the name `EKSClusterStack/CodeServerInstance`
1. Under the Instance Actions menu on the upper right choose Start Session
1. You need to run `sudo bash` to get to root's profile where we've set up kubectl
1. Run `kubectl get nodes` to see that all the tools are there and set up for you.


## (Optional) Set up your Client VPN to access the environment

If you set `deploy_vpn` to `True` in `cluster-bootstrap/cdk.json` then the template will deploy a Client VPN so that you can securely access the cluster's private VPC subnets from any machine. You'll need this to be able to reach the Kibana for your logs and Grafana for your metrics by default (unless you are using an existing VPC where you have already arranged such connectivity)

Note that you'll also need to create client and server certificates and upload them to ACM by following these instructions - https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/client-authentication.html#mutual - and update `ekscluster.py` with the certificate ARNs for this to work.

Once it has created your VPN you then need to configure the client:

1. Open the AWS VPC Console and go to the Client VPN Endpoints on the left panel
1. Click the Download Client Configuration button
1. Edit the downloaded file and add:
    1. A section at the bottom for the server cert in between `<cert>` and `</cert>`
    1. Then under that another section for the client private key between `<key>` and `</key>` under that
1. Install the AWS Client VPN Client - https://aws.amazon.com/vpn/client-vpn-download/
1. Create a new profile pointing it at that configuration file
1. Connect to the VPN

Once you are connected it is a split tunnel - meaning only the addresses in your EKS VPC will get routed through the VPN tunnel.

You then need to add the EKS cluster to your local kubeconfig by running the command in the clusterConfigCommand Output of the EKSClusterStack.

Then you should be able to run a `kubectl get all -A` and see everything running on your cluster.

## (Optional) How access to Elasticsearch and Kibana if you choose to deploy them

We put the Elasticsearch both in the VPC (i.e. not on the Internet) as well as in its own Security Group - which will give access by default only from our EKS cluster's SG (so that can ship the logs to it) as well as to from our (optional) Client VPN's Security Group to allow us access Kibana when on VPN.

Since this ElasticSearch can only be reached if you are both within the private VPC network *and* allowed by this Security Group, then it is low risk to allow 'open access' to it - especially in a Proof of Concept (POC) environment. As such, we've configured its default access policy so that no login and password and are required - choosing to control access to it from a network perspective instead.

For production use, though, you'd likely want to consider implementing Cognito to facilitate authentication/authorization for user access to Kibana - https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/fgac.html#fgac-walkthrough-iam

### Connect to Kibana and do initial setup

1. Once that new access policy has applied click on the Kibana link on the Elasticsearch Domain's Overview Tab
1. Click "Explore on my own" in the Welcome page
1. Click "Connect to your Elasticsearch index" under "Use Elasticsearch Data"
1. Close the About index patterns box
1. Click the Create Index Pattern button
1. In the Index pattern name box enter `fluent-bit*` and click Next step
1. Pick @timestamp from the dropdown box and click Create index pattern
1. Then go back Home and click Discover

TODO: Walk through how to do a few basic things in Kibana with searching and dashboarding your logs.

## (Optional) How to access Prometheus and Grafana if you choose to deploy them

We have deployed an in-VPC private Network Load Balancer (NLB) to access your Grafana service to visualize the metrics from the Prometheus we've deployed onto the cluster.

To access this enter the following command `get service grafana-nlb --namespace=kube-system` to find the address of this under EXTERNAL-IP. Alternatively, you can find the Grafana NLB in the AWS EC2 console and get its address from there.

Once you go to that page the default login/password is admin/prom-operator.

There are some default dashboards that ship with this which you can see by going to Home on top. This will take you to a list view of the available dashboards. Some good ones to check out include:

- Kubernetes / Compute Resources / Cluster
    - This gives you a whole cluster view
- Kubernetes / Compute Resources / Namespace (Pods)
    - There is a namespace dropdown at the top and it'll show you the graphs including the consumption in that namespace broken down by Pod
- Kubernetes / Compute Resources / Namespace (Workloads)
    - Similar to the Pod view but instead focuses on Deployment, StatefulSet and DaemonSet views

Within all of these dashboards you can click on names as links and it'll drill down to show you details relevant to that item.

## Deploy some sample/demo apps to explore our new Kubernetes environment and its features

Various demo application are in the demo-apps folder showing how to use the various features and add-ons installed by this Quick Start. There is a [README](https://github.com/aws-quickstart/quickstart-eks-cdk-python/blob/main/demo-apps/README.md) in that folder with more information about them and how to install them.

## Upgrading your cluster

Since we are explicit both with the EKS Control Plane version as well as the Managed Node Group AMI version upgrading these is simply incrementing these versions, saving `cluster-bootstrap/cdk.json` and then running a `cdk deploy`.

As per the [EKS Upgrade Instructions](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html) you start by upgrading the control plane, then any required add-on versions and then the worker nodes.

Upgrade the control plane by changing `eks_version` in  `cluster-bootstrap/cdk.json`. You can see what to put there by looking at the [CDK documentation for KubernetesVersion](https://docs.aws.amazon.com/cdk/api/latest/python/aws_cdk.aws_eks/KubernetesVersion.html). Then run `cdk deploy` - or let the CodeBuild GitOps provided in `cluster-codebuild` do it for you.

Upgrade the worker nodes by updating `eks_node_ami_version` in  `cluster-bootstrap/cdk.json` with the new version. You find the version to type there in the [EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/eks-linux-ami-versions.html) as shown here:
![](eks_ami_version.PNG)

## Upgrading an add-on

Each of our add-ons are deployed via Helm Charts and are explicit about the chart version being deployed. In the comment above each chart version we link to the GitHub repo for that chart where you can see what the current chart version is and can see what changes may have been rolled in since the one cited in the template.

To upgrade the chart version update the chart version to the upstream version you see there, save it and then do a `cdk deploy`.

**NOTE:** While we were thinking about parameterizing the chart versions within `cluster-bootstrap/cdk.json`, it is possible as the Chart versions change that the values you have to specify might also change. As such, we have not done so as a reminder that this change might require a bit of research and testing rather than just popping a new version number parameter in and expecting it'll work.

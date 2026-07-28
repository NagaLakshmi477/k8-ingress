# ============================================================
# AWS LOAD BALANCER CONTROLLER SETUP ON EKS
# ============================================================
#
# Why are we doing this?
#
# Our Kubernetes application may need to create AWS resources
# such as:
#   - Application Load Balancer (ALB)
#   - Network Load Balancer (NLB)
#
# Kubernetes itself does not automatically have permission to
# create AWS resources.
#
# Therefore, we need to connect:
#
# Kubernetes ServiceAccount
#          ↓
# IAM Role
#          ↓
# IAM Policy (Permissions)
#          ↓
# AWS Services
#
# This allows a Kubernetes application/controller to securely
# access AWS services without storing AWS Access Keys inside pods.
# ============================================================


# ============================================================
# STEP 1: SET ENVIRONMENT VARIABLES
# ============================================================

# AWS Region where our EKS cluster is running
REGION_CODE=us-east-1

# Name of our EKS cluster
CLUSTER_NAME=roboshop

# AWS Account ID
ACC_ID=352742379477


# ============================================================
# STEP 2: ENABLE OIDC IDENTITY PROVIDER FOR EKS
# ============================================================

# OIDC = OpenID Connect
#
# OIDC allows AWS IAM to trust identities coming from
# Kubernetes ServiceAccounts in our EKS cluster.
#
# In simple words:
#
# Kubernetes ServiceAccount
#          ↓
# OIDC Provider
#          ↓
# AWS IAM Role
#          ↓
# AWS Permissions
#
# Without OIDC, AWS IAM cannot identify and trust a
# Kubernetes ServiceAccount using IAM Roles for Service Accounts
# (IRSA).

eksctl utils associate-iam-oidc-provider \
  --region $REGION_CODE \
  --cluster $CLUSTER_NAME \
  --approve


# ============================================================
# STEP 3: DOWNLOAD AWS LOAD BALANCER CONTROLLER IAM POLICY
# ============================================================

# The AWS Load Balancer Controller needs permission to communicate
# with AWS services.
#
# For example, it may need permissions to:
#   - Create Load Balancers
#   - Modify Load Balancers
#   - Create Target Groups
#   - Register targets
#   - Manage security groups
#
# The required permissions are defined in iam-policy.json.
#
# curl downloads the IAM policy file from the official
# AWS Load Balancer Controller GitHub repository.

curl -o iam-policy.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v3.4.2/docs/install/iam_policy.json


# ============================================================
# STEP 4: CREATE IAM POLICY IN AWS
# ============================================================

# An IAM Policy is a collection of permissions.
#
# Here, we create an AWS IAM Policy named:
# AWSLoadBalancerControllerIAMPolicy
#
# The permissions are read from the iam-policy.json file.
#
# Important:
# Policy = What actions are allowed?
#
# Example:
#   Create a Load Balancer
#   Modify a Load Balancer
#   Describe EC2 resources
#
# At this point, we are ONLY creating the policy.
# We have not yet connected it to Kubernetes.

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json


# ============================================================
# STEP 5: CREATE IAM ROLE + KUBERNETES SERVICEACCOUNT
# ============================================================

# Now we connect Kubernetes with AWS IAM.
#
# This command creates:
#
# 1. An AWS IAM Role
# 2. A Kubernetes ServiceAccount
# 3. A trust relationship between them
# 4. Attaches the IAM Policy to the IAM Role
#
# The flow will be:
#
# AWSLoadBalancerController Pod
#            ↓
# Kubernetes ServiceAccount
# aws-load-balancer-controller
#            ↓
# IAM Role
#            ↓
# IAM Policy
#            ↓
# AWS Permissions
#
# This is called IRSA:
# IAM Roles for Service Accounts
#
# This allows the AWS Load Balancer Controller pod to
# access AWS APIs using IAM permissions.

eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::$ACC_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region=$REGION_CODE \
  --approve


# ============================================================
# WHY DO WE NEED ALL OF THIS?
# ============================================================
#
# IAM:
# IAM = Identity and Access Management
#
# IAM controls:
#   - WHO can access AWS resources
#   - WHAT they can do
#   - WHICH AWS resources they can access
#
# IAM contains:
#   - Users
#   - Groups
#   - Roles
#   - Policies
#
#
# KUBERNETES:
#
# Kubernetes has its own security and authorization system
# called RBAC.
#
# RBAC = Role-Based Access Control
#
# Kubernetes RBAC controls what users and Kubernetes
# ServiceAccounts can do INSIDE the Kubernetes cluster.
#
# Example:
#
# Kubernetes RBAC can allow a ServiceAccount to:
#   - Get Pods
#   - Create Services
#   - Watch Ingress resources
#
# But Kubernetes RBAC does NOT automatically give permission
# to AWS services.
#
#
# ============================================================
# THE PROBLEM
# ============================================================
#
# Suppose we create an Ingress in Kubernetes:
#
# Kubernetes Ingress
#        ↓
# AWS Load Balancer Controller
#        ↓
# Needs to create an AWS Application Load Balancer
#
# The controller is running inside Kubernetes,
# but the Load Balancer is an AWS resource.
#
# Therefore, the controller needs AWS IAM permissions.
#
#
# ============================================================
# THE SOLUTION
# ============================================================
#
# We integrate Kubernetes with AWS IAM using:
#
# Kubernetes ServiceAccount
#            ↓
#          OIDC
#            ↓
#       IAM Role (IRSA)
#            ↓
#      IAM Policy
#            ↓
#      AWS Permissions
#
# This allows the AWS Load Balancer Controller running
# inside Kubernetes to securely call AWS APIs.
#
# We do NOT need to store AWS Access Key and Secret Key
# inside the Kubernetes Pod.
#
# This is more secure and is the recommended approach.
# ============================================================


# ============================================================
# STEP 6: ADD AWS EKS HELM REPOSITORY
# ============================================================

# Helm is a package manager for Kubernetes.
#
# Helm Charts are pre-packaged Kubernetes application
# definitions.
#
# Here, we are adding the AWS EKS Helm repository.
#
# "eks" is the local name we give to the repository.
#
# This repository contains the AWS Load Balancer Controller
# Helm Chart.

helm repo add eks https://aws.github.io/eks-charts


# ============================================================
# STEP 7: INSTALL AWS LOAD BALANCER CONTROLLER
# ============================================================

# Now we install the AWS Load Balancer Controller
# into the kube-system namespace.
#
# The controller will run as a Pod inside our EKS cluster.
#
# Important settings:
#
# --set clusterName=$CLUSTER_NAME
#     Tells the controller which EKS cluster it belongs to.
#
# --set serviceAccount.create=false
#     We are telling Helm NOT to create a new ServiceAccount.
#
#     Why?
#     Because we already created the ServiceAccount using
#     eksctl in STEP 5.
#
# --set serviceAccount.name=aws-load-balancer-controller
#     Tells the controller to use the existing ServiceAccount.
#
# The final relationship is:
#
# AWS Load Balancer Controller Pod
#             ↓
# ServiceAccount:
# aws-load-balancer-controller
#             ↓
# IAM Role
#             ↓
# IAM Policy
#             ↓
# AWS APIs
#
# Now the controller can create and manage AWS Load Balancers
# based on Kubernetes resources such as Ingress.

helm install aws-load-balancer-controller \
  eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
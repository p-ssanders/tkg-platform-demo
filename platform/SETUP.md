#   Platform Setup

##  Prepare to deploy management clusters

1.  Create an access key
1.  `aws configure --profile tkg-demo`
1.  `export AWS_PROFILE=tkg-demo`
1.  `aws ec2 create-key-pair --key-name tkg-demo --output json | jq .KeyMaterial -r > tkg-demo.pem`

([docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/1.6/vmware-tanzu-kubernetes-grid-16/GUID-mgmt-clusters-aws.html))

##  Deploy a management cluster

    $ tanzu management-cluster create --ui

##  Create a Workload Cluster

    $ tanzu cluster create demo-acceptance-workload --file platform/demo-acceptance-workload.yaml

##  Install Cert-Manager

    $ tanzu package install cert-manager --package-name cert-manager.tanzu.vmware.com --version 1.7.2+vmware.1-tkg.1
    $ k apply -f platform/cert-manager/aws-credentials-secret.yaml

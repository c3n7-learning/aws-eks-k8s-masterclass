# Section 2: EKS - Create Cluster using `eksctl`

## 04. Introduction

There are 3 CLIs we can use to manage our AWS EKS clusters, we need three CLIs:

1. AWS CLI: to control multiple AWS services via the command line and automate them through scripts
2. `kubectl`: to control k8s clusters and objects using this tool
3. `eksctl`: is used for creating and deleting clusters on AWS EKS. We can create, autoscale and delete node groups. We can even create fargate profiles using eksctl. In short, it is a very powerful too for managing EKS cluster on AWS.

## 05. Install AWS CLI

We will follow [this guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). For MacOS:

1. Download the package file:

```shell
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
```

2. Run the standard macOS installer program, specifying the downloaded `.pkg` file as the source. Use the `-pkg` parameter to specify the name of the package to install, and the `-target /` parameter for which drive to install the package to. The files are installed to `/usr/local/aws-cli`, and a symlink is automatically created in `/usr/local/bin`. You must include `sudo` on the command to grant write permissions to those folders.

```shell
sudo installer -pkg ./AWSCLIV2.pkg -target /
```

3. Verify that the CLI was installed successfully:

```shell
$ which aws
/usr/local/bin/aws

$ aws --version
aws-cli/2.27.41 Python/3.11.6 Darwin/23.3.0
```

### Configure AWS-CLI using Security Credentials

Open AWS Management Console:

1. Click on your account (top-right)
2. Click on `security credentials`
3. Click on `create access key`
4. Use case: `Command Line Interface`. Check the confirmation
5. Description tag value: `my-cli-token`
6. Copy _access key_ and _secret access key_.

Now configure the `aws` cli:

```shell
$ aws configure
> Access Key: my-access-key
> Secret Access Key: my-secret-access-key
> Default region name: us-east-1
> Default output format: json
```

Verify if we have successfully configured it:

```shell
aws ec2 describe-vpcs
```

We'll get a response like:

```shell
{
    "Vpcs": [
        {
            "OwnerId": "662513131574",
            "InstanceTenancy": "default",
            "CidrBlockAssociationSet": [
                {
                    "AssociationId": "vpc-cidr-assoc-04477871503a68692",
                    "CidrBlock": "172.31.0.0/16",
                    "CidrBlockState": {
                        "State": "associated"
                    }
                }
            ],
            "IsDefault": true,
            "BlockPublicAccessStates": {
                "InternetGatewayBlockMode": "off"
            },
            "VpcId": "vpc-0d120dce03766407b",
            "State": "available",
            "CidrBlock": "172.31.0.0/16",
            "DhcpOptionsId": "dopt-0ecc0c9738514ef9e"
        }
    ]
}
```

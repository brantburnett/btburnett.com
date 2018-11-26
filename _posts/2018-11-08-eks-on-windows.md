---
id: 308
title: Using Amazon Elastic Container Service for Kubernetes (EKS) on Windows 10
date: 2018-11-08T09:45:00-05:00
categories:
  - Kubernetes
  - AWS
---
At [CenterEdge Software](https://centeredgesoftware.com/), we currently operate our [Kubernetes](https://kubernetes.io/) clusters on [AWS](https://aws.amazon.com/). We manage the clusters ourselves, using the [kops](https://github.com/kubernetes/kops) tool. Unfortunately, managing your own Kubernetes cluster adds a lot of overhead.

Therefore, I recently embarked on a proof of concept using [Amazon Elastic Container Service for Kubernetes](https://aws.amazon.com/eks/), a.k.a. EKS. I quickly found that a significant friction point in this process was my Windows 10 laptop, which is a problem since CenterEdge Software is a Microsoft shop.

Below I share some of the steps I found that helped along the way. I won't cover setting up the EKS cluster itself, I'll let the [AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html) handle that.

## Prerequisites

1. Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and the [AWS CLI](https://aws.amazon.com/cli/). I used [Chocolatey](https://chocolatey.org/) to install both.

    ```powershell
    choco install -y kubernetes-cli
    choco install -y awscli
    ```

2. Configure AWS CLI with your credentials

    ```powershell
    aws configure
    ```

## Setting Up AWS IAM Authenticator

When using EKS, kubectl must be configured to use the [https://github.com/kubernetes-sigs/aws-iam-authenticator](AWS IAM Authenticator). This lightweight utility is called by kubectl to get authentication tokens, and uses your credentials configured for the AWS CLI. It can support IAM roles and multiple profiles, but for this example I'll keep it simple and assume we're using the default profile configured via `aws configure`.

1. Download the authenticator. The current URL for Windows is https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/windows/amd64/aws-iam-authenticator.exe, but you may want to find the up-to-date version [here](https://docs.aws.amazon.com/eks/latest/userguide/configure-kubectl.html).
2. Place aws-iam-authenticator.exe somewhere in your system path. For example, I was lazy and put it in C:\ProgramData\Chocolatey\bin.
3. Right-click on aws-iam-authenticator.exe, select Properties, and Unblock the file so it can be executed.
4. Confirm that the command is working from a new shell window:

    ```powershell
    aws-iam-authenticator --help
    ```

## Adding Your Cluster To Your Kubernetes Config

The easiest way to add your cluster to your Kubernetes configuration is using the AWS CLI. It's also possible to keep multiple configuration files, but I prefer having multiple contexts inside my default configuration file.

```powershell
# Substitute "brant" below with the name of your EKS cluster
aws eks update-kubeconfig --name brant
```

However, after this is complete I recommend changing the name of the created context to be more usable. The first parameter below is the name of the context output by the update-kubeconfig command.  The second is the new name.

```powershell
kubectl config rename-context arn:aws:eks:us-east-1:000000000000:cluster/brant brant
```

Finally, test it out!

```powershell
kubectl get svc
```

## Working From Ubuntu on Windows using WSL

Unfortunately, many tools you may wish to use are Linux tools and don't work well from Windows. An easy solution is to [install Ubuntu on Windows 10](https://www.microsoft.com/en-us/p/ubuntu/9nblggh4msv6?activetab=pivot:overviewtab). However, making your previous configuration for EKS work in Ubuntu requires a few more steps.

1. Install [kubectl]https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-binary-using-native-package-management) in Ubuntu.
2. Add a KUBECONFIG environment variable to your Windows user profile (alter the path below if needed):

    ```powershell
    setx KUBECONFIG ${env:USERPROFILE}\.kube\config
    ```
3. Configure WSL to pass KUBECONFIG into Ubuntu, [while remapping the path](https://blogs.msdn.microsoft.com/commandline/2017/12/22/share-environment-vars-between-wsl-and-windows/):

    ```powershell
    # WSLENV is a colon-separated list of environment variables to copy to Ubuntu from your Windows Profile
    # Appending "/p" to the variable name tells WSL that the variable is a path, and to remap the path to the Ubuntu path when it's copied
    setx WSLENV $($(if ([System.String]::IsNullOrWhitespace(${env:WSLENV})) { "" } else { ${env:WSLENV} + ":"}) + "KUBECONFIG/p")
    ```

4. Restart Ubuntu
5. Test it out. Kubectl will run in Ubuntu, which in turn executes the Windows aws-iam-authenticator.exe process to get the authentication token.

    ```sh
    kubectl get svc
    ```

## Conclusion

At this point, you should have complete access to your EKS cluster via kubectl from both Powershell and Ubuntu Bash. Now the real fun can begin!

---
id: 241
title: Docker Login For Amazon AWS ECR Using Windows Powershell
date: 2017-01-02T15:41:41+00:00
guid: http://btburnett.com/?p=241
permalink: /2017/01/docker-login-for-amazon-aws-ecr-using-windows-powershell.html
categories:
  - Docker
  - .NET Core
---
My recent studies in .Net Core have lead me to the new world of Docker (new for .Net developers, anyway).  The idea of developing low-cost microservices while still working using  my favorite development platform is very exciting.  In the process, I began using the Amazon AWS Docker platform, Elastic Container Services (ECS).

I quickly found that documentation for using ECS from Windows is a bit scarce.  In particular, the Powershell tools are missing a pretty key helper method, get-login.  Calling "aws ecr get-login" on a Linux box delivers you a complete "docker login" command for authenticating to the Elastic Container Registry (ECR).  There is currently no such helper for Windows.  At least, not that I can find, someone correct me if I'm just missing  it.

Instead, I've done a bit of digging and found [how to authenticate programatically](https://aws.amazon.com/blogs/compute/authenticating-amazon-ecr-repositories-for-docker-cli-with-credential-helper/).  From that, I've created the helper code below for reuse.

```powershell
# Get the authorization token
$token = Get-ECRAuthorizationToken -Region us-east-1 -AccessKey your-access-key -SecretKey your-secret-key
# Split the token into username and password segments
$tokenSegments = [System.Text.Encoding]::ASCII.GetString([System.Convert]::FromBase64String($token.AuthorizationToken)).Split(":")
# Get the host name without https, as this can confuse some Windows machines
$hostName = (New-Object System.Uri $token.ProxyEndpoint).DnsSafeHost
# Perform login
docker login -u $($tokenSegments[0]) -p $($tokenSegments[1]) -e none $hostName
```

This login should then be valid for 12 hours. Note that if you use your own Region, AccessKey, and SecretKey on the first line. Alternatively, you could use Set-DefaultAWSRegion and Set-AWSCredentials to store them in your Powershell session. If you're on a build server running in AWS you could also use IAM roles to grant access directly to the build server.

## Update 2/8/2017

You can add the section below to your PowerShell profile to add an easy to use cmdlet. To install:

  1. Run "notepad $PROFILE"
  2. Paste the code below into the file and save
  3. Run ". $PROFILE" or restart Powershell
  4. Run "Auth-ECR your-access-key your-secretkey".

```powershell
function Auth-ECR {
    [CmdletBinding()]
    param (
        [parameter(Mandatory=$true, Position=0)]
        [string]
        $AccessKey,

        [parameter(Mandatory=$true, Position=1)]
        [string]
        $SecretKey,

        [parameter()]
        [string]
        $Region = "us-east-1"
    )

    # Get the authorization token
    $token = Get-ECRAuthorizationToken -AccessKey $AccessKey -SecretKey $SecretKey -Region $Region `
        -ErrorAction Stop

    # Split the token into username and password segments
    $tokenSegments = [System.Text.Encoding]::ASCII.GetString([System.Convert]::FromBase64String($token.AuthorizationToken)).Split(":")

    # Get the host name without https, as this can confuse some Windows machines
    $hostName = (New-Object System.Uri $token.ProxyEndpoint).DnsSafeHost

    # Perform login
    docker login -u $($tokenSegments[0]) -p $($tokenSegments[1]) -e none $hostName
}
```

Note that this script defaults to using the us-east-1 region. You can change the default in your profile, or use "-Region" on the command line.

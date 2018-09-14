---
id: 306
title: Using Halyard To Deploy Spinnaker From Windows
date: 2018-09-14T15:25:00-04:00
categories:
  - Spinnaker
---
Those of us who work from Windows laptops sometimes don't the the same love when it comes to tooling that Linux and macOS users receive. The [Windows Subsystem for Linux](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux) has certainly helped a lot, but it isn't perfect. A case in point is [Halyard](https://www.spinnaker.io/reference/halyard/), the Linux tool designed to manage and deploy [Spinnaker](https://www.spinnaker.io/). While it could probably be run on WSL, I personally chose a different approach because of the need to use Kubernetes credentials from my Windows profile folder. Also, I just don't like installing things on my computer when Docker can serve instead.

## Prerequisites

1. [Docker for Windows](https://docs.docker.com/docker-for-windows/)
2. [Share your drives with Docker](https://docs.docker.com/docker-for-windows/#shared-drives)

## Register the Module

To make the use of Halyard more transparent, I've created a PowerShell module which makes the use of Halyard from Windows largely transparent.  The [https://www.powershellgallery.com/packages/SpinnakerHalyard](SpinnakerHalyard) module is available for installation from the PowerShell Gallery.

```powershell
# Install the module, you just need to do this once
Install-Module SpinnakerHalyard -Scope CurrentUser

# Import the module into your Powershell session
Import-Module SpinnakerHalyard
```

## Startup Halyard

Halyard functions using a background service that must be running, and the "hal" command communicates with that service. To that end, the next step is to startup Halyard.

There are some useful tools missing from the image, such as Vim, so these will be installed during the startup process.

```text
PS > Start-Halyard
Get:1 http://security.debian.org/debian-security stretch/updates InRelease [94.3 kB]
Ign:2 http://deb.debian.org/debian stretch InRelease
Get:3 http://deb.debian.org/debian stretch-updates InRelease [91.0 kB]
Get:4 http://deb.debian.org/debian stretch Release [118 kB]
Get:5 http://security.debian.org/debian-security stretch/updates/main amd64 Packages [414 kB]
Lots more fun stuff I'm skipping here...

Id                                                               Name
--                                                               ----
3e1dac1690f5e905641b8f4c691fe5b37d360d630674928ac0e51e0b011a0725 halyard
```

This will start the Halyard service and leave it ready to execute commands.  Optionally, the `-Pull` argument can be used to ensure that Docker has downloaded the latest version of Halyard, or `-Version` can select a specific version from [the Docker registry](https://console.cloud.google.com/gcr/images/spinnaker-marketplace/GLOBAL/halyard).

## Run Commands!

At this point, `hal` commands can be run freely, including copying and pasting from examples.

```text
PS> hal config version edit --version 1.9.3
+ Get current deployment
  Success
+ Edit Spinnaker version
  Success
+ Spinnaker has been configured to update/install version "1.9.3".
  Deploy this version of Spinnaker with `hal deploy apply`.

PS> hal deploy apply
```

## But What About Editing Files?

Running `hal` commands works great for most basic configuration, but a lot of more advanced configuration involves manually editing configuration files.

```text
PS > Connect-Halyard
spinnaker@85057be1061c:~/.hal$ cd default/profiles
spinnaker@85057be1061c:~/.hal/default/profiles$ vim gate-local.yml
spinnaker@85057be1061c:~/.hal/default/profiles$ exit
exit
```

The `Connect-Halyard` command will open a bash prompt within the Halyard container, allowing access to files with Vim.

Want to script commands inside the container? `Connect-Halyard` has that power, too! Non-zero exit codes will be converted to PowerShell exceptions.

```text
PS > Connect-Halyard mv default/service-settings/gate.yml default/service-settings/gate.bak
```

## Backing Up Halyard Configuration

Halyard configuration is persisted using a Docker volume named `halyard`. However, there is always the risk that the volume may be lost. It's therefore advisable to regularly backup the Halyard configuration.

Normally, backups are performed using `hal backup create`. However, when working using the PowerShell module the `Backup-Halyard` command is recommended.

```text
PS > Backup-Halyard

- Create backup

+ Create backup

  Success
+ Successfully created a backup at location:
/home/spinnaker/halbackup-Fri_Sep_14_19-21-56_UTC_2018.tar
```

Backups will be found in the `.halbackups` folder of your user profile. They may be restored later using the `Restore-Halyard` command.

```text
PS > Restore-Halyard ${HOME}\.halbackups\halbackup-Fri_Sep_07_14-13-34_UTC_2018.tar
```

## Stopping The Service

When not in use, the Halyard service can be shutdown.

```text
PS > Stop-Halyard
```

## Conclusion

As an added bonus for users like me who prefer Docker images to installing software, this should work for Linux/macOS users as well. Just install PowerShell Core!

Also, check out the [https://github.com/brantburnett/halyard-powershell](SpinnakerHalyard Git repo). Enjoy Halyarding your Spinnaker!

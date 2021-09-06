---
id: 313
title: Reclaiming HD Space from Docker Desktop on WSL 2
date: 2020-12-11T06:00:00-05:00
permalink: /docker/2021/09/06/reclaiming-hd-space-from-docker-desktop-on-wsl-2.html
categories:
  - Docker
  - Windows
summary: Over time, Docker Desktop running on WSL 2 may start to eat away at your hard drive space. Here is how I was able to free up over 100GB.
---

Over time, Docker Desktop running on [Windows Subsystem for Linux 2 (a.k.a. WSL 2)](https://docs.docker.com/desktop/windows/wsl/) may start to eat away
at your hard drive space. Here is how I was able to free up over 100GB.

## Freeing up space within Docker

The first problem is freeing up the space being used by Docker internally. This may include container images, containers, dangling image layers,
volumes, and even networks. There are several different command which will help clean these up.

It isn't necessarily required to perform all of these steps, you should edit based on your particular needs. I will say that container images are
the biggest culprit in most cases. Also, in general it is safe to delete container images, as they may be downloaded again as needed.

Since we're talking primarily about Windows, these examples are in Powershell. However, the general concept should work from Bash as well.

```powershell
{% raw %}
# Delete all containers (if acceptable). This allows the images they are referencing to be deleted,
# but is not required. The first command ensures that all containers are stopped.
docker ps -q | % { docker stop $_ }
docker ps -aq | % { docker rm $_ }

# Delete all container images. This may return a few errors but that is usually fine.
docker images --format "{{.Repository}}:{{.Tag}}" | % { docker rmi $_ }
{% endraw %}
```

Once this is done, further commands will do more cleanup:

```powershell
# Cleans up several different items.
docker system prune

# Optionally, this format will also delete your Docker storage volumes. Beware of data loss.
docker system prune --volumes
```

If you want to delete some volumes, but leave others in place, I have used scripts like this one in the past.

```powershell
# Delete all volumes with names that do not contain "halyard" or "vscode".
docker volume ls -q | where { -not ($_ -match "halyard" -or $_ -match "vscode") } | % { docker volume rm $_ }
```

## Reclaiming the space for Windows

After freeing space within Docker, the space will (unfortunately) not be freed for Windows to use. The
problem is that Docker keeps its data within a VHDX virtual drive file. This file now has lots of free space within
it, but it needs to be shrunk.

Adding to the problem is that Windows Hyper-V isn't great about recognizing unused sectors within a VHDX file using
the EXT4 Linux file system. I won't pretend to be an expert, but my understanding is that we need to convince Linux
to send TRIM commands to the hardware abstraction layer to indicate unused sectors. Once this is done, Hyper-V can
more successfully shrink the VHDX file.

```powershell
# Run this as an Administrator, not all of these steps work as a regular Windows user.

# Perform the TRIM commands and then delete the image we just used.
docker run --rm --privileged --pid=host docker/desktop-reclaim-space
docker rmi docker/desktop-reclaim-space

# Shutdown Docker and WSL 2. You may wish to exit Docker Desktop gracefully before this step.
# If you get file in use errors on the next step, not performing this step is the likely culprit.
wsl --shutdown

# Optimize the VHDX file to reclaim free space. This may take a few minutes.
Optimize-VHD -Path ${env:LOCALAPPDATA}\Docker\wsl\data\ext4.vhdx -Mode Full
```

### Side note

If you're running antivirus, be sure you've excluded VHDX files from real time scans for better performance
within Docker. See [Recommended antivirus exclusions for Hyper-V hosts](https://docs.microsoft.com/en-us/troubleshoot/windows-server/virtualization/antivirus-exclusions-for-hyper-v-hosts) for details.

## Conclusion

In my case, this took my VHDX file from over 120 GB to about 10GB, which was a pretty big win for my
laptop's 500GB SSD. Please let me know if you find any other tips/tricks in the comments that I should add
to this post.

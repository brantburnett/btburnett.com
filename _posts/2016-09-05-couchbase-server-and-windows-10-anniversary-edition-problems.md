---
id: 200
title: Couchbase Server and Windows 10 Anniversary Edition Problems
date: 2016-09-05T15:19:45+00:00
guid: http://btburnett.com/?p=200
permalink: /2016/09/couchbase-server-and-windows-10-anniversary-edition-problems.html
categories:
  - Couchbase
---
**Update:** This issue has been resolved in Couchbase Server 4.6 Developer Preview. You can certainly continue to use Docker, but there is no longer a requirement with Windows 10 Anniversary Edition.

## The Problem

Recently, I ran into some problems with my Couchbase Server 4.5 installation on my Windows 10 development box. The memcached process would crash over and over again with an error code 255.

After doing some research (and getting some assistance, thanks [@ingenthr](https://twitter.com/ingenthr)), I determined it's a known bug in Couchbase Server introduced by the recent release of Windows 10 Anniversary Edition. Apparently, Couchbase Server uses a third party library which incorrectly uses some private Windows APIs for memory allocation. The Windows 10 Anniversary Edition update removed these API calls, causing the crashes. The bug report is filed with Couchbase as [MB-20519](https://issues.couchbase.com/browse/MB-20519).

## The Workaround

The only known direct workaround is to uninstall the Windows 10 Anniversary Update. Personally, I don't find this to be a very good solution. Additionally, based on the bug report, I'm not optimistic about a quick fix from Couchbase. It seems like there's a lot of work involved, and it understandably isn't urgent because Windows is only supported for development, not production.

I decided instead to play with Docker, and I was very pleasantly surprised at how easy it was to use Docker to get Couchbase Server running on a Windows box. It only took me a few minutes.

1. Be sure that Hyper-V is installed on your machine via "Turn Windows features on or off" in Control Panel
2. Install [Docker for Windows](https://docs.docker.com/docker-for-windows/) (I used the Stable Channel)
3. Start Docker (I did this as the last step of the installation)
4. Right click the Docker icon in your system tray (next to the clock), and open Settings.  Go to Shared Drives, and share your C drive.  This will require your WIndows password.
5. Open Powershell and run this command to make a data folder:

    ```powershell
    mkdir $env:userprofile\Couchbase
    ```

6. Then run this command to startup the Docker container:

    ```powershell
    docker run -d --name db -p 8091-8094:8091-8094 -p 11207:11207 -p 11210-11211:11210-11211 -p 18091-18093:18091-18093 -v ${env:userprofile}/Couchbase:/opt/couchbase/var couchbase
    ```

7. Once complete, open http://localhost:8091/ to complete server configuration

## Notes

This configuration will always create the Docker container with the latest version of Couchbase Server, currently 4.5.  Command line arguments can be used to alter this, see the [Docker pages for Couchbase](https://hub.docker.com/r/library/couchbase/) for more information.

This configuration puts all Couchbase data in your C:\Users\myusername\Couchbase folder.  If you remove the Docker container and recreate, it will start up with your configuration and data already intact.  If you want to start from scratch, delete this folder before recreating the Docker container.

There are a few of compatibility requirements for this solution:

1. Hyper-V is incompatible with VirtualBox. If you are using VirtualBox, you should use a different solution.
2. The [client and management ports used by Couchbase](http://developer.couchbase.com/documentation/server/current/install/install-ports.html) must be available on your local machine.
3. This setup only supports running a single Couchbase node, otherwise there would be network port contention.

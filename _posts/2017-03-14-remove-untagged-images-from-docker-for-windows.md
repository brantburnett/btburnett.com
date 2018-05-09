---
id: 271
title: Remove Untagged Images From Docker for Windows
date: 2017-03-14T15:45:20+00:00
guid: http://btburnett.com/?p=271
permalink: /2017/03/remove-untagged-images-from-docker-for-windows.html
categories:
  - Docker
---
Here&#8217;s a quick note for Docker for Windows users. This is based on Jim Hoskin&#8217;s post [Remove Untagged Images From Docker](http://jimhoskins.com/2013/07/27/remove-untagged-docker-images.html "Remove Untagged Images From Docker").

I&#8217;ve simply reformatted his two scripts for use on Docker for Windows via Powershell. To delete all stopped containers:

```sh
docker ps -a -q | % { docker rm $_ }
```

To delete all untagged local images:

```powershell
docker images | ConvertFrom-String | where {$_.P2 -eq "&lt;none&gt;"} | % { docker rmi $_.P3 }
```
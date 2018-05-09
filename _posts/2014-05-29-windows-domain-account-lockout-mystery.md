---
id: 171
title: Windows Domain Account Lockout Mystery
date: 2014-05-29T09:15:34+00:00
guid: http://btburnett.com/?p=171
permalink: /2014/05/windows-domain-account-lockout-mystery.html
categories:
  - Windows
---
In addition to development, I sometimes get saddled with some domain administration.  We recently encountered a strange mystery, where a user's account was being locked out every day as soon as they booted up their computer.  They hadn't even tried to login yet, but their account was being magically locked out.

After lots of research, all of the obvious solutions were excluded.  We finally tracked it down by [turning on Kerberos logging on the client computer](http://support.microsoft.com/kb/262177).  We then found Event ID 14, stating "**The password stored in Credential Manager is invalid**".  But there were no passwords stored in the Credential Manager!

At this point, we found [this very helpful forum discussion that explains it](http://social.technet.microsoft.com/Forums/windows/en-US/e1ef04fa-6aea-47fe-9392-45929239bd68/securitykerberos-event-id-14-credential-manager-causes-system-to-login-to-network-with-invalid?forum=w7itprosecurity).

Apparently, your user account credentials can get saved to the SYSTEM (a.ka. local computer) account on the computer.  Once there, you can't access it through any normal UI to remove it.  We think this probably had something to do with our RADIUS auth on the WiFi network, but we're not sure.  Fortunately, the instructions in the post were spot on.

1. Download [PsExec.exe](http://technet.microsoft.com/en-us/sysinternals/bb897553.aspx) and copy it to C:\Windows\System32.
2. From a command prompt run: `psexec -i -s -d cmd.exe`
3. From the new DOS window run: `rundll32 keymgr.dll,KRShowKeyMgr`

> The only additional note I would add is that you need to run the command prompt as an Administrator, if you have UAC enabled.
{: .notice--info}

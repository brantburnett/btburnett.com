---
id: 95
title: Create A Self-Signed SSL Certificate In .NET
date: 2009-05-08T09:50:43+00:00
guid: http://btburnett.com/?p=95
permalink: /2009/05/create-a-self-signed-ssl-certificate-in-net.html
categories:
  - .NET Framework
---
A problem that I have commonly run into is trying to secure communications using SSL or other encryption for a intranet application. In this scenario, it is unnecessary to have a secure certificate signed by an expensive Internet authority. And often it is intended for deployment in a small-scale scenario where there might not be a Certification Authority running on a Window Server. In this case, you want to create a self-signed certificate and use the thumbprint of the certificate for phishing prevention.

Microsoft does provide a utility, makecert, which can create a self-signed certificate. However, it isn't distributed with Windows, is command line only, and definately NOT end user friendly. I wanted a method for creating a certificate just by clicking a button, without using a shell calls and distributing a copy of makecert with my applications.

To this end, I created a VB.Net class that calls out to the CryptoAPI and creates a self signed certificate with a 2048-bit RSA key. The certificate and private key are stored in the Local Machine store. In the Local Machine store it can be accessed by system processes and services. I've attached an example of the class to this post, feel free to use it as you see fit.

[CertificateCreator](/wp-content/uploads/2009/05/certificatecreator.zip)

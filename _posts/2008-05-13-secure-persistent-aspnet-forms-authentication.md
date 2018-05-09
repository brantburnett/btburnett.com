---
id: 11
title: Secure Persistent ASP.NET Forms Authentication
date: 2008-05-13T10:08:00+00:00
guid: http://www.btburnett.com/?p=11
permalink: /2008/05/secure-persistent-aspnet-forms-authentication.html
blogger_blog:
  - blog.btburnett.com
blogger_author:
  - Brant Burnetthttp://www.blogger.com/profile/16900775048939119568noreply@blogger.com
blogger_permalink:
  - /2008/05/secure-persistent-aspnet-forms.html
categories:
  - ASP.NET
---
<div>
  <small>While the ASP.NET Forms Authentication system is a great system for authentication, it has one significant shortcoming for a lot of sites. You can either only restrict it to always pass the authentication cookies in a secure manner or always pass them even if the connection is not secure. There is no intermediate method of authentication available to you. This means that if you are operating a web store you have a problem.</small></p>

  <p>
    <small>Normally, a web store wants the customer identified as soon as they come to the site, and throughout the shopping experience. However, when the user goes to edit their account or checkout, you want to switch them to a secure mode. In order to be secure, the cookie used to authenticate them for checkout must be restricted to SSL connections. This means that to maintain their login, you would have to remain in SSL from the moment they sign in forward, which adds a lot of unnecessary server load. Plus, it can cause headaches with external content you might want to include on your page that isn&#8217;t encrypted.</small>
  </p>

  <p>
    <small>The solution is to modify the forms authentication system to use a pair of cookies. One is valid only to identify you but not access secure functions, doesn&#8217;t require SSL to be transmitted, and is persistent across sessions. The other is a full authentication and requires SSL to be transmitted.</small>
  </p>

  <p>
    <small>The basic method this system uses is adding two additional HttpModules to your web.config file, PartialAuthenticationModule and PartialAuthorizationModule.</small>
  </p>

  <p>
    <small>The PartialAuthenticationModule works by adding to the existing FormsAuthenticationModule. The FormsAuthenticationModule is used as normal to process authentication for the secure authentication cookie (requireSSL should be set to true for the forms module in web.config). The PartialAuthenticationModule kicks in after the FormsAuthenticationModule. If there is no user on the current request (i.e. not a secure request, so client didn&#8217;t transmit the cookie) then the user is added to the context from the secondary insecure cookie instead. The user that is added will have the &#8220;Partial&#8221; AuthenticationType instead of &#8220;Forms&#8221;. In case the login has changed, it will also clear the old secondary cookie if the username doesn&#8217;t match with the secure cookie.</small>
  </p>

  <p>
    <small>The PartialAuthorizationModule processes later in the request life cycle. It uses the section of the section of your web.config files to identify security requirements on each folder. You just add a web.config file to each subfolder as required. This section supports two key fields. requiresLogin is either true or false, and specifies if the folder requires a secure login using the primary FormsAuthentication cookie. It verifies this by checking the AuthenticationType on the logged in user. The second field is requireSSL, which can be None, Optional, or Required. This will automatically redirect the request to or from https based on how you want the folder to be handled. This allows you to use normal app-relative URLs throughout your application without worrying about switching to and from https. Normally, for most folders you would use requiresLogin=&#8221;false&#8221; and requireSSL=&#8221;None&#8221;, and for secure folders you would use requiresLogin=&#8221;true&#8221; and requireSSL=&#8221;Required&#8221;.</small>
  </p>

  <p>
    <small>I&#8217;ve also implemented a PartialAuthentication static class that is very similar to the FormsAuthentication static class. It actually uses a lot of the methods from the FormsAuthentication class to help it with a lot of the processing. In particular, you should switch any login or logoff code to use the methods in the PartialAuthentication class instead of FormsAuthentication. This will create or remove both of the necessary cookies. To sign off a user from the secure section but still leave the persistent insecure cookie, use the FormsAuthentication.SignOff method instead.</small>
  </p>

  <p>
    <small>As far as security is concerned, the partial authentication cookie is generated using a FormsAuthenticationTicket just like with FormsAuthentication. It only difference is that the name of the cookie is prepended to the username in the ticket, and on decryption that string is validated. This prevents a client from editing their cookie file to set the secure cookie to have the same value as the insecure cookie. The cookies are encrypted using the same machine authentication keys as the forms authentication tickets, so they should work on web farms in the same manner.</small>
  </p>

  <p>
    <small>Please note that this library is designed for .NET 3.5 and Visual Studio 2008, though it should be easily<br /> convertible back to .NET 2.0 if you change the project settings.</small></div>

    <div>
      <strong>UPDATE: Please see updated version <a href="http://btburnett.com/2008/08/update-to-partial-authentication-system-2.html">here</a></strong>
    </div>

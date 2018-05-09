---
id: 222
title: 'Positron &#8211; HTML 5 UI For .Net Desktop Applications'
date: 2016-12-12T02:09:01+00:00
guid: http://btburnett.com/?p=222
permalink: /2016/12/positron-html-5-ui-for-net-desktop-applications.html
categories:
  - Positron
---
At [CenterEdge Software](http://centeredgesoftware.com/), the development department recently had management walk into a meeting and drop a bombshell on us. They wanted us to completely rebuild the UI from the ground up and &#8220;make it sexy&#8221;. Oh, and &#8220;make it last until 2025&#8221;. Don&#8217;t you just love wild management direction?

Well, we scratched our collective heads for a while and tried to come up with a solution. Preferably one that didn&#8217;t leave us clawing out our eyeballs, nor made management come asking about how we blew the budget. We came up with three key criteria we had to meet.

1. We need to leverage our existing .Net code as much as possible.
2. We need to be able to deliver the UI redesign in phases over time, rather than operate in a completely independent branch for a couple of years.
3. We need to support the current hardware infrastructure at our 600+ clients, so that hardware upgrades and networking redesigns are not required before upgrading.

### What We Aren&#8217;t Doing (And Why)

Based on these criteria, our first thought was WPF. We could continue writing .Net desktop client/server applications to operate our Point of Sale and other local applications. This would allow us to easily maintain our current low-level hardware integrations (read: lots of serial ports and USB serial emulation &#8211; ugh). It would also allow us to easily phase in portions of our application as WPF while other portions remain WinForms.

The downside to WPF is the size of the developer pool. There just aren&#8217;t that many WPF developers out there, especially great ones, and they tend to be expensive. But what about HTML5 and Javascript? There&#8217;s all sorts of great work happening surrounding web technologies and user interfaces. And there&#8217;s a much larger pool of developers with these skills for us to draw on.

Looking at HTML5 UI options, we looked at and discarded the two obvious solutions:

* **Convert to a cloud application.** We already have cloud components to our suite. But the core on-premise system, if converted, would become too reliant on stable internet connections, and too far from our current architecture to convert easily. This would also be very difficult to implement in a phased manner.
* **Operate an on-premises web server.** For our larger clients this wouldn&#8217;t be an issue. But a significant portion of our client base are smaller businesses that use workstations as their servers. IIS isn&#8217;t an option, and a computer with a couple extra GB of RAM might work for running SQL Express, but not for running the whole application for a half dozen stations.

### Can We Have Our Cake And Eat It, Too?

Is it possible to have the best of both worlds? Can we run a desktop client/server application like WPF, but yet build our UI using HTML5? Well, this question stumped us for a bit. But there are systems that do this, like [Electron](http://electron.atom.io/), just not for .Net.

Enter [Positron](https://github.com/CenterEdge/Positron)! Positron is a solution for building a .Net desktop application using HTML5 user interface components (rendered using Chromium). It hosts an in-process version of MVC 6 (yes, that&#8217;s the new .Net Core flavor), which is then wired in-process to Chromium running within a WPF window.

### Fun Facts

* All requests from Chromium to MVC are handled in-process, there&#8217;s no network stack or HTTP involved. This keeps performance and security very high.
* The window itself is the only WPF involved, the entire window content is the Chromium browser.
* ASP.Net Core MVC, despite having &#8220;Core&#8221; in it&#8217;s name, isn&#8217;t actually .Net Core specific. You can use it in traditional .Net Framework applications as well.
* All resources (images, views, CSS, JS, etc) are embedded in the DLL, making application distribution easy.
* Positron even supports Chromium developer tools (via a separate Chrome window).
* Positron is agnostic about how you build your HTML5 application. We&#8217;re currently using using React and TypeScript at CenterEdge. We&#8217;re even using Browserify and Gulp to build the JS files. But any web technology will work, pick your favorite flavor.
* There is currently one significant issue, the Razor view editor in Visual Studio doesn&#8217;t recognize the fact we&#8217;re working with MVC, so it&#8217;s a [squiggle fest](https://github.com/CenterEdge/Positron/issues/4). I&#8217;m sure support for this will be forthcoming, and it works fine after you compile and run. If there are any Visual Studio experts out there, we could use some help with this!

### What About Automated Testing?

All the QA and QE people out there are yelling at me now. They&#8217;re saying their test frameworks won&#8217;t work with Positron. We have a solution for that, too. You&#8217;re building an MVC application, just hosting it in-process. There&#8217;s nothing stopping you from spinning up a Kestrel server instead to support automated testing over HTTP. Just use Chrome as the test browser for parity with Chromium.

It&#8217;s also possible to install plugins in Chromium, so it might be possible to get some of the testing frameworks up and running directly against Positron if their plugin is installed. But we haven&#8217;t vetted this out yet.

### Open Source

Our specific use case is probably not common. But at CenterEdge we do feel like there is a need for desktop applications with HTML UI. There are many .Net desktop applications that could benefit from the plethora of great UI tools and frameworks available in the web development community. Therefore, we&#8217;ve decided to make Positron open source. It&#8217;s available on [NuGet](https://www.nuget.org/packages/Positron.Server/) (there are 4 packages), and the code is available on [GitHub](https://github.com/CenterEdge/Positron).

At this point it&#8217;s an early version (0.2.0), and there&#8217;s lots of room for improvement. We are already working on integrating this into our application suite, and then learning the pain points to make improvements

We&#8217;d also welcome community feedback. And feel free to fork the repo and send back pull requests.
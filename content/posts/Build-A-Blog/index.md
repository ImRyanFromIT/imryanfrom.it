---
title: "Build-A-Blog"
date: 2020-06-08T08:06:25+06:00
description: How to build your own blog from "scratch"
menu:
  sidebar:
    name: Build-A-Blog
    identifier: BaB
    weight: 40
hero: boat.jpg
draft: false
---
## What I did and why I did it, and how you can too
A little while ago I had my fill of using 3 HP Prodesks in a cardboard box as my homelab and decided to spring for something a little easier to work with, purchasing a ProLiant DL360 from ebay. One of my primary goals at the start was to lab it up and make something that seemed interesting and useful. Having a decent tenure in professional Googling, I’ve noticed that the people who create extremely detailed guides on obscure driver+OS combinations[1], or make public their trove of tutorials[2] always have their own blog. With this important information in mind I decided I’d make one too. 

## What I've done

There were a lot of options open when deciding on the self hosting route. Using Nginx, Apache, or IIS were the finalists for on-prem, with Azure Static Storage being a potential cheater option. Nginx and Apache are by far the most commonly used in the real world[3] but unfortunately require a little more Linux knowledge than I’m currently comfortable with. Azure Static Storage obviously isn’t an on-prem option, but it was still configuration intensive enough for me to earn its place as the break glass option in case of unforeseen issues. In the end, IIS was the winner. Granted, there are many reasons why IIS has largely fallen off in market-share when comparing it against the *nix options above, but for my existing Windows home environment it slotted as the easiest, and kept the scope of my project focused. 

To create the website itself I was looking at the static site generators Hugo and Jekyll. I know much less about css/node.js/ruby/go than I do about IT infrastructure, so I ended up choosing the Hugo exclusively because it had the prettiest theme. Simple. That pretty theme is Toha[4] by the way, which I highly recommend. It has a lot more capabilities than what I’m utilizing here AND has a fantastic wiki that I referenced often. 

## How I did it

### Hugo Website
#### Pre-Provision Requirements
1. Windows AD-DS
2. Create a Service Account. We need to create a service account in order to provide the web server access to files on the file server. I’m just using a standard service account, not a gMSA. 
3. Acquire a domain name from GoDaddy/Cloudflare/etc 
4. Acquire SSL Certificate. There are ways to get one of these very cheap or straight up for free. I go into more detail on this in the ‘Security’ section. 
5. Acquire a test site from Hugo/Jekyll/etc and drop it into your file server(or finish your own site before you start). Make sure it's in a share accessible by the service account you created earlier. **Important** - If you’re using Hugo, you need to ensure your site has a /public/ folder. This is the folder that we point IIS to. If it doesn’t, Hugo will create one with the command hugo server -w
6. Port forward the IP address of the web server you’ll be exposing to the internet. **I highly recommend you get a reverse proxy service rather than expose your WAN address directly.** More details on this will again be found in the ‘Security’ section.

#### Self hosting and IIS
The three primary goals I have with IIS hosting are to keep everything simple, stable, and secure. Specifically:
1. I want to keep the website files off of the web server, and centralized on my file server. 
2. I want to keep the building of the website(and the dependencies required) off of the web server.
3. I want to manage the SSL certificate in one place. 
4. I want the ability to burn and recreate the web server when necessary as easily as possible. 

#### Provision Server
I used PowerShell primarily deploy and configure IIS and the scripts used can be found here or below.[1]

1. Roles and Features
  * As with any Windows server, you should only install the roles and feature you need. Below is what I consider the minimum when deploying IIS for the first time. 

{{< highlight PowerShell >}}
Install-WindowsFeature -Name Web-Server, Web-HTTP-Logging, Web-Log-Libraries, Web-Filtering, Web-CertProvider, Web-Http-Redirect -includemanagementtools

Write-Host "Features have been installed" 
{{< /highlight >}}

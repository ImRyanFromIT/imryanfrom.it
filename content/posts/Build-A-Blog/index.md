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

2. App Pool Configuration
    * IIS has a feature called an application pool. Each application pool houses applications or sites on IIS. They provide lots of features, but we’ll be using it primarily to provide the authentication necessary to access the website files on my file server, and to manage security. 
    * Because I’m not working with multiple application pools in my environment, I’m just using the default pool to keep it simple. The script below prompts for credentials, and then configures the app pool to use those credentials instead of pass-through authentication. We’ll be using the service account that was setup earlier. 
    * Variables to change are **$app_pool_name**

{{< highlight PowerShell >}}
Import-Module WebAdministration

$app_pool_name = "DefaultAppPool" # change to your app pool name

$credentials = (Get-Credential -Message "Please enter the Login credentials including Domain Name").GetNetworkCredential()
$userName = $credentials.Domain + '\' + $credentials.UserName

Set-ItemProperty IIS:\AppPools\$app_pool_name -name processModel.identityType -Value 3
Set-ItemProperty IIS:\AppPools\$app_pool_name -name processModel.userName -Value $username
Set-ItemProperty IIS:\AppPools\$app_pool_name -name processModel.password -Value $credentials.Password

Write-Host "$app_pool_name has been configured."
{{< /highlight >}}

3. Central Certificate Store
    * We’re using a central certificate store to centrally store our certificates. Staying true to the theme of simplicity, I want to keep as much possible off of the web server as possible. This includes certificates, and the IIS Central Certificate Store allows us to do this. If you don't care, you’re also able to just copy the SSL cert locally to the Web server and specify it in the bindings section of your site.
    * You’ll also be prompted to enter the PrivateKeyPassword for your .pfx certificate. 
    * Keep in mind the store only works with .pfx files, so if you have any .crt certificates you’ll need to convert them. I used OpenSSL for this, and will cover it briefly later in this post. 
    * Variables to change are **$certStore**

{{< highlight PowerShell >}}
Import-Module IISAdministration

$certStore = "\\gv-data01\Shares\WebShare\SSL Central Store" # Change to your certificate store path
$c = Get-Credential # Prompts for certificate management service account

Enable-IISCentralCertProvider -CertStoreLocation $certStore -UserName $c.Username -Password $c.Password

Write-Host "IIS Central Certificate Store has been configured."
{{< /highlight >}}

4. Site and Bindings
    * Finally it's time to configure the IIS site and its bindings. This points people trying to access the website to the specific IIS server within my network. 
    * The below script has three parts to it
        * The first part removes the default site if it's around. This isn’t necessary but It makes the web server a little cleaner. 
        * The second part is the site creation sauce. It creates the site, changes the path from inetpub to my file server, and creates the HTTPS bindings.
        * The third solves a quirk of the AppPoolConfiguration script, and configures anonymous authentication from the App Pool to the App Pool’s service account. 
        * Variables to change are **$siteName**, **$physicalPath**, **$bindingInformation1**, and **$bindingInformation2**.

{{< highlight PowerShell >}}
Import-Module IISAdministration


# Remove Default Web Site
if (Get-IISSite -Name "Default Web Site") {
    Remove-iissite -Name "Default Web Site" -Confirm
    Write-Host "Default Web Site has been deleted."
}

# Variables
$siteName = "imryanfrom.it"
$physicalPath = "\\gv-data01\Shares\WebShare\imryanfrom.it\public" # Update this to your desired path
$bindingInformation1 = "*:443:imryanfrom.it"
$bindingInformation2 = "*:443:www.imryanfrom.it"

# Create new IIS website - bind both
New-IISSite -Name $siteName -PhysicalPath $physicalPath -BindingInformation $bindingInformation1 -Protocol "https" -sslFlag "CentralCertStore"
New-IISSiteBinding -Name $siteName $bindingInformation2 -Protocol "https" -sslFlag "CentralCertStore"

Write-Host "Website $siteName created with HTTPS binding."

# Ensure the WebAdministration module is loaded
Import-Module WebAdministration

# Set anonymous authentication to use the application pool identity
Set-WebConfigurationProperty -Filter "/system.webServer/security/authentication/anonymousAuthentication" -Name "username" -Location $siteName -Value ""
Set-WebConfigurationProperty -Filter "/system.webServer/security/authentication/anonymousAuthentication" -Name "password" -Location $siteName -Value ""

Write-Host "Set anonymous authentication to use application pool identity for $siteName."
{{< /highlight >}}

5. Voilà!
    * Assuming you completed the preparatory steps correctly and have changed the variables in the above commands to reflect your environment, then you’ll have an up and running website! However, we aren’t yet finished.
    * What if you want your website to be secure? Out of the box defaults for IIS can and should be tweaked. What if you want to make a change to your website? What if you want to write a blog? It's pretty annoying to have to hugo server -w and copy files into the file server. There are better ways to live! Answers to security, convenience, and more found deeper in this post. 

### Security
Web servers, specifically IIS, present very scary security challenges. One of the major benefits of having a vendor create and host your website is that you don’t have to worry as much about addressing all that annoying security stuff. However, as we’re self hosting, and it's paramount to provide powerful protection precipitating publishing. I’ve broken the steps I took into two sections - network security and system security. Please keep in mind these are recommendations, and you certainly should do more, less, or tweak what I’ve done to best fit your environment. 

#### Network Security
1. I've only exposed the HTTPS port to the internet. 
2. I’m using a reverse proxy so I don’t broadcast my WAN address to the world. Specifically, I’m using Cloudflare’s free reverse proxy service. You can find out more about it here. 
3. I’m ensuring my networking equipment is continually updated.
4. IIS server is segmented off on a different VLAN 

#### System Security
I’m using CIS recommendations for Microsoft IIS 10 security benchmarks. I love this organization and highly recommend it to anyone else in the infrastructure space. If a recommendation by RyanFromIT isn’t enough, here's why Microsoft recommends them. Due to my home lab not holding any sensitive information, I’m going to use the Level 1 Policy Definitions for **Basic Configurations**, **Request Filtering and Other Restriction Modules**, **IIS Logging Recommendations**, and **Transport Encryption sections**. All other sections I’ll be ignoring as they’re not applicable to my web server. Brief overview of changes I'm making are below. If you'd like to know more, I highly encourage you check them out yourself here. 
1. Basic Configurations
    * (L1) Ensure 'Web content' is on non-system partition
        * Completed by moving configuring the site to point to a file server
    * (L1) Ensure 'application pool identity' is configured for anonymous user identity
        * Completed in Site and Bindings section
2. Request Filtering
    * (L1) Ensure Double-Encoded requests will be rejected
    * (L1) Ensure Dynamic IP Address Restrictions' is enabled
3. Logging Recommendations
    * (L1) Ensure Advanced IIS logging is enabled (Automated)
        * We installed the feature for it
    * (L1) Ensure Default IIS web log location is moved (Automated) 
4. Transport Encryption
    * (L1) Ensure SSLv2 is Disabled
    * (L1) Ensure SSLv3 is Disabled
    * (L1) Ensure TLS 1.0 is Disabled
    * (L1) Ensure TLS 1.1 is Disabled
    * (L1) Ensure TLS 1.2 is Enabled
    * Ensure TLS 1.3 is Enabled
    * (L1) Ensure NULL Cipher Suites is Disabled
    * (L1) Ensure DES Cipher Suites is Disabled
    * (L1) Ensure RC4 Cipher Suites is Disabled
    * (L1) Ensure AES 128/128 Cipher Suite is Disabled 
    * (L1) Ensure AES 256/256 Cipher Suite is Enabled



{{< highlight PowerShell >}}
# Came from recommendations below https://www.cisecurity.org/benchmark/microsoft_iis

##############################################################################################################
###                                          Request Filtering                                             ###
##############################################################################################################

# (L1) Ensure Double-Encoded requests will be rejected
# Description: This Request Filter feature prevents attacks that rely on double-encoded requests and applies if an attacker submits a double-encoded request to IIS. When 
# the double-encoded requests filter is enabled, IIS will go through a two iteration process of normalizing the request. If the first normalization differs from the second, 
# the request is rejected and the error code is logged as a 404.11. The double-encoded requests filter was the VerifyNormalization option in UrlScan
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter "system.webServer/security/requestFiltering" -name "allowDoubleEscaping" - value "True"

# (L1) Ensure 'Dynamic IP Address Restrictions' is enabled
# Dynamic IP address filtering allows administrators to configure the server to block access for IPs that exceed the specified number of requests or request frequency.
$requestNumber = "10" # Restricts number of concurrent requests
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter "system.webServer/security/dynamicIpSecurity/denyByConcurrentRequests" -name "enabled" -value "True"
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter "system.webServer/security/dynamicIpSecurity/denyByConcurrentRequests" -name "maxConcurrentRequests" -value $requestNumber


##############################################################################################################
###                                       Logging Recommendations                                          ###
##############################################################################################################

# IIS will log relatively detailed information on every request. These logs are usually the first item looked at in a security response and can be the most valuable. Malicious 
# users are aware of this and will often try to remove evidence of their activities. It is recommended that the default location for IIS log files be changed to a restricted, non-system drive.
# Description: 
$logDirectory = "\\Share\Logs" # Your log location
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter "system.applicationHost/sites/siteDefaults/logFile" -name "directory" -value $logDirectory


##############################################################################################################
###                                         Transport Encryption                                           ###
##############################################################################################################

# (L1) Ensure SSLv2 is Disabled
# Description: The SSLv2 protocol is not considered cryptographically secure, therefore should be disabled.
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\ SSL 2.0\Server' -Force | Out-Null
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\ SSL 2.0\Client' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 2.0\Server' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 2.0\Client' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 2.0\Server' -name 'DisabledByDefault' -value '1' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 2.0\Client' -name 'DisabledByDefault' -value '1' -PropertyType 'DWord' -Force | Out-Null

# (L1) Ensure SSLv3 is Disabled
# Description: The SSLv3 protocol is not considered cryptographically secure, therefore should be disabled.
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 3.0\Server' -Force | Out-Null
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 3.0\Client' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 3.0\Server' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 3.0\Client' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 3.0\Server' -name 'DisabledByDefault' -value '1' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 3.0\Client' -name 'DisabledByDefault' -value '1' -PropertyType 'DWord' -Force | Out-Null

# (L1) Ensure TLS 1.0 is Disabled
# Description: The TLS 1.0 protocol is not considered cryptographically secure, therefore should be disabled.
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server' -Force | Out-Null
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Client' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Client' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Server' -name 'DisabledByDefault' -value '1' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.0\Client' -name 'DisabledByDefault' -value '1' -PropertyType 'DWord' -Force | Out-Null

# (L1) Ensure TLS 1.1 is Disabled
# Description: The TLS 1.0 protocol is not considered cryptographically secure, therefore should be disabled.
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server' -Force | Out-Null
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Client' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Client' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Server' -name 'DisabledByDefault' -value '1' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.1\Client' -name 'DisabledByDefault' -value '1' -PropertyType 'DWord' -Force | Out-Null

# (L1) Ensure TLS 1.2 is Enabled
# Description: TLS 1.2 is a more recent and mature protocol for protecting the confidentiality and integrity of HTTP traffic.
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server' -name 'Enabled' -value '1' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server' -name 'DisabledByDefault' -value '0' -PropertyType 'DWord' -Force | Out-Null

# Ensure TLS 1.3 is Enabled
# Description: TLS 1.3 is most recent
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.3\Server' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.3\Server' -name 'Enabled' -value '1' -PropertyType 'DWord' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.3\Server' -name 'DisabledByDefault' -value '0' -PropertyType 'DWord' -Force | Out-Null

# (L1) Ensure NULL Cipher Suites is Disabled
# Description: The NULL cipher does not provide data confidentiality or integrity, therefore it is recommended that the NULL cipher be disabled.
New-Item 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers\NULL' -Force | Out-Null
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers\NULL' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null

# (L1) Ensure DES Cipher Suites is Disabled
# Description: The DES Cipher Suite is considered a weak symmetric-key cipher, therefore it is recommended that it be disabled.
(Get-Item 'HKLM:\').OpenSubKey('SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers', $true).CreateSubKey('DES 56/56')
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers\DES 56/56' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null

# (L1) Ensure RC4 Cipher Suites is Disabled
# Description: The RC4 Cipher Suites are considered insecure, therefore should be disabled.
(Get-Item 'HKLM:\').OpenSubKey('SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers', $true).CreateSubKey('RC4 40/128')
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers\RC4 40/128' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null
(Get-Item 'HKLM:\').OpenSubKey('SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers', $true).CreateSubKey('RC4 56/128')
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers\RC4 56/128' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null
(Get-Item 'HKLM:\').OpenSubKey('SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers', $true).CreateSubKey('RC4 64/128')
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers\RC4 64/128' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null
(Get-Item 'HKLM:\').OpenSubKey('SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers', $true).CreateSubKey('RC4 128/128')
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers\RC4 128/128' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null

# (L1) Ensure AES 128/128 Cipher Suite is Disabled 
# Description: The AES 128/128 Cipher Suite is not considered secure and therefore should be disabled, if possible
(Get-Item 'HKLM:\').OpenSubKey('SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers', $true).CreateSubKey('AES 128/128')
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers\AES 128/128' -name 'Enabled' -value '0' -PropertyType 'DWord' -Force | Out-Null

# (L1) Ensure AES 256/256 Cipher Suite is Enabled
# Description: AES 256/256 is the most recent and mature cipher suite for protecting the confidentiality and integrity of HTTP traffic. Enabling AES 256/256 is recommended.
(Get-Item 'HKLM:\').OpenSubKey('SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers', $true).CreateSubKey('AES 256/256')
New-ItemProperty -path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Ciphers\AES 256/256' -name 'Enabled' -value '1' -PropertyType 'DWord' -Force | Out-Null
{{< /highlight >}}

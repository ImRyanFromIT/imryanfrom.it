---
title: "Github Actions with IIS"
date: 2023-09-08T08:06:25+06:00
description: How to make updates to a blog less annoying
menu:
  sidebar:
    name: Github Actions with IIS
    identifier: GawI
    weight: 40
hero: overview-actions-simple.png
draft: false
---
## Hugo, GitHub Actions, CI/CD, and De-annoyifying Processes
From my personal experience, CI/CD isn’t required in the systems world until you get to higher positions. I’ve been working on this project spontaneously though, and using Git has made the whole changing and deployment process a lot easier. Instead of manually copying and pasting Hugo Site chnages to my file share, I can just push whatever changes I've made in VSCode and watch them appear on my website. Instead of completely forgetting what I did last, the friendly website tells me. Very neat, very simple, very nice. Heres how I did it.  

### CI
To begin, you'll need a basic grasp on Git. My website is mostly an infrastructure project, with very little “programming” or internal processes that benefit from features like version control or branching. At the end of the day, changing the Toha theme around is basically just editing .yaml files. With that being said, working with my own repository really made the entire process easier to manage and track. Despite being the only person working on this project, there were many times I needed to roll back some changes or check what specifically I last changed something from. Overall I’m not including a how-to for the CI stuff as I don’t think it's necessary, but if you’re a newer in the systems world and want to try it out for yourself [FCC on youtube has a great tutorial](https://www.youtube.com/watch?v=RGOj5yH7evk) that handholds beginners through the entire process. 

### CD 
This part was very fun. I liked this part. I’m using Github Actions to make whatever changes are pushed to the imryanfrom.it repo show up automatically in the imryanfrom.it website. I keep saying it, but this makes the process so much easier. Before I get started explaining my actions, here's a primer on the feature. Github Actions are .yaml files stored in .github/workflows. They include workflows that are executed on an event (like a push to the main branch). The workflows within an action are ‘jobs’. Jobs define which machine or machines the workflow is ran, what steps it takes, and which shell it uses. The action is ran on on-prem machines using a runner agent, which must be installed on the machine. Finally, once you’ve pushed your action .yaml to your repo, you can find it under the ‘Actions’ tab in Github. If you’d like to know more [this getting started page](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions) on Github has more detailed explanations. If you like to learn by watching, [this youtube series](https://www.youtube.com/playlist?list=PLArH6NjfKsUhvGHrpag7SuPumMzQRhUKY) is very detailed and includes plenty of examples. 

### Design
Simple website gets a simple design. There are three machines involved in making everything work. The web server, the file server, and the build server. The web server has the necessary IIS roles but nothing else. Because it's facing the internet I want as little on that machine as possible. The file server server only contains the SSL certificate and website content. Finally, the build server will contain the NPM, Hugo, Git and Go packages. This is the machine that will do the heavy lifting when it comes to building out the changes that are made. Artist's rendition below. Arrows denote which machines are allowed to communicate which ways. I'm splitting the workload into 3 for a couple reasons:
1. I can spare the compute space
2. I want to keep the packages mentioned above segmented and off the web and file servers
3. The ability to burn and recreate the web server on demand was an earlier goal of mine
4. Web content and SSL certificates are much easier to manage when everything is centralized

Eventually the web and build servers can be containerized for further efficiency gains, but I'm keeping it simple for now. 

**Artist's Rendition**
{{< img src="/posts/Git-Actions-with-IIS/GrandDesigns.png" height="150" width="400" align="center" title="Artist's Rendition" >}}

### Creating the Build Action
Github Actions aren’t actually all that complicated if you don’t want them to be. We’ll be using the below action to ensure changes pushed to the master repository are automatically pushed to the IIS. Make sure to change the paths and the runs-on field to fit your usage. 

{{< highlight yaml >}}
# Name of the action
name: Website Integrate and Deploy

# Workflow triggers
# either triggers on a push to the main branch, or via a button in Github(workflow dispatch field)
on:
    push:
        branches:
            - master
    workflow_dispatch:

# Jobs within the workflow. In this example there are 2. The pull job and the build job. It's important to note that jobs in Github actions run concurrently, so in the 'build' job, we have the field 'needs: pull' to force it to wait for a success from the 'pull' job

# The pull's job is to pull the updated repository from Github to the file server. We're using pushd and popd to circumvent network path issues.
# pushd opens the network path, popd closes it.
jobs:
    pull:
      runs-on: [GV-BUILD01]

      steps:
        - name: Pull repo
          run: |
            pushd \\gv-data01\Shares\WebShare\imryanfrom.it
            git pull origin master
            popd
          shell: cmd

# The build's job is to integrate the changes and build it in Hugo. It consists of three parts. The first is to update Hugo modules, the second is install and update the node modules, and the third is to put everything together and build the website.
    build:
      runs-on: [GV-BUILD01]
      needs: pull

      steps:
      - name: Update Hugo modules
        run: |
          pushd \\gv-data01\Shares\WebShare\imryanfrom.it
          hugo mod tidy
          popd
        shell: cmd

      - name: Install node modules
        run: |
          pushd \\gv-data01\Shares\WebShare\imryanfrom.it
          hugo mod npm pack
          npm install
          popd
        shell: cmd
         
      - name: Build the website
        run: |
          pushd \\gv-data01\Shares\WebShare\imryanfrom.it
          hugo --minify
          popd
        shell: cmd
{{< /highlight >}}

### Configuring GV-BUILD
Our build server needs Hugo and Toha dependencies preinstalled in order for our actions script to work. We’ll use Chocolatey and a batch file to take care of this. Script is named IIS_BuildDependencies.bat [found here](https://github.com/ImRyanFromIT/PowerShell_Misc/tree/master/IIS). It installs:
1. Chocolatey
2. Hugo
3. GoLang
4. Node JS
5. Git

{{< highlight bat >}}
@echo off
echo Installing Chocolatey
powershell Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
:: Section 2: Package Installs
echo ========================================================================
echo Installing Hugo
choco install hugo --confirm
echo ========================================================================
echo Installing Go
choco install golang -y
echo ========================================================================
echo Installing NodeJs
choco install nodejs -y
echo ========================================================================
choco install git.install -y
{{< /highlight >}}

### Installing the Runner Agent
We’ll be installing a self-hosted runner on the build server as a service. Therefore, please ensure you’ve created a service account specifically for this process. I’m using a standard service account called IISRunner. 
1. Log into Github and head over into **Repository > Settings > Runners > Create self-hosted runner**
2. Run the script on that page in PowerShell, or the one below. Make sure you’re in your drive root before running this script. If you want to mess around with the location, please consult Github’s documentation on self-hosted runners. When I tried to move the install(and the _work folder) to a different location it caused issues.
{{< highlight PowerShell >}}
# Create a folder under the drive root
mkdir actions-runner; cd actions-runner
# Download the latest runner package
Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v2.308.0/actions-runner-win-x64-2.308.0.zip -OutFile actions-runner-win-x64-2.308.0.zip
# Optional: Validate the hash
if((Get-FileHash -Path actions-runner-win-x64-2.308.0.zip -Algorithm SHA256).Hash.ToUpper() -ne '05aa7d07223e7591f138db5a6a5f1b6f24ed22ab8b539307a6cf899f377f320f'.ToUpper()){ throw 'Computed checksum did not match' }
# Extract the installer
Add-Type -AssemblyName System.IO.Compression.FileSystem ; [System.IO.Compression.ZipFile]::ExtractToDirectory("$PWD/actions-runner-win-x64-2.308.0.zip", "$PWD")
{{< /highlight >}}

3. Run the **configure** script in the Github page referenced above. It’ll include a specific token you’ll need. Looks like this:
{{< highlight PowerShell >}}
# Create the runner and start the configuration experience
./config.cmd --url https://github.com/ImRyanFromIT/imryanfrom.it --token 123456789ABCDEFGHIJKLMN
# Run it!
./run.cmd
{{< /highlight >}}

4. If successful, getting started will look like this:
{{< img src="/posts/Git-Actions-with-IIS/Gactions1.png" height="624" width="232" align="center" title="Gactions1" >}}

5. You’ll be prompted to **“Enter the name of the runner group to add this runner to:”** We’re not using any special groups here, so press enter for default. 

6. Next prompt is to **“Enter the name of runner”**. Press enter for default, which is the hostname. 

7. Prompt after that will prompt you for runner labels. I like to add the hostname of the machine in my labels, but you don’t have to. If you’d like to go back later and add a label you can do so through the specific runners tab in Github. 

8. Following the labels you’ll be asked to enter the name for the _work folder. Keep it as default
{{< img src="/posts/Git-Actions-with-IIS/Gactions2.png" height="624" width="260" align="center" title="Gactions2" >}}

9. You’ll then be asked if you’d like to run the runner as a service (Y/N). You can run it as the **NT AUTHORITY\NETWORK SERVICE** if you’d like to leave it as default, but I'm going to use a specific service account. In my case it's **“greatvaluelab.com\iisrunner”**. Make sure you include the domain or else the installer will get confused. After that, enter the password. 
  * If you make a mistake on the username part you’ll need to completely uninstall the runner and try again. 

10. Looks good! Lets check our services and GitHub's Runners page to make sure everything looks like it should. 
{{< img src="/posts/Git-Actions-with-IIS/Gactions3.png" height="624" width="273" align="center" title="Gactions3" >}}
{{< img src="/posts/Git-Actions-with-IIS/Gactions4.png" height="578" width="81" align="center" title="Gactions4" >}}
{{< img src="/posts/Git-Actions-with-IIS/Gactions5.png" height="624" width="272" align="center" title="Gactions5" >}}

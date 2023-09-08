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

{{< img src="/posts/Git-Actions-with-IIS/GrandDesigns.png" height="400" width="600" align="center" title="Artist's Rendition" >}}

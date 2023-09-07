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
A little while ago I had my fill of using 3 HP Prodesks in a cardboard box as my homelab, and decided to spring for something a little easier to work with, purchasing a ProLiant DL360 from ebay. One of my primary goals to start was to lab it up and make something that seemed interesting and useful. Having a decent tenure in professional Googling, I’ve noticed that the people who create extremely detailed guides on obscure driver+OS combinations[1], or make public their trove of tutorials[2] always have their own blog. With this important information in mind I decided I’d make one too. 

## What I've done

There were a lot of options open when deciding on the self hosting route. Using Nginx, Apache, or IIS were the finalists for on-prem, with Azure Static Storage being a potential cheater option. Nginx and Apache are by far the most commonly used in the real world[3] but unfortunately require a little more Linux knowledge than I’m currently comfortable with. Azure Static Storage obviously isn’t an on-prem option, but it was still configuration intensive enough for me, and that earned it the break glass option in case of unforeseen issues. In the end, IIS was the winner. Granted, there are reasons why IIS has largely fallen off by market-share when comparing it against the *nix options above, but for my existing Windows home environment it slotted in the easiest, and kept the scope of my project focused. 

To create the website itself I was looking at the static site generators Hugo and Jekyll. I know much less about css/node.js/ruby/go than I do about IT infrastructure, so I ended up choosing the Hugo because it had the prettiest theme. Simple. That pretty theme is Toha[4] by the way, which I highly recommend. It has a lot more capabilities than what I’m utilizing here AND has a fantastic wiki that I referenced often. 


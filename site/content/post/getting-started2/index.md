---
title: 'SSL Inspection on Check Point'
subtitle: 'Outbound SSL Inspection: A war story'
summary: Leassons learnt working no SSL Inspection :rocket:.
authors:
- fede
tags:
- SSL
categories:
- Security
date: "2019-09-30T00:00:00Z"
lastmod: "2019-09-30T00:00:00Z"
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Placement options: 1 = Full column width, 2 = Out-set, 3 = Screen-width
# Focal point options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
image:
  placement: 2
  caption: 'Image credit: [**Unsplash**](https://unsplash.com/photos/CpkOjOcXdUY)'
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

 
Hello everyone,

I'm looking to share my experience and concerns regarding the current state of SSL Inspection in general and with Check Point, I'm also looking to see what is your approach in this matter.

Important: This post is strictly to Outbound SSL Inspection, my experience with inbound is limited with Check Point.

I'll start by making a vendor agnostic statement: One of the things that bothers me most about this subject is that there are many people that states that they have this technology implemented in their organizations and have no issues at all, after I met those organizations I found one of the following:

They have 10 users at most.
Their SSL Inspection technology is shady: SSL Inspection is enabled but there is a working and HUGE fail open at the bottom of the engine, if there's something that may cause issues, the connection is accepted, the log shows that the packet was "Inspected" and everyone is happy.
They think they have SSL Inspection but they don't have it enabled, instead they have a feature similar to Categorize HTTPS sites.

## My personal experience

We all know the importance to inspect HTTPS traffic in our network due to visibility and security, but I times I feel like this way of thinking is my InfoSec persona speaking that only gives woo woo advices to the business regarding security and doesn't even know how to install an antivirus software. Why? Because properly implementing this solution is just painful. I also know that most customers don't even want to dive into this matter and prefer to lay this layer of security into their endpoint solutions, even if it's not the same I respect that.

Within our customers we have one in particular that is very tech savvy with a 900+ users and likes to turn on all the features in their NGFW and one of the requirements was to enable full outgoing ssl inspection in the Check Point firewall. We started with this customer years ago with R77.30 in gateways so you can imagine that I've been through all kinds of fun experiences with SSL Inspection.

We have fully tested SSL Inspection in the following versions: R77.30 / R80.10 / R80.30

During this journey we went through a lot of information, there are many awesome posts here in Check Mates, SKs, SRs with the TAC, you name it.

## Issues that we faced

The following is a summarized list of issues that we had.

Heavy performance issues in R77.30 (Fixed in R80.10+)
There are many pages that fail to load or load sometimes.
There are many pages that work but after you enter a specific section just a login page, they fail.
Sophos Antivirus solution doesn't work, either install or updates: There are some post about this issue
AWS Connectors failing: (Solved in R80.30)
The issue with Check Point and Outbound SSL inspection

I really like Check Point firewalls, but sometimes I feel that they take control from the administrator. One of the first thing that we did was to disable all the options regarding dropping connections that don't follow the RFC line by line, allowing connections to not trusted certificates, we have tried it all, and still we faced a lot of issues.

Bypassing the connection? Good luck with that, Check Point firewalls always inspect the first packet and only by that there are many connections that fail.

Probe bypass to mitigate the previous issue? Sure, be prepared to have other issues due to SNI verification.

Fail open in probe bypass? This was a huge surprise after it was changed in Take 189, but we still had a lot of issues after enabling this flag.

WSTLSD debug? Many times, good luck not taking down the firewall with it and be prepared to wait a long time for the TAC to inspect it, not their fault, it's just really hard to troubleshoot these issues.

The only way that we found to properly bypass connections was to exclude it COMPLETELY from the SSL policy. Example: Let's say that you have two network segments and you only want to inspect traffic in one of them:

## What most people do

In this example all traffic from 10.0.0.0 will be inspected, however you probably will have some issues in the 192.168.0.0 network as well since the bypass action enforces to inspect the first packet of the SSL Handshake.

SSL 1.png

Only way do nothing with the connections

In our research we found that the only way to properly bypass a connection was excluding it completely from the policy. Obviously this approach is not scalable and somewhat utopic in a big network.

SSL 2.png

## The most stable scenario that we reached

We reached a state of stability in R80.10 with the JHF prior to 189 and by enabling the following flags and features:

appi_urlf_ssl_cn_allow_not_rfc_ssl_protocols=1 (Don't know where I get this, also there is not documentation about it)
enhanced_ssl_inspection 1 (Probe bypass)
bypass_on_enhanced_ssl_inspection 1 (Fail open probe bypass)
Almost all features that drop packets turned off in SSL Inspection.
HTTPS Categorization was turned on.
Sophos antivirus worked fine, issues with web pages went down to a minimum.

## The journey to R80.30

We decided to migrate one of the cluster members to R80.30 to test the new SSL Inspection engine and to solve some issues that we had with UserCheck. After the deployment we had some issues regarding Proxy ARP:

https://community.checkpoint.com/t5/Enterprise-Appliances-and-Gaia/Proxy-ARP-after-upgrade-to-R80-30...

Some Inspection Settings that started to cause issues in R80.30 and not previously.

We sorted them all and the first impression was just great:

AWS Connectors worked flawesly without enabling any of the previously stated kernel flags.
Sophos Antivirus could get updated without enabling any of the previously stated kernel flags.
All the services detailed in the preliminary testing document worked great.
The next day a waterfall of user complains started to appear:

Sophos Antivirus could not be installed: Updates worked fine but installation failed. After looking into the logs no traffic was dropped (Logs and output from fw ctl zdebug). The only log was a Detect regarding untrusted certificates which we configured to accept it in the SSL settings. We tried the flags, setting up a FQDN object just for *.sophos.com and setting this up in the SSL Inspection policy and still failed.
The main billing service of the company stopped working, again, no logs or possible leads why. We even looked at PCAPs and all seemed fine in the firewall perspective. At soon as we routed this traffic through the pfSense everything worked flawlessly.
Another invoice service stopped working: Again, no leads whatsoever, after we routed this traffic through pfSense all started to work.
Webpages that did not load properly or have some functions affected.
We tried everything, perform captures by turning off SecureXL, even the bypass flags could not solve these issues in R80.30, we had these issues in R80.10 and after turning on the different flags all worked, but not in this new version.

At this point there were many issues impacting production, we blocked one hole and another 10 appeared. It was just impossible to properly troubleshoot each issue, we had no other option but to go back to our most stable version.

## Future plans

There is no way to deploy SSL Inspection without issues, problem is that these issues will probably affect your production environment heavily, there is no way that you can test all your organization use cases, also there is no way to properly assure functionality in a lab enviroment.

Our main concern now is the remote possibility that we will have to live for life in R80.10, we know that we will have to update sooner or later that's why no we are implementing a parallel CHKP Frontier such as we have our failback pfSense, this new gateway will have R80.30 with the same features, thing of it as a hybrid testing/production enviroment.

The main idea is to route certain subnets to the R80.30 gateway and study their behaviour and troubleshoot without all the user complaining.

## Concerns regarding the current state of SSL Inspection

Check Point firewalls don't provide a proper solution to bypass desired SSL Traffic, therefore is really hard to deploy this solution on a big environment.
Lab tests are not representative, only way to test is in production.
Really difficult to troubleshoot: Many times there are no leads and everything seems fine in the firewall, forcing you to perform PCAPs on different parts of the network and debugging.
Check Point current approach on SSL Inspection impacts image perception of the brand: I heard it all the time "I have a friend who inspects SSL traffic with YYY and has no issues". Most people don't know that it's more of a technology issue in general regarding SSL/TLS instead of a Check Point fault, but other vendors offer this functionality and have fail backs mechanism that works without the user knowing, it's less secure but at the end of the day the main metric is functionality and not security in most cases.
Hope this post helps you in implementing this feature in a harmless way.

Regards,
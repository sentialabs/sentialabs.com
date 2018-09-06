---
layout: project
title: Data Continuity Services for AWS
section: Projects
navigation_weight: 2
---

**Why Data Continuity Service for AWS?**

Finding yourself landed on this page you might ask yourself why you would need a Data Continuity Service (DCS) for AWS in the first place because you are already creating daily backups for your databases or storage volumes.

Well, first of all it is really great to hear you are creating daily backups and for those who do not we strongly advise you to configure them asap as it is the first step in building continuity for your service or company. However having daily backups also means these backups are hosted in the same region and within the same account as your source data.

So although it's highly unlikely, it could happen that a region becomes out of service and you will not be able to access your data anymore. Now imagine your backups are in that same region. You would have no means to restore the service to your customers! It would mean disaster! And to make things even worse, imagine someone gets hold of your administrator access keys and starts deleting all of your AWS resources, data __and__ backups. [It would mean the end of your business](https://threatpost.com/hacker-puts-hosting-service-code-spaces-out-of-business/106761/) (no joke, must read!). This is where a Data Continuity Service for AWS would be a perfect fit. The DCS solutions we  will be showing make sure your data is replicated into another region and to another account so you will always be able to access your data in case disaster really strikes.

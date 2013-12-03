---
layout: post
title: Turkey Day Down Time
date: 2013-12-02T22:49Z
---
*Based on a true story, Thanksgiving 2012.*

Thanksgiving is a four-day long American holiday dedicated to stuffing yourself
with seasonal food and shopping on the Internet in your underwear. So it was no surprise that nobody
logged into the chat room at Cheezburger over the long weekend.

Well, until our monitoring systems indicated a total site meltdown.  I was pulled in by PagerDuty when 
the primary contact missed a page because he was in the middle of leftover-food-related activities. He
joined me online quickly. We found no evidence of a traffic abnormalities.
Our visitors were behaving themselves, not running a DDOS on our servers or anything discourteous
like that. The hardware was performing flawlessly. Our database layer seemed to be running a bit
more efficiently than usual, if anything. So we turned to the tried-and-true troubleshooter tradition:
turning it off and back on again. We ran an `IISRESET` on each of the web servers.

The site came back up. And stayed up.

We signed off, but I was uneasy with the lack of explanation and kept expecting another outage at any
minute. Nothing came, so I scheduled a Five Whys post-outage analysis for Monday afternoon. The
Director of Technology, the primary contact, and I set aside an hour to dig into the underlying reasons
the site was down over Thanksgiving. But we were stumped at the very first "why?"
**Why did the entire site go down?**

Without any explanation to the first question, there was nowhere deeper we could go. Luckily, we had
copious notes from every previous outage postmortem, with a Google Spreadsheet for collation. There
were a few outages with similar characteristics. For some reason, one of us decided to look up the
time since previous release for each of the outages in this cluster. The data screamed at us:

**Our site goes down 80 hours after the last release, &sigma; &lt; 12 hours.**

**We had been practicing continuous deployment for so long, site stability depended on it.**

Finally, we had an answer to our first question. The site went down because nobody releases code
over Thanksgiving weekend. At this point, it was the only hypothesis that remained, however improbable.
Modus Holmes; Q.E.D.

A geological dig through early layers of the code base revealed an in-memory static variable cache
that had been accidentally missed in the switch to centralized cache servers. That cache had functioned,
silently and unnnoticed, for aeons in Internet time, until it finally began to stretch the resource 
boundaries on the web layer. Our load balancer would detect the first server straining, and increase
the load on the remaining servers in the cluster, which accelerated their respective cache growth.
Eventually a second server would strain, then a third; eventually the cluster would be running at
full utilization. The load balancer would have nowhere else to shift the load, the servers would
start blocking requests, and the site would finally fall before the mighty torrent of thousands of
visitors every second. Only the cadence of continuous delivery prevented that cache from filling up
during normal weeks.

We tore out the static cache and replaced it with a centralized one, then resumed our vigilant quest
for site stability.


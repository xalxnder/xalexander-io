---
draft: false
title: "Breaking Things (Slightly) with a Nexus Upgrade: A Retrospective"
---
### Pre-Upgrade

Last night, I upgraded Nexus Repository Manager at my job. Nexus acts as a central hub for storing and distributing binaries, serving as a proxy for remote repositories, and hosting private artifacts. It sits behind Traefik, both of which are defined in a Docker Compose file. Since Traefik also needed an update, I figured I’d kill two birds with one stone 😅. I wasn’t nervous about the Nexus upgrade—it was a minor version jump. But Traefik? That made me nervous because it was a big jump.

### The Upgrade

After reviewing the release notes, I felt confident. I bumped the image tags and restarted the containers. Everything seemed to be going well. I could access the Nexus UI and successfully push and pull images. I went to bed feeling extremely accomplished, as this was my first-ever production upgrade.

### The Aftermath

The next morning, I logged in and went through my normal tasks—checking emails, pull requests, and double-checking that I could still push and pull images from Nexus. Everything seemed fine… until I got a message from someone in IT:

> **“Hey man. Sorry to be the bearer of bad news, but Nexus is mega-bricked.”**

My heart sank. This issue affected a high-priority team—we’ll call them Team Alpha. Minutes later, a ticket came in with more details: some of their Jenkins jobs were returning a 502 error when attempting to push artifacts to Nexus. Oddly, other jobs were successfully pushing artifacts to Nexus.

After a few hours of digging and troubleshooting, I got another ticket—this time from Team Alpha’s team lead. Again, my heart sank. Now they were requesting an elevated priority for the ticket. I reached out to their team lead, who mentioned that this same issue happened a few years ago, before I joined the company:

> **“I can’t remember exactly how you guys fixed it, but it was something with the proxy.”**

To me, this was good news—if it happened before, that meant it had already been fixed once. Though I didn’t know how, just knowing there was a solution was a step in the right direction.

He also mentioned that the 502 timeout only happened when pushing really large images—the kind that take some time to upload. And it was failing at exactly 60 seconds. That immediately made me think of Traefik. I went back through the release notes, and then it hit me right in the face:

	Starting with v2.11.2, the entryPoints.readTimeout option default value changed to 60 seconds.

The pushes of the large files were timing out because they were exceeding 60 seconds, the default value for `readTimeout`. This also explains why other jobs , pushing smaller files to Nexus, were succeeding.  I let Team Alpha’s lead know, and he gave me the go-ahead to update the Docker Compose file and restart Traefik.

After what felt like hours of waiting, he responded:

> **“Issue fixed!”**


🎉🎉🎉
### Lessons Learned

1. Don’t wait until the night of an upgrade to review release notes. If I had started going over the relase notes a week earlier, I would’ve caught the `readTimeout` change.
2. Avoid big gaps between upgrades if possible. Jumping from version 2.0.0 → 2.0.5 is much easier than 2.0.0 → 5.1.0.
2.Expand the scope of the post upgrade testing and validating. Ok it works for my team, but what about other teams who use it?
3. Really, really read every change in the release notes. (Yes, every change.)
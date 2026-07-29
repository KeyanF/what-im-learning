# LiteLLM setup

## Goal
Because of my interest in AI and hosting local models, I wanted to be able to run Claude Code using a local model. I also wanted to be able to access this from anywhere, and share access with other people.

## Choosing how to share access
The first decision I had to make was *HOW* I share access. One way I considered was through Tailscale, a VPN specifically designed for remote access to networks with authentication. It is based on the WireGuard protocol and supports subnet broadcasting/routing, among other features, and I've used it for other projects. It is genuinely useful.

However, from my understanding Tailscale doesn't have the best options for sharing a service to someone outside your Tailnet. While I *COULD* have shared a whole node to someone else with a Tailscale account, that would allow access to **ANY SERVICE** bound to all interfaces (`0.0.0.0`) or the Tailscale interface specifically. And while I could have made firewall rules restricting access from that interface except for a specific service, that option seemed more messy to me, and messy is where mistakes and vulnerabilities live.

Instead, I decided to use Cloudflare Tunnels.

## Why Cloudflare Tunnels
I've actually used Cloudflare Tunnels before, and that was to make the admin panel for my Minecraft server available on the public internet behind Auth, so my co-owner could help me manage.

Without writing a whole essay on why I picked Cloudflare Tunnels over Tailscale/some kind of VPN, I'll summarize by saying: I didn't want to go to the trouble of adding someone to the access list of my shared Tailscale node every time I wanted to let them try my local model API. I know, not some super technical security reason, I guess this is why they say it's always a balancing act between security and accessibility/ease of use.

## How it fits together: Cloudflare, LiteLLM, and the local models
So basically the flow is: somebody goes to the API endpoint on my public website > it goes through Cloudflare, so I get the DDOS protection and additional Auth if I wanted it. From there the tunnel translates requests so it's as if they're from a local service letting me bind vulnerable services like llama.cpp and vLLM to localhost. But the Auth for llama.cpp and vLLM are...questionable at best. ALSO, Claude Code uses Anthropic's API format, not the OpenAI-based API format that local servers typically use. That's where LiteLLM comes in to play. LiteLLM does both API key auth, which isn't as questionable as the model backends' servers themselves, and it also acts as a proxy to translate that OpenAI format to Anthropic's API format.

## Running the service as a dedicated user
Setting this up mostly involved 2 files, a configuration file for the server and a `.env` file for the secret (API key). The computer I'm using to host these models runs Linux, which by default would run that server as me (runs the service under my name). This is bad because my user is added to other groups, such as `docker` and `sudo`, which is bad because it lets an attacker run commands as my user instead of a low-privilege user should they ever find a way to get code execution by finding a vulnerability in the service such as a buffer overflow, or maybe a backdoor in a python package I used.

## Users and groups
So I made a new user. This user would be the one that runs the service, so if the procesr gets a low privilege user. On Linux, every file needs to be owned by one user, AND one
group. Normally Linux will make a new group called a User Private Group, which literallyThis is so you don't have to add access to that file to a system group, which willtypically have more than one users, giving more permissions to other users than you need. Problem with this, another use needs access to one of two files i mentioned earlier. That's me. When I start Claude
Code from the same machine that I host the models from, I don't want to have to type in n each time it changes.

## Locking down the secret file
So instead of a UPG (User Private Group) I made a system group and put my user and the user the service runs as, in it. Without going into too much detail of how Linux file permissions work, I made it so only those 2 users could read the key, but neither could change it. Only root can change that secret key.

## Hardening with systemd
I checked the service with systemd's security check, and it was still a 5.5/10, which isn't great. This was mainly because of 2 things, first, because of the Set User ID bit (SUID). A normal user in Linux
isn't allowed to do a lot of things. For example, access the file that stores passwords,y still need to be able to change their own password. The SUID bit lets regular users run
a service as the owner (root, in most cases) instead of themselves. This would let that  password without accessing other secret parts of the file.

Problem with this is sometimes there are vulnerable services with SUID privileges. This  as ANY user do pretty much ANYTHING. Obviously very bad. To fix this, I set some settings to make sure no process started by that user could EVER get higher privileges.

The other problem is that even though certain users can't run a lot of system functions and commands, they can still TRY to run those. And sometimes there are vulnerable functions where they can TRY to run
with some sketchy parameters, thereby getting the attacker access to code execution in tmissions. By fixing these 2 large issues in addition to hardening those config file
ownership properties, I was able to reduce the risk score down to a 1.6, which is consid

### What I learned/TL;DR
For maximum security, you need minimum privileges. You should use multiple layers of defense in case one is broken. Linux is secure, but only if you configure it correctly.

I would say this project greatly helped my understanding of Linux permissions, users, grle to practice this stuff with a lot of hands on reps would help it get even moreingrained in my head.

# What I Learned: AWS Bastion Host SSH Timeout

Personal notes from actually going through this — not a generic how-to, just what I ran into and what it turned out to mean.

---

## How my traffic was actually getting to the instance

This is the path my connection was taking, which I only really understood once I started debugging it:

```
My Laptop
     │
     │ Wi-Fi
     ▼
Mobile Hotspot
     │
     ▼
ISP (Reliance Jio)
     │
     │ assigns Public IPv4
     ▼
152.xx.xx.xx
     │
     ▼
Internet
     │
     ▼
AWS Internet Gateway
     │
     ▼
EC2 Public IP (3.83.8.190)
     │
     ▼
Ubuntu Server
Private IP (10.0.11.38)
```

Nothing about this was broken, ever. That turned out to be the whole point of the exercise.

---

## The IPs I ran into, and what I got wrong about each

**My laptop's own IP** — something like `192.168.189.25`. Private, only exists inside the hotspot's own little network. AWS never sees this, and it never mattered to my problem.

**The hotspot's own IP** — `192.168.189.1`. My phone acting as a mini router. Also irrelevant.

**My public IPv4** — this is the one that actually mattered, and the one I struggled to pin down. It kept showing up as different things depending on how I checked it, which was the whole source of my confusion.

**My IPv6** — `2409:40c4:2000:5496:...`. I got this back from `curl ifconfig.me` at one point and it told me nothing, because my Security Group rule was IPv4-only. Unless you've deliberately set up IPv6 on both the EC2 side and the Security Group, an IPv6 address is just noise for SSH.

**EC2's public IP** — `3.83.8.190`. What I was actually SSH-ing into: `ssh -i key.pem ubuntu@3.83.8.190`.

**EC2's private IP** — `10.0.11.38`. Only exists inside the VPC, never directly reachable from outside.

---

## What was actually happening during the timeout

My laptop was sending traffic like this:

```
Laptop
Public IP = 152.59.31.54
       │
       │ SSH
       ▼
Internet
       │
       ▼
AWS Internet Gateway
       │
       ▼
EC2
Public IP = 3.83.8.190
```

And on the way in, AWS was checking it against the Security Group rule:

```
Allow SSH from
152.59.28.61/32
```

My actual public IP by that point had become `152.59.31.54` — not `152.59.28.61`, which is what the rule still had. AWS compared the two, they didn't match, and it just silently dropped the packets. No error, no rejection — just a timeout on my end after waiting long enough.

That mismatch was the entire problem. Nothing else — not routing, not the instance, not the port — was ever actually wrong.

---

## Where I actually went to find my real IP

I tried `curl -4 ifconfig.me`, and also checked sites like ifconfig.me and whatismyipaddress.com in the browser. Both of those gave me *an* answer, but on a hotspot, those answers weren't something I could fully trust in the moment — the browser check in particular ended up showing a different IP than what the server logs later confirmed was actually connecting. A client-side check is only ever a snapshot of that exact second.

To actually get back in and stop guessing, I temporarily opened the Security Group rule to `0.0.0.0/0` (allow SSH from anywhere) and SSH'd in from my own terminal — that connected immediately, since with the rule wide open it no longer mattered what my IP actually was. That got me a working session, but it didn't tell me anything about *why* the original rule had failed.

So, from inside that session, I ran:

```bash
sudo journalctl -u ssh --since "10 minutes ago"
```

And that's the moment it actually clicked. The log showed my connection coming in from an IP that was **not** the one I'd been assuming was mine the whole time — it matched neither what `curl` had shown me locally, nor what I'd had saved in the Security Group rule. In other words: logging in with the `0.0.0.0/0` rule fixed my *access*, but it was `journalctl` afterward that fixed my *understanding*. Two separate problems, two separate fixes. Even after I was back in, I still didn't actually know my real IP until I read it straight out of the server's own log. That log was the only thing in the entire session that told me the truth.

What actually settled it for me was checking from the server's side. I first installed `wtmpdb` (since `last` wasn't available on the image out of the box):

```bash
sudo apt update
sudo apt install wtmpdb
```

and then ran `last`, which gave me a clean login history:

```
ubuntu ssh 152.59.31.54
```

Then, to double-check against the actual SSH daemon log:

```bash
sudo journalctl -u ssh --since "10 minutes ago"
```

which showed something like:

```
Accepted publickey for ubuntu from 152.59.31.54
```

That's the IP AWS genuinely received the connection from — not a guess, an actual record.

Between `last` and `journalctl`, I had something I could actually rely on — unlike the client-side IP checks, which kept disagreeing with each other and with what the server had actually seen.

---

## The EC2 Instance Connect surprise

At one point the logs showed something like:

```
Accepted publickey for ubuntu from 18.206.xxx.xxx
```

I almost mistook this for my own laptop's IP. It wasn't — it was AWS's own EC2 Instance Connect service, from when I'd used the browser-based "Connect" button in the console instead of my own terminal. Good thing to recognize, otherwise it's easy to chase the wrong address entirely while debugging.

---

## How I actually fixed it

Once I had the real IP from the logs, I went to the Security Group and updated the SSH rule's source to that confirmed address (`152.59.31.54/32` in my case), saved it, and reconnected. That worked immediately — `Welcome to Ubuntu...` — and re-checking `journalctl`/`last` afterward confirmed everything now lined up: the rule, the real IP, and the server's own record of the connection, all agreeing.

---

## The part that really stuck with me: it can happen again anytime

Since I was on a mobile hotspot, my public IP wasn't something fixed — it moved. If I switch phones, SIMs, or Wi-Fi, or just reconnect the hotspot, the IP that gets assigned to me can be completely different from before, with zero warning. Every time that happens, the old Security Group rule stops matching and SSH goes back to timing out — not because anything broke, but because the one thing the rule depended on (my IP staying the same) quietly stopped being true.

---

## How to actually prevent this from happening again

Chasing my own IP every time my hotspot decided to reassign it isn't a real fix — it's just a recurring chore. A few actual ways to prevent this instead of re-living it every session:

**Use a VPN with a static exit IP.** This is the most direct fix for exactly this problem, and it's the one I'd actually reach for first now.

- *Why it works:* the root cause in this whole write-up was never AWS, routing, or the instance — it was that my ISP handed me a different public IP every time the hotspot reconnected, and the Security Group rule was written against one specific, frozen-in-time IP. A VPN with a static exit IP removes the moving part entirely. Instead of AWS seeing "whatever IP Jio felt like assigning me this hour," it always sees the same one fixed address, every time, from every network.
- *What "static exit IP" actually means:* when you connect to the VPN, your traffic leaves the VPN provider's server, not your hotspot — and that server's outbound IP is the one AWS sees. If that outbound IP is fixed (dedicated to you), the Security Group rule only ever has to be written once. If it isn't fixed — most free or basic consumer VPNs rotate exit servers for privacy — you're back to square one, just with a different unpredictable IP instead of your ISP's.
- *How to set it up, practically:*
  1. Get a VPN with a **dedicated/static IP** — either a paid feature from a commercial VPN provider (sold as "dedicated IP" or "static IP" add-on), or a self-hosted option like WireGuard or OpenVPN running on a small, always-on cloud VM (even a $4–5/month box) that has its own fixed public IP.
  2. Connect to that VPN from your laptop before you SSH.
  3. In the EC2 Security Group, set the SSH rule's source to that VPN's static IP as a `/32` (e.g. `203.0.113.10/32`) — once, permanently.
  4. From then on, it doesn't matter whether you're on Jio, Airtel, hotel Wi-Fi, or a friend's hotspot — as long as you connect through the VPN first, the IP AWS sees never changes.
- *The one thing to double-check:* confirm the "static" part actually holds by running `curl -4 ifconfig.me` while connected to the VPN, a few times, on different days/networks. If that number ever changes, the provider isn't giving you a truly dedicated IP, and the Security Group rule will silently start failing again — exactly like it did in this write-up, just with the VPN's IP instead of the hotspot's.

**Use EC2 Instance Connect for browser-based access when you're not near your usual network.** It connects through AWS's own service IP range rather than your own changing address, so it doesn't care what your current public IP is. It's not a full substitute for a proper terminal SSH workflow, but it's a reliable fallback when your IP has clearly moved and you just need to get in.

**Use a bastion/jump host with a Session Manager or SSM-based connection instead of raw SSH+IP rules.** AWS Systems Manager Session Manager lets you connect to an instance without opening port 22 to any IP at all — no Security Group SSH rule to maintain, no IP to chase.

**If you do want to stick with plain SSH + Security Groups, at least automate the IP update** rather than doing it by hand each time — a small script that runs `curl -4 ifconfig.me`, compares it to the current SG rule, and calls the AWS CLI (`aws ec2 modify-security-group-rules` or authorize/revoke) to update it automatically before you connect. Still fragile in the sense that it depends on `curl` and a live network, but at least it removes the manual guesswork.

**Whatever you choose, don't rely on `curl` or a browser check alone to confirm what actually reached the server** — that's the one lesson this entire session kept reinforcing. Those tools are useful for a first guess, but `journalctl -u ssh` and `last` on the instance itself are what actually confirm whether a given IP got through, which matters a lot if you're troubleshooting rather than just connecting.

---

## Other errors I ran into along the way

**`Permission denied (publickey)`** — this one was about the username or key, not the IP. Worth remembering: Ubuntu AMIs use `ubuntu`, Amazon Linux uses `ec2-user`. Easy to mix up if you're bouncing between instance types.

**Bad key permissions** — Git Bash wanted `chmod 400 key.pem`; on Windows PowerShell it was a matter of stripping other users' access from the file's Security settings (or using `icacls`) instead.

---

## What this all came down to, for me

Whenever SSH suddenly stopped working after switching networks, hotspot, SIM, or ISP, the actual root cause was always the same: my public IP had moved, and the Security Group rule hadn't moved with it. `curl` and browser IP checkers were only ever partial, sometimes contradictory clues — even after opening the rule to `0.0.0.0/0` and getting back in, it was `journalctl -u ssh` that finally told me what my actual IP really was. That's the thing I actually trusted, every time, to see exactly which IP had really gotten through. Routing, NAT, the Internet Gateway — none of that was ever the actual issue, no matter how much it looked that way at first. And going forward, a static-IP VPN is the real fix — everything else here was just me working around the symptom, one changed IP at a time.
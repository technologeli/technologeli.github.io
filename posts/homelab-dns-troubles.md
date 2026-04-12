---
title: Homelab DNS Troubles
---
# Homelab DNS Troubles

Over the past week, I've been setting up my homelab on a pretty beefy laptop
whose screen unfortunately broke on me a few years ago.

I recently moved into an apartment that manages its network in a way that I
can't configure any router settings. As of right now, my homelab is built on
DHCP and a dream - but that's an improvement from what it was!

## A wishlist

I want my homelab to be relatively functional with security and flexibility in
mind. With that being said, here's a non-exhaustive list of what I set out to
set up:

- [Proxmox](https://www.proxmox.com/en/), a hypervisor
- [Tailscale](https://tailscale.com/), for remote access
    - Because of my apartment's network settings, I couldn't even access my
      instance without Tailscale. No local IPs can access each other.
- [Vaultwarden](https://github.com/dani-garcia/vaultwarden), a password manager

These were pretty much my only goals at first.

## Attempt 1

I got pretty far before I learned a few things that in hindsight felt pretty obvious:

- Don't encrypt a drive of a service you want available on system reboot, such
  as, I don't know, Proxmox itself. Otherwise, you'll have to physically
  connect to the device to even start it.
- If you want to run a server on a laptop, disable hibernation/sleep on lid
  close. Linux has this cool feature that prevents this when an external
  display is connected (such as the monitor I was using anyway with my broken
  laptop screen), but the moment it's disconnected, it's hibernation time.

To access my Proxmox instance, I created a Tailscale LXC and made it a [subnet
router](https://tailscale.com/docs/features/subnet-routers) so that I wouldn't
have to put Tailscale on the host or any other VM/LXC.

Then, it came time to install Vaultwarden. I was very excited, until I
learned I needed to self-sign certificates. Fortunately, `openssl` is very well
documented and online tutorials and AI make up for my lack of patience.

## Certified shenanigans

Unfortunately, Android does not appreciate my self-signed certificate
(rightfully so). The Bitwarden app really didn't like it. I thought manually
importing the certificate on my phone would work, but then the Bitwarden app
still threw a tantrum. I decided I would kill two birds with one stone, and set
up DNS and handle this certificate issue using [Nginx Proxy
Manager](https://nginxproxymanager.com/).

Now, I've never used (this) NPM, but I found [a
video](https://www.youtube.com/watch?v=qlcVx-k-02E) outlining the exact process
I wanted. Granted, he was using Docker, but it didn't really make a difference.
I'll be summarizing what I did based off of the video, but I like my solution
more.

In the later half of the video, the homelabber's strategy was to create a
public DNS A record to point his NPM subdomain to his private IP. He then
created a wildcard CNAME record that pointed all other subdomains to his NPM
subdomain.

If you're like me and don't know what A and CNAME records are, see it like this:
- An A record maps a domain or subdomain to an IP.
- A CNAME record maps a subdomain to another domain or subdomain.
- A wildcard is just a catch-all.

One aspect that he did not cover that I was concerned about was using a domain
that is already in use. I'm cheap, and I'm using this domain (`elieu.dev`) for
both this blog and my private homelab.

It turns out, I can kill two eggs with one bird, because his end solution ends
up with sub-subdomains like `foo.bar.baz.com`, but mine is only `foo.bar.com`.
My blog is unaffected if you're reading this.

What I ended up doing was making two more records on Cloudflare:
- An A record mapping my NPM subdomain to my private IP
- A CNAME record mapping `*` to my NPM subdomain.

My existing records handle my blog.

Here's the key part: on NPM, I made an SSL certificate for just `*.elieu.dev`,
not whatever the video did. This allowed me to create all of the single
subdomains I wanted that get resolved by NPM!

## Real homelab hours

With DNS and SSL fixed, my Vaultwarden now works from the Bitwarden app.

Some things I'll be adding to my homelab soon (or have added):

- [Glance](https://github.com/glanceapp/glance) - a minimal homepage
- [Immich](https://immich.app/) - photo sync and local facial recognition
- [Miniflux](https://miniflux.app/) - a really simple RSS reader
- [GitLab](https://about.gitlab.com/) - I hope you know what this is
- [NextCloud](https://nextcloud.com/install/) - A cloud calendar and file
  backup
- Probably some local CTF tools and a solving VM

See you when my DHCP lease ends!

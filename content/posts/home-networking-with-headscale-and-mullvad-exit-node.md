+++
title = "Home Networking in the 2020s with Headscale and Mullvad Exit Node"
date = 2024-12-08T23:45:00+00:00

[taxonomies]
tags = []
+++
This post details my home network that utilizes a self-managed mesh VPN. I'll walk through motivations and iterations of this, highlight the ease-of-use and privacy benefits that this provides, and share enough detail to help with replicating something like this on your home network.

![Headscale-managed tailnet diagram](/headscale-tailnet.webp)
<!--more-->

#### First things first: why?
I've always run a home server of some kind. Around 2013, I ditched my Mac Mini server and big old Linux box for a NUC and a storage array that hosted at least backups, a media server, and a place to tinker with software that I found cool, interesting, or lucrative.  Having a setup like this at home was rewarding in several ways: it's fun, it's functional, and it provokes learning. When the company I worked for at the time was starting to look into configuration management, my "homelab" was where I really learned Puppet hands-on and became a strong proponent of automation. Fast forward to 2015, my home server is where I first discovered Docker, and wrapping my head around that felt even more revolutionary than understanding configuration management. If you're reading this, it's likely that you understand the concepts of containerization. I'll guess you probably had a big "aha" moment the first time you stood up a container and the app was instantly built, configured, and made live. Coming from the now-historic way that Linux systems used to be managed by hand, or with some scripts that were still a manual and hacky process, gaining an understanding of Configuration Management and Containerization feels like a superpower. I mention all of this to set some background, and also to set an expectation: if this is relatable so far, please read on! I'm going to be discussing other concepts that felt similarly groundbreaking upon discovery. Ultimately, the "why" behind setting this up is connectivity to my server.

#### Let's talk networking.
We all know that a server isn't much without a usable network around it. I know a home server is awesome and you know a home server is awesome, but it's not awesome when you are out in the world and you don't have a way to get back to your server as you have ideas and time. This led me, several of my friends, and nerds all around the world to end up self hosting our own VPNs at home. The solution that I stuck with for several years was OpenVPN on a docker container, running on a home server that stayed on 24x7 hosting a handful of services. (Hooray for mini PCs that sip power!) This worked great. I would just "OpenVPN in" when I want to access home stuff. If I was out and about without a laptop, no problem! I can just as easily connect from a smartphone.

As Wireguard came to prominence in the late 2010s, I switched my OpenVPN server out for Wireguard and was extremely happy with the performance gains and reduced technical complexity of the VPN itself. (Wireguard's stance on authentication is a good example of this.) Something I have done at times, both with Wireguard and OpenVPN, is run side-by-side tunnels: one that captured routing for just my home network range, and another that all traffic would route through. Both tunnels served the purpose I've covered up to this point: accessing computers on my home network when I'm not physically at home. The second tunnel would route all network traffic through the tunnel, adding privacy benefits. Using the second tunnel, I could not only get to my computers at home, all network traffic leaving my device would traverse a (VPN) encrypted link back home, then out to the internet from there. Sketchy wifi? Not as sketchy now! Geo-restrictions? Ha.

Nowadays, VPN has become a loaded term. While VPN always means *virtual private network*, there are two very common, very distinct use cases:

 - Traditionally, a VPN provides network access to devices that sit outside of a physical IP-based network. The big use case here is employees accessing company resources when they are not in the office. Site-to-site VPNs can be considered a more advanced version of this use case. This is how I would use my home VPN.
 
 - The past 10 years have seen an explosion in "VPN privacy services". These are not VPNs for accessing your systems, but rather VPNs that shield your traffic from network middlemen that increasingly collect and sell everything they can learn about you to data brokers. In the United States, ISPs spend big on lobbying to oppose policy that would stop this. Many of these VPN privacy service companies have controversial practices of their own, and they tend to overstate their benefit. At the time I'm writing this, I've just renewed my Mullvad privacy VPN subscription for another year. Please don't take this as a recommendation for VPN privacy services in general.

So VPNs are useful, and VPNs are useful, and combining the VPN use cases is even more useful. Without a combination, you're hopping back and forth between a privacy service and your personal VPN as needed. This adds major friction to computer use and will eventually confuse your operating system to the point you just have to reboot with your fingers crossed, and that's true regardless of operating system!

Combining these approaches is simpler than it may seem. The trick is to route your network range to a VPN that is talking to your network, and to route everything else out to the privacy VPN. Just don't try to run your privacy VPN provider's mediocre app on top of a system connected to your home Wireguard tunnel or something similar. The best case scenario there is it mostly works in that moment, because the multiple components trying to handle the default route for the system were launched in a lucky order. DNS will also need some luck. Even if it works, this isn't reliable.


#### Tailscale has entered the chat
At the same time that VPN privacy services have seen a boom in growth in recent years, Tailscale's popularity has grown maybe even more. [Tailscale](https://tailscale.com) is both the name of a mesh VPN and the company behind it. It provides a VPN that allows you to connect to your devices that are also connected to the same tailnet (Tailscale network). There are some significant differences with Tailscale compared to a traditional VPN though:

- Traditional VPNs are typically seen as a gateway into another network. Once you're in, you can get to the systems on your LAN without those LAN systems being are of the VPN. This is possible in Tailscale too (via subnet routes) by enabling options, but it's not a default.
- The primary way device-to-device communication works involves connecting all of your devices that need to reach each other to the tailnet. These devices (the tailscale clients) all receive an IPv4 address in the CGNAT range, an IPv6 address in the `fd7a:115c:a1e0::/48` unique local address (ULA) range, and a DNS name by default. Tailscale routes those IP ranges over a Wireguard-based connection that it manages.

Tailscale has built a fantastic reputation and a fantastic product. The client is open source, however the Tailscale control plane service that all Tailscale users connect to is not.  There's a generous free tier for this management service, though it is understandably a subscription-based service. If you don't mind the SaaS model (that is currently free until you are connecting 100+ devices), you can just use Tailscale and avoid some of the complexity later in the post.

Tailscale covers that first VPN use case and makes it easier to do than has ever been possible. It also can cover the second use case, VPN privacy services, with a feature called exit nodes. Tailscale clients can advertise an exit node, which can then be used as the default route out to the internet by any other devices on the Tailnet that choose to use it. Let's say you have a cloud VPS somewhere, a laptop, and a smartphone all connected to a tailnet. If the VPS were advertising an exit node, the other devices could be out in the world with the exit node enabled, and their internet traffic would be sent via wireguard to the VPS where it would exit to the internet. This is effectively the same way my old "all traffic VPN tunnel" worked.  That's not a privacy VPN though. For that, Mullvad, the same privacy VPN vendor that I've been using, has partnered with Tailscale to offer Mullvad exit nodes. Tailscale users can pay for a Mullvad subscription through Tailscale, and then get access to Mullvad's endpoints as tailnet exit nodes... eliminating the hassle of having to switch between tunnels if you want to use a privacy VPN and need to access another Tailscale client. 

If the SaaS model or shared control plane aren't your cup of tea, there's still a way to have this functionality. There is an open source project called [Headscale](https://headscale.net/) which serves a self hosted Tailscale control server that you can connect the regular Tailscale client to. Tailscale (the company) is supportive of this too! Because Headscale is a separate control server, Tailscale's Mullvad integration and some features like tailnet sharing with Tailscale users are not available. A Mullvad exit node is possible with Headscale though, and that's the route I've gone with this. Pun intended.

#### Implementation
I make that work using a dedicated Linux system with the following:
 - Static routing, eliminating the conflicting dynamic routes that would otherwise be added.
     - Default route to `wg0` interface
        - Wireguard tunnel (`wg0` interface) connected to Mullvad (via native wireguard, not the Mullvad app)
     - Route Tailscale ranges to `tailscale0` interface
        - Tailscale (`tailscale0` interface) with exit node enabled, connected to Headscale
     - Route LAN range to the upstream gateway
     - Route Headscale endpoint addresses to the upstream gateway

Here's a Nix expression that shows specifics. Nix isn't required for this of course, but it *is* the best way!
```

{ lib, config, ... }: {

  boot = {
    kernel.sysctl = {
      "net.ipv4.ip_forward" = true;
      "net.ipv6.conf.all.forwarding" = true;
    };
  };

  # Static networking to keep things managed
  networking = {
    defaultGateway = {
      address = "192.168.1.1";
      interface = "eno1";
    };
    defaultGateway6 = {
      address = "fe80::ae1f:1234:1234:1234"; 
      interface = "eno1";
    };
    # nameserver for initial connections, /etc/resolv.conf gets overridden by tailscale service
    nameservers = [ "192.168.1.1" ];
    # Static interface configuration
    interfaces = {
      eno1 = {
        useDHCP = false;
        ipv4 = {
          addresses = [{
            address = "192.168.1.4"; # Static IPv4 address
            prefixLength = 24;
          }];
          routes = [
            {
              address = "192.168.1.0"; # LAN Subnet
              prefixLength = 24;
              via = "192.168.1.1"; #LAN gateway
            }
            {
              address = "52.x.y.z"; # Headscale EC2 instance public IPv4 address
              prefixLength = 32;
              via = "192.168.1.1"; # LAN gateway
            }
          ];
        };
        ipv6 = {
          addresses = [{
            address = "2601:602:1234:2::1234"; # Static IPv6 address
            prefixLength = 128;
          }];
          routes = [
            {
              address = "2600:1f14:2f74:1234:1234:1234:1234:1234"; # Headscale EC2 instance public IPv6 address
              prefixLength = 128;
              via = "fe80::ae1f:1234:1234:1234"; # LAN gateway
            }
          ];
        };
      };
      # Static routes to keep tailnet traffic on tailscale interface
      tailscale0 = {
        ipv4.routes = [
          {
            address = "100.64.0.0"; # CGNAT range https://tailscale.com/kb/1015/100.x-addresses
            prefixLength = 10;
          }
        ];
        ipv6.routes = [
          {
            address = "fd7a:115c:a1e0::"; # Tailscale ULA range https://tailscale.com/kb/1121/ipv6
            prefixLength = 48;
          }
        ];
      };
    };

    # Wireguard connection to privacy VPN service
    wg-quick.interfaces = {
      wg0 = {
        address = [ "10.x.y.z/32" "fc00:bbbb:1234:1234::6:1234/128" ]; # Internal Mullvad IP
        listenPort = 51820;
        privateKeyFile = "mullvad.key";
        peers = [
          {
            publicKey = "4k1234123412341234123412341234123412341234w="; # Mullvad server public key
            allowedIPs = [ "0.0.0.0/0" "::/0" ];
            endpoint = "138.x.y.z:51820"; # Mullvad server endpoint
            persistentKeepalive = 25;
          }
        ];
      };
    };

    # Firewall configuration using nftables
    nftables.enable = true;

    firewall = {
      enable = true;
      allowedTCPPorts = [ 22 ]; # ssh local access outside of Tailscale
      checkReversePath = "loose";
      trustedInterfaces = [ "tailscale0" ];
      allowPing = true;
    };

    nat = {
      enable = true;
      externalInterface = "wg0";
      internalInterfaces = [ "tailscale0" "eno1" ];
    };
  };

  # Enable Tailscale with exit node capability
  services.tailscale = {
    enable = true;
    useRoutingFeatures = "server";
  };

  # GRO forwarding for exit node
  # https://tailscale.com/kb/1320/performance-best-practices#ethtool-configuration
  systemd.services.tailscale-network-optimizations = {
    description = "Apply network optimizations for Tailscale";
    after = [ "network.target" "tailscale.service" ];
    wantedBy = [ "multi-user.target" ];
    path = [ pkgs.iproute2 pkgs.ethtool ];
    script = ''
      NETDEV=$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")
      ethtool -K $NETDEV rx-udp-gro-forwarding on rx-gro-list off
    '';
    serviceConfig = {
      Type = "oneshot";
      RemainAfterExit = true;
    };
  };

  # Wait for network stability
  systemd.services.tailscaled = {
    after = [ "network-online.target" ];
    wants = [ "network-online.target" ];
    serviceConfig = {
      ExecStartPre = "${pkgs.coreutils}/bin/sleep 5";
    };
  };
};
}
```

At the time of writing, this serves as one of three available exit nodes on my Headscale-managed tailnet. My phone and laptop are usually configured to use this until something I need blocks privacy VPN users. In that case, I can disable usage of the exit node in my tailscale client. Traffic now flows out the default gateway, and I still have the benefit of access to everything on my tailnet, as well as Tailscale DNS. DNS, by the way, goes through [NextDNS](https://nextdns.io/) in my current confguration, regardless of exit node use. If I'm not at home, but I want my internet traffic to go through my home IP, I select the exit node that my home router (just another client from Tailscale's perspective) advertises. There's a third option that I have running on a Raspberry Pi at a family member's house a few states over. Using these residential ISP exit nodes still provides the benefit of wireguard-protected internet traffic between the device and the exit node, making this option perfect for using the internet from untrustworthy wifi networks, if the Mullvad exit node isn't a better fit for some reason. This all means that anywhere I have internet connectivity, I'm able to connect to all of my computers, and route my internet traffic securely out some of the other tailnet devices, including the Mullvad-connected solution described in this post. So far, it's been working flawlessly for me and I highly recommend adopting a similar setup. I plan to write a follow-up post on the specifics of my Headscale implementation. For now, I'll mention that it runs on a free-tier AWS EC2 instance, and that allows me to have no exposed ports on my home router. I welcome feedback in comments.

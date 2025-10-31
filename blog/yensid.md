---
title: A first look into yensid
description: How yensid solves my very specific use case for Nix remote builders
tags:
- nix
published_date: 2025-10-31 16:42:43.285514044 +0000
layout: default.liquid
is_draft: false
---
# Yensid

A few days ago, [garnix](https://garnix.io) published a tool called [yensid](https://github.com/garnix-io/yensid).
The tool provides an approach to load balance Nix remote builds globally and with a custom algorithm. It is a set of NixOS modules wrapping [HAproxy](https://www.haproxy.org/), the modules provide configuration for the proxy, a CA and a builder.

## About Nix remote builds

Remote builds are something a tool like Nix could be extremely good at. Unfortunately the current implementation in CppNix doesn't utilize the full potential.
In addition to being somewhat unreliable, in my experience, the build distribution algorithm could be improved upon, and in my opinion, should be configurable by the user.

## My use case

My use case for remote builds is very specific. I'm almost exclusively working on my laptop, which is quite powerful on its own. This means that often times it doesn't make sense to distribute builds, especially if my internet connection is slow.

When using CppNix's default remote builders, I often find myself having to specify `--builders ""`, which is not ideal. I'd like to be able to specify some arbitrary logic to decide whether to use builders or build locally. This could be achieved by editing the `/etc/nix/builders` file on the fly, but this doesn't seem like a _Nix way_ of doing things.

This is where yensid comes in, since it allows me to specify my own load balancing algorithm in Lua code, it should be more than good enough for my purposes.

## Implementation

My initial plan was to register both the local machine and the remote builder(s) as builders to yensid and then if the Lua script decides that using builders is not desired, it would redirect all workload to the local builder.
This idea, however, did not work at all, due to the fact that using `localhost` as a remote builder causes Nix builds to deadlock[^1].

The second, a bit more hackier, approach is to have yensid refuse the connection whenever using a remote builder is not desired, this effectively tells Nix to build the derivation locally, while printing an ugly error.

For now, instead of looking into figuring out the link speed between machines, I've configured the proxy to refuse connections if the device is not connected via ethernet.

## Configuration

I've listed my configuration for the different parts below. I noticed that the builder module for yensid is very dependent on the presence of a CA, which I don't have. Instead of using it, I opted into manually creating the `builder-ssh`-user. In addition to this, you'll also have to add the host to the known-hosts of the root user on the client. If you have multiple builders, you'll have to add all of their keys to the known hosts file.

### Yensid proxy (running on localhost)

```nix
config.yensid.proxy = {
  enable = true;
  builders.builder.ip = "<IP>";
  loadBalancing.strategy = "custom";
  loadBalancing.luaFile =
    let
      nmcli = "${pkgs.networkmanager}/bin/nmcli";
    in
    pkgs.writeText "custom-balanging.lua" ''
      local function is_ethernet_connected()
          -- Use nmcli to check for active ethernet connections
          local handle = io.popen("${nmcli} -t -f TYPE,STATE device 2>/dev/null")
          local result = handle:read("*a")
          handle:close()
          
          -- Check if any ethernet device is connected
          for line in result:gmatch("[^\n]+") do
              local device_type, state = line:match("^([^:]+):([^:]+)")
              if device_type == "ethernet" and state == "connected" then
                  return true
              end
          end
          
          return false
      end

      local ethernet_status = nil
      local last_check = 0

      core.register_fetches('custom-strategy', function(txn)
          local now = os.time()
          if ethernet_status == nil or (now - last_check) >= 5 then
              ethernet_status = is_ethernet_connected()
              last_check = now
          end
          
          return ethernet_status and "builder" or nil
      end)
    '';
};
```

### Yensid builder running on each/a remote builder

```nix
# A piece of config stolen from the yensid builder module
users.users.builder-ssh = {
  isSystemUser = true;
  shell = pkgs.bash;
  group = "users";
  openssh.authorizedKeys.keys = [
    "<USER SSH PUBKEY>"
  ];
  extraGroups = [ "wheel" ];
};
```

### Nix configuration (localhost)

```nix
nix.buildMachines = [
  {
    sshUser = "builder-ssh";
    sshKey = "<PATH-TO-SSHKEY>";
    protocol = "ssh-ng";
    hostName = "localhost";
    systems = [ "x86_64-linux" ];
  }
];
```

## Problems with the presented solution

The main problem with the presented solution is that it's not very easy to override the decision of whether to use remote builders or not. I can of course still use `--builders ""` from earlier to opt out of remote builds, but there's not an easy way to opt into them.

The solution also forces me to change the port where I run OpenSSH on my laptop, since port 22 will be occupied by yensid/haproxy.

## Final thoughts

I'm overall somewhat happy with the outcome of this, but I do not think that this kind of solutions are the final solution for Nix remote builds.
In the future, I would be interested in looking into replacing the [remote building logic](https://github.com/NixOS/nix/blob/master/src/nix/build-remote/build-remote.cc) in CppNix, with a custom implementation. This, however, would be significantly more effort than setting up yensid.

Solving Nix remote builds once and for all would require the joint effort of CppNix developers in allowing custom (global) logic for build distribution, and a third-party project that would implement such a logic. Allowing externally computed and global (as in knowing the state of the entire cluster) build distribution, would eventually get us significantly closer to optimal Nix build distribution. A sophisticated build distribution algorithm could even use a database of historical build times of packages from Hydra to estimate their sizes.

[^1]: [https://github.com/NixOS/nix/issues/2029](https://github.com/NixOS/nix/issues/2029)

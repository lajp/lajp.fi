---
title: NixOS port allocation
description: An automatic port allocation system for NixOS that prevents collisions
published_date: 2026-01-31 16:17:13.717981319 +0000
layout: default.liquid
is_draft: false
---
# NixOS port allocation

I was recently implementing a new service to [my NixOS configuration](https://github.com/lajp/nix) with the help of [Claude Code](https://claude.com/product/claude-code).

The conversation went something like this:
```
me: implement service X on host Y
claude: *produces plan where service X runs on 3000*
me: port 3000 is surely already occupied, use another one
claude: *updates the plan to use port 3004 instead*
me: *thinking*: ok, it checked which ports were in use and found that 3004 is free
me: ok, proceed to implementation
```

After deploying the change and testing it out (in production, as one does), I noticed that port 3004 in fact wasn't free on that host and there were now two services racing to reserve that post. At first, I was surprised and disappointed that Claude didn't actually check which ports were occupied and started updating CLAUDE.md to tell it to always checked whether a port is occupied. In the midst of doing this, I realized that this is a problem that could (and should!!) be solved systematically.

## Systematic solution drafts

Given that we, at the time of evaluating the configuration, already know which services allocate which ports, we should be able to find conflicts and raise evaluation errors when we do.
While this is technically possible to achieve this, it's rendered unviable by the fact that the options for specifying ports are not standardized across nixpkgs, some service might have a simple `port` option, while another one expects it to be given as an environment variable.

While thinking about this, it appeared to me that for most of the services, it doesn't matter which port they use as long as it's not occupied. The services are going to be behind an nginx reverse proxy anyways.
I started thinking of a system that could, in addition to collision checking, automatically allocate ports for services.

My first idea was to have a function `getPort` that would take in some state (like `config`) and return an unused port.
This, however, proved to be an unwise approach and instead, after some iteration, I ended up with the following.

## How the current approach is used

In the implemented port allocation mechanism, each service can request either a specific port or then just some port.
The request is done via `config.lajp.portRequests.<name>` where `<name>` is the name of the port allocation (usually the name of the service).

The allocation mechanism produces produces the `config.lajp.ports` map that includes the realized port allocation for each service name.

An example usage could be as follows (hedgedoc.nix):

```nix
# details omitted
config = mkIf cfg.enable {
    # true means "allocate any port"
    lajp.portRequests.hedgedoc = true;

    # ...
    services.hedgedoc.settings.port = config.lajp.ports.hedgedoc;
    # ...
};
```

If a specific port is required, it can be requested by changing `true` to the port number:

```nix
lajp.portRequests.ssh = 22;
```

If two services attempt to allocate the same port, the evaluation reaches the following error:
```md
error:
Failed assertions:
- Duplicate explicit port requests: 22 22
- Port collision detected: 22 22
```

This saves a lot of time in future debugging (speaking out of experience).

The implementation details are available in [modules/system/ports.nix](https://github.com/lajp/nix/blob/main/modules/system/ports.nix) of my config.

## Caveats

An important thing to note out before implementing this to your setup is that the port allocations are not static.
This means that existing port allocations might change when adding a new service.
This is not a problem for me since I don't care about the ports used, nor do I rely on them in other places.
If a port allocation is accessed from another system or outside of the NixOS config altogether, it should be made static by requesting a specific port.

Another, more obvious caveat is that this system won't be of benefit unless it is used throughout the config. But luckily using it is just 2 extra lines :)).

I really can't think of any other good reasons to not use a system like this.
Manually managing the ports of 10s or 100s of services is just not worth it.

Feel free to steal my code, I hope it benefits you!

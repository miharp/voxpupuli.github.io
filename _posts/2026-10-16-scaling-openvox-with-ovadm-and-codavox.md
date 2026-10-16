---
date: 2026-10-16
layout: post
title: "Same code, every server: scaling OpenVox with ovadm and codavox"
github_username: miharp
excerpt: "Two tools and a few optional modules for the jobs open-source Puppet leaves to you: ovadm installs the servers from your workstation, the modules keep them configured, and codavox gets your code onto every server and proves which version each one runs."
---

<link href="https://cdn.jsdelivr.net/npm/asciinema-player@3.17.0/dist/bundle/asciinema-player.css" rel="stylesheet" integrity="sha384-05cmIVRzN7mR7nmqajPpGPUPqJ5VyTAGHL1xJuiGWfhpWDp5hEfBk50kr21f3ILM" crossorigin="anonymous">
<style>.cast { margin: 1.25rem 0 1.75rem; }</style>

[OpenVox](https://voxpupuli.org/openvox/) is the community-maintained fork
of Puppet. Running one OpenVox server is well documented. The work starts
when one is no longer enough: a CA server, which signs every node's
certificate, with compilers beside it that share the work of building
catalogs. Puppet Enterprise sets that up for you. With open source, someone
has to install the CA server and each compiler, join the compilers to the CA
with their own CA switched off, keep every server configured, and get the
same version of your Puppet code onto all of them.

That last job is the one that bites. You push a change, one compiler serves
it, another still serves yesterday's code, and nothing tells you which is
which. Puppet Enterprise handles this with Code Manager and file sync, and
because file sync tells the server which version it is serving, static
catalogs work there out of the box: each catalog records the exact version
of the code it was built from. OpenVox Server has the same setting, turned
on, but nothing that supplies the version, so out of the box it does
without. Agents then ask the server about every file they manage from a
module, one request per file on every run, and get each file from whatever
code the server has at that moment, which may not be the code their catalog
was built from.

I wrote two tools for those jobs, [ovadm](https://github.com/miharp/ovadm)
and [codavox](https://github.com/miharp/codavox), and a few optional Puppet
modules that go with them. All of them are young, with version numbers
starting at 0, and so far they have only run on test machines like the ones
in the recordings. They live under my name for now; my plan from the start
has been to donate them to Vox Pupuli if there is interest. This post walks
through them in the order you would use them. Each section has a short
recording, made on fresh virtual machines with nothing installed.

A few terms first:

- **Server**: OpenVox Server. It builds each node's catalog, the list of
  everything Puppet should apply there. It also runs the certificate
  authority (CA) that signs every node's certificate.
- **Agent**: the Puppet agent. It runs on every managed node, the servers
  included, asks a server for its catalog and applies it.
- **Compiler**: an extra server that only builds catalogs, so a large site is
  not limited by one machine. It trusts the main server's CA.
- **Control repo**: the git repository holding your Puppet code. Each branch
  becomes an environment on the server, such as `production`.
- **r10k**: the tool that deploys a control repo. It checks out each branch
  as an environment on the server and downloads the modules listed in the
  branch's `Puppetfile`.

## 1. Installing: ovadm

ovadm is a module for
[OpenBolt](https://github.com/OpenVoxProject/openbolt), which runs commands on
other machines over SSH. It is modelled on
[peadm](https://github.com/puppetlabs/puppetlabs-peadm), Puppet's Bolt module
for installing and upgrading Puppet Enterprise, and does the same job for
OpenVox. Everything here runs from your workstation, the
machine where Bolt is installed. The servers need no software beforehand,
but Bolt must be able to reach them over SSH as root, or as a user who can
use sudo, which Bolt's `run-as` setting takes care of. Add ovadm to a Bolt
project, and `bolt project init` creates one if you have none yet:

```console
bolt module add miharp-ovadm
```

Bolt writes it into the project's `Puppetfile` and installs it.

Then one Bolt plan installs the server, and its compilers too if you name
them:

```console
bolt plan run ovadm::install server_host=puppet.example.com compiler_hosts=compiler01.example.com,compiler02.example.com
```

The Bolt plan checks the machine first: memory, Java, the clock, port 8140. Then
it adds the OpenVox package repository, installs the server, and creates the
CA the way Puppet Enterprise does, with a root and an intermediate
certificate that last 15 years. Left to itself, the server creates a single
CA certificate that expires after five years, on the same day as your first
agents' certificates. Nobody writes that date down. Five years later, every
agent stops trusting the server at the same moment, Puppet stops working
everywhere at once, and whoever is on call learns what a certificate
authority is before their first coffee.

Later, two more Bolt plans grow the setup. `add_compiler` adds compilers to
a running server, and `add_agent` brings in the nodes you actually want to
manage, pointed at the server or at a compiler:

```console
bolt plan run ovadm::add_compiler server_host=puppet.example.com compiler_hosts=compiler01.example.com
bolt plan run ovadm::add_agent server_host=puppet.example.com agent_hosts=web01.example.com,web02.example.com
```

Each one installs the packages, has the certificate signed, and runs the agent
once, so the new node has applied its first catalog when the Bolt plan finishes.
ovadm also writes a
[role](https://docs.openvoxproject.org/openvox/latest/ssl_attributes_extensions.html#puppet-specific-registered-ids)
into each server's and compiler's certificate. That matters in section 3:
every agent has a certificate from your CA too, so codavox does not hand
your code to just any of them, only to the machines you allow, and the
easiest way to allow your compilers is by that role. It is only a label in
the certificate, separate from the role classes in your control repo that
decide what Puppet applies to a node.

There are Bolt plans for upgrades and status checks as well. What ovadm does not
do is stay around. It installs, then gets out of the way, and nothing it does
runs on a schedule.

<div id="act1" class="cast"><p><a href="{{ page.url | absolute_url }}#act1">▶ Recording, act 1: ovadm installs OpenVox Server, its CA and a compiler (plays on voxpupuli.org)</a></p></div>

## 2. Keeping it configured: the supporting modules

Once the server is installed, it is a node like any other, and the Puppet way
to look after a node is with Puppet. ovadm does not mind what you use: your
own code, [theforeman-puppet](https://forge.puppet.com/modules/theforeman/puppet),
or these three modules, which are written to pick up exactly where ovadm stops.

- **[openvox_server](https://forge.puppet.com/modules/miharp/openvox_server)**
  manages the server itself: the package, the CA settings, the web server
  settings, compiler mode and the service.
- **[openvox_tune](https://forge.puppet.com/modules/miharp/openvox_tune)**
  sizes the server for its machine. Out of the box the server uses 2 GB of
  memory and at most four workers (JRuby instances, each compiling one
  catalog at a time), however big the machine is. A Bolt plan recommends
  values for your hardware, and the class applies the ones you put in Hiera.
- **[openvox_agent](https://forge.puppet.com/modules/miharp/openvox_agent)**
  looks after the agent itself on every node. You choose a collection, such
  as `openvox8`, and a version or `latest`. On each run it points the node at
  that collection's package repository, installs or upgrades the agent, and
  leaves the agent service running or stopped as it found it. It also moves
  nodes from Puppet 7 to OpenVox, keeping their certificates and settings,
  and refuses moves that would break a node.

The control repo in the recordings is the one step 3 of the
[OpenVox getting started guide](https://docs.openvoxproject.org/openvox/latest/getting_started.html#step-3-set-up-a-control-repository)
creates from the OpenVox control repo template, with a few modules and three
roles added ([miharp/openvox-demo-control](https://github.com/miharp/openvox-demo-control)).
Each node gets its role class in `manifests/site.pp`, by name:

```puppet
node 'puppet.example.com' {
  include role::primary
}
node /^compiler\d+\.example\.com$/ {
  include role::compiler
}
```

The main server's role is one line:

```puppet
# site-modules/role/manifests/primary.pp
class role::primary {
  include openvox_server
}
```

A compiler is the same module with the CA switched off, in its node data,
the file the template's `hiera.yaml` reads for each node by its certificate
name. The same file holds the size openvox_tune's Bolt plan recommended for
it:

```yaml
# data/nodes/compiler01.example.com.yaml
openvox_server::ca: false
openvox_server::ca_server: 'puppet.example.com'

# OpenVox Server sized for this machine by openvox_tune's plan.
openvox_server::max_active_instances: 7
openvox_server::heap_mb: 4096
openvox_server::reserved_code_cache_mb: 1024
```

The compiler in the recordings has 8 CPUs and 16 GB of memory. Out of the
box, OpenVox Server would use four workers and 2 GB of that; the plan
recommends seven workers and 4 GB:

```console
bolt plan run openvox_tune::tune --targets puppet.example.com,compiler01.example.com hiera=openvox_server
```

Run against both servers together, it also knows the main server has a compiler beside it, and sizes that one
down, from three workers to two and from 2 GB to 1.5 GB, because the
compiler does the compiling.

The first Puppet run after ovadm is the test of the hand-over. It should
change the size and nothing else, and it does: the module agrees with
everything ovadm set up, and from here on it is in charge whenever Puppet
runs. Nothing runs Puppet on a schedule yet, though: ovadm runs the agent
once, and the recordings run Puppet by hand. openvox_agent can keep the
agent service running. openvox_server already declares it on the servers, so
only your other nodes need `include openvox_agent` in their role; then turn
the service on for every node in Hiera. Left unset, openvox_agent leaves the
service as it found it.

```yaml
# data/common.yaml
openvox_agent::service_ensure: 'running'
openvox_agent::service_enable: true
```

<div id="act2" class="cast"><p><a href="{{ page.url | absolute_url }}#act2">▶ Recording, act 2: openvox_tune sizes the servers and Puppet takes over (plays on voxpupuli.org)</a></p></div>

## 3. Getting code to every server: codavox

Even with one server, deploying code has a catch. r10k changes the code in
place, so a catalog built in the middle of a deploy can mix old and new code,
and an agent that is mid-run can get its catalog from the old code and its
files from the new. Add a compiler and there are also two copies of your code
to keep in step. The usual answers both work until they don't:

- **r10k on every compiler**, usually triggered by a webhook when you push. A
  compiler that is down or restarting when the webhook fires misses that
  deploy and keeps the old code until the next push. And if the Puppetfile
  lists a module as `:latest`, or follows a git branch, compilers that deploy
  at different times can fetch different code from the same control repo.
- **One copy shared with every compiler** over a network file system such as
  NFS. Every compiler then sees r10k's changes as they happen, mid-deploy
  included, and the file server becomes one more thing every compiler depends
  on: when it is down, so are they.

Either way, nothing tells you which server has which version.

This is where static catalogs come in. Each catalog records a `code_id`, the
exact version of the code it was built from, and the agent fetches files
from that version only. It also carries the details of the files it copies
from your modules, so agents stop asking the server about each one, one
request at a time, on every run. That adds up fast. At a previous job we had
over 300 modules, many of them managing dozens of files, and those requests
put a noticeable load on the compilers. OpenVox Server has the hook for
supplying that version but ships nothing that fills it; codavox fills it.

It works like this:

1. You deploy on the main server with `codavox deploy production`, by hand,
   from CI, or from the push webhook that `codavox::deploy_server` sets up.
   Only the main server needs to hear about a push: the compilers poll it
   every 30 seconds, so each one catches up to the latest version on its
   own, even if it was down when you pushed. codavox runs r10k once,
   exactly as you do now.
2. It takes a fingerprint of the result, the `code_id`, and packs that
   version into an ordinary `.tar.gz` file for the compilers to download.
   The main server keeps the current one, which you can open with `tar` to
   see exactly what is deployed.
3. When a compiler next checks in, it downloads the new version, checks the
   fingerprint, and switches to it by repointing a symlink, a single step, so
   the server sees either the old version or the new one, never a half-copied
   one. The old version stays
   on disk for agents still using it, and a compiler that was switched off
   catches up when it comes back. If the main server goes down, compilers keep
   serving the version they have; only new deploys wait.
4. Every catalog a server builds carries the `code_id` of the version it
   used. The environment and the first 12 characters of the `code_id` are
   also the catalog's version in the agent's output, `Applying configuration
   version 'production-821a20908b17'`, the same as the `ENVIRONMENT` and `CODE_ID`
   columns below.

The label is enough to get back to git. On the main server,
`codavox provenance production-821a20908b17` prints the commit that produced it.

None of this needs a compiler. On a single server,
[`codavox::primary`](https://github.com/miharp/puppet-codavox#a-primary)
runs codavox's publisher and its agent (the part that downloads code, not
the Puppet agent) next to OpenVox Server, and that server gets
the same one-step switch, static catalogs and catalog labels. Adding
compilers later changes nothing on it.

Compilers prove who they are with the certificates they already have, and
the role ovadm wrote into them is their permission. There are no extra
passwords to manage. Compilers connect to the main server on port 8150, so
open that if you run a firewall.

The [codavox module](https://forge.puppet.com/modules/miharp/codavox) sets
it up. Adding it to a running setup is a small change to the control repo
([the commit merged in the recording](https://github.com/miharp/openvox-demo-control/commit/d2fc4b84b8162962ffed180a3ea5f59b10a73dfc)):
the module and its two dependencies in the `Puppetfile`, a few settings in
Hiera, a few lines in `scripts/config_version.sh` for the catalog label
described above (without them, catalogs keep r10k's label), and three lines
in the roles. The main server gets
`include codavox::primary`, each compiler gets `include codavox::agent` and
`include codavox::server`, and Hiera says where things are:

```yaml
codavox::basedir: '/etc/puppetlabs/code/environments'
codavox::publish_allow_roles: ['openvox_compiler']
codavox::agent_publisher: 'https://puppet.example.com:8150'
```

On the main server, the module waits until the first version has arrived
before it points the server at it, so the server is never left with an empty
code directory. That is why, in the recording, the main server needs one more
Puppet run after codavox is installed. That run makes the switch, and its own
catalog still carries r10k's label, `puppet-production-d2fc4b84b81`, because
it was built just before. The run after it carries codavox's,
`production-991bc15eb41b`.

The recording merges that change, installs codavox with one Puppet run on
each server, and shows both servers on the same `code_id`. Then it changes
the message in `/etc/motd`, a file agents get from a module, deploys it with
codavox, and adds an agent with ovadm. The compiler's access log shows the
agent fetching that file by the same `code_id` the compiler reports, and the
main server names the commit behind the label.

<div id="act3" class="cast"><p><a href="{{ page.url | absolute_url }}#act3">▶ Recording, act 3: codavox gets the same code onto every server (plays on voxpupuli.org)</a></p></div>

If you skip the recording, this is the command that shows where every server
stands, after the second deploy:

```console
$ codavox compilers
COMPILER                ENVIRONMENT  CODE_ID       COMMIT        LAST POLL
puppet.example.com      production   821a20908b17  0c5e36ade88f  16s ago
compiler01.example.com  production   821a20908b17  0c5e36ade88f  26s ago
```

Both servers run the same version of `production`, and the commit beside it
is the one I pushed. `LAST POLL` is how long ago each one last checked in
with the main server. The `code_id` is a fingerprint of the code alone, so the
same code gives the same `code_id` on any machine on any day. In rehearsals
for this post, VMs rebuilt from scratch hours apart produced the same
`code_id`s for the same code.

## Trying it

They are published under my name on the Forge for now. If you would like to
see them under Vox Pupuli, say so; that is where I would like them to end
up. And if you try them, I would like to hear what works and what doesn't.
Open an issue on any of the repositories:

- [ovadm](https://github.com/miharp/ovadm)
- [openvox_server](https://github.com/miharp/puppet-openvox_server),
  [openvox_tune](https://github.com/miharp/puppet-openvox_tune),
  [openvox_agent](https://github.com/miharp/puppet-openvox_agent)
- [codavox](https://github.com/miharp/codavox) and its
  [module](https://github.com/miharp/puppet-codavox)
- [openvox-demo-control](https://github.com/miharp/openvox-demo-control), the
  control repo from the recordings

The recordings run at real speed, but any wait longer than four seconds is
cut; the ovadm Bolt plan in the first one took just over three minutes for
the server and the compiler. In the Puppet runs, informational lines other
than the catalog version, and file diffs other than openvox_tune's change to
`JAVA_ARGS`, are left out, and so are Bolt's JSON summary at the end of a
plan and openvox_tune's note about the `environment_timeout` setting.

<script src="https://cdn.jsdelivr.net/npm/asciinema-player@3.17.0/dist/bundle/asciinema-player.min.js" integrity="sha384-s55nTYAdrPwGWmKKQ1lCnoB8H9LbqmsXsqqqPAHK2+T5h9IfI2dTXTDXJcZnySJD" crossorigin="anonymous"></script>
<script>
  // Each recording slot holds a link for feed readers and browsers without
  // JavaScript; the player replaces it once its script has loaded.
  if (window.AsciinemaPlayer) {
    ['act1', 'act2', 'act3'].forEach(function (act) {
      var el = document.getElementById(act);
      el.textContent = '';
      AsciinemaPlayer.create(
        '{{ "/static/asciinema/codavox/" | relative_url }}' + act + '.cast',
        el,
        { idleTimeLimit: 4, fit: 'width', poster: 'npt:0:3.3', preload: true, controls: true }
      );
    });
  }
</script>

+++
title = "puppet lessons learned"
date = "2015-04-13"
slug = "2015/04/13/puppet-lessons-learned"
Categories = []
+++

Over the past couple of years my team has iterated several times on the proper way of managing systems using Puppet. For a while it was a gigantic time sink while we tested and prototyped several different appraoches to configuring things with many frustrating failures. This post will be an exploration of some of the lessons learned.

<!-- more -->

## Lesson #1: Puppet is not deterministic

Yup, that's right. The tool you're trying to use to get all your servers to a deterministic state isn't very deterministic in resolving that state. Many times we've had to hack around adding all sorts of dependancies and ordering to puppet manifests to get it to do the right thing.

![Oh yeah deterministic baby](http://sahajapower.files.wordpress.com/2008/05/dice.jpg)

## Lesson #2: Node classification with the dashboard sucks.

One of our initially successful approaches to our puppet deployment made use of the dashboard for all of our classes and variables. This quickly spiralled out of control as managing different environments was next to impossible as there was no clear way to propagate changes to the ENC to the various environments. Eventually this led us to completely scrap the dashboard.

![dashboard sucky](http://weknowmemes.com/wp-content/uploads/2011/11/rage-face-dash-board.jpg)

## Lesson #3: Hierarchical configuration is nice

Hiera is your friend. But managing complex hierarchies is hard. Unless you can extend hierarchies with imports. Cue the custom hiera yaml backend built by my esteemed co-worker Fabio (see code section below for link). Once installed, this allows us to specify complex hierarchies and split out the configuration in an extensible manner.

![hierarchy](http://www.markstivers.com/wordpress/comics/2012-08-18-hierarchy-of-nee.gif)

## Lesson #4: Package managers are sometimes stupid.

I had to write a custom service handler to prevent packages from auto-starting services on package installation. This is extremely annoying in my book as almost all services are misconfigured with unsafe boilerplate configurations by package maintainers. I get the whole convenience factor, but from a security perspective it's horrible. Not sure if this one is ubuntu specific, probably not...

![apt...](https://encrypted-tbn3.gstatic.com/images?q=tbn:ANd9GcTFzoc6hOiNYebIkF-x7nKh4u2X_LzVguGQPdyhPKZB25U8m9eByA)

## Lesson #5: Configure everything through class parameters pulled in via hiera hash merging. Let the node tell puppet what it is.

This one is nice. We only have one file in /etc/puppet/manifests. Everything is done from hiera or facter. Taking this approach, you can have a single instance of puppet running to manage several different environments.

![mind blown](http://www.reactiongifs.com/r/2011/09/mind_blown.gif)

## Lesson #6: Store everything in Git, make your puppet master serve from git repo

No brainer here. Much simpler to manage things if puppet's config is running from a git repo checkout.

## Lesson #7: Puppet is riddled with memory leaks. 

Run it under apache/mod_passenger. Even then it's a hog. Make sure you've got MaxRequestsPerChild to a fairly lowish number to try to mitigate it eating up all the RAM. nom nom nom.

![leaky](http://cdn.meme.am/instances/500x/35849520.jpg)

## Lesson #8: Puppet classes/templates should include ALL possible options for the object being configured

Do something right, or don't do it at all. The puppet forge is riddled with modules that do a crappy job of configuring services, or only expose a small number of possible settings. A pattern that I've come to often use is where possible make the template configure everything. Example below.

![dammit do it right](http://memeshare.net/memes/preview/9/8558.png)

## Lesson #9: Ansible is simple and deterministic.

This is possibly one of the most painful lessons given the time we put into Puppet. I haven't personally spent a ton of time with Ansible, but I'm seriously reconsidering my time investment in Puppet. If in doubt, try Ansible first. There, I said it.

--

And now, to some code. Here's a full configuration example to demonstrate how this can all come together.

Puppet master

/etc/puppet/puppet.conf
```ini 
[main]
logdir=/var/log/puppet
vardir=/var/lib/puppet
ssldir=/var/lib/puppet/ssl
rundir=/var/run/puppet
factpath=$vardir/lib/facter

[master]
ssl_client_header = SSL_CLIENT_S_DN
ssl_client_verify_header = SSL_CLIENT_VERIFY

enviroment = development
certname   = puppet-master.yourdomain.com
autosign   = true
```

``` plain /etc/puppet/manifests/site.pp
node "default" {
    if ( hiera('classes', null) != null ) {
        hiera_include('classes')
    }
}
```

The custom yaml backed for puppet can be found here: https://github.com/instaclick/hiera-ic-yaml

/etc/puppet/hiera.yaml
```yaml 
+++
:backends:
  - 'ic_yaml'

:hierarchy:
    - %{::environment}/%{::hostname}
    - %{::environment}/%{::role}
    - %{::hostname}
    - %{::role}

:ic_yaml:
  :datadir: '/etc/puppet/nodes'
  :parameters_key: 'parameters'
  :imports_key: 'imports'
  :cacheable: false

:logger: console
:merge_behavior: 'deeper'
```

Now let's configure collectd for a node. First, we install the puppet content:

```bash
cd /etc/puppet/modules
git clone --recursive https://github.com/marksteele/collectd-puppet.git
```

First, a base role that all our boxes inherit:

/etc/puppet/nodes/base-role.yaml

```yaml 
imports:
    - "common/collectd.yaml"
#    - "common/ntp.yaml"
#    - "common/snmp.yaml"
#    - "common/dnsmasq.yaml"
#    - "common/resolvconf.yaml"
#    - "common/instaclick::apt.yaml"
#    - "common/instaclick::sshd.yaml"
#    - "common/instaclick::users.yaml"
#    - "common/instaclick::sysctl.yaml"

classes:
    - collectd
#    - ntp
#    - apt
#    - snmp
#    - dnsmasq
#    - resolvconf
#    - instaclick::apt
#    - instaclick::bash
#    - instaclick::sshd
#    - instaclick::users
#    - instaclick::sysctl
#    - instaclick::timezone
#    - instaclick::logrotate
```

Common configuration for all boxes that have collectd:
```bash
mkdir /etc/puppet/nodes/common
cat <<'EOF' >/etc/puppet/nodes/common/collectd.yaml
collectd::config:
  'Hostname': "%{::hostname}"
  'FQDNLookup': 'false'
  'Interval': 10
  'Timeout': 2
  'ReadThreads': 5

collectd::core_plugins:
  'syslog': ~
  'conntrack': ~
  'contextswitch': ~
  'entropy': ~
  'load': ~
  'memory': ~
  'disk':
    'Disk': '/^[vhs]d[a-f][0-9]?$/'
    'IgnoreSelected': 'false'
  'df':
    'MountPoint': '/'
  'interface':
    'Interface': 'lo'
    'IgnoreSelected': 'true'

collectd::perl_engine_config:
  'Globals': 'true'

collectd::perl_plugin_config:
  'IncludeDir': '/usr/lib/collectd'
  'BaseName': 'Collectd::Plugins'

collectd::perl_plugins:
  'CPUSummary': ~
  'AmqpJsonUdp':
    'Buffer': '8196'
    'Host': "%{::monitoring_ip}"
    'Port': '9999'
    'Prefix': "%{::role}"
'EOF'
```


Setup role specific config (least specific). Let's imagine our role is 'webserver', and we want to install collectd as a service managed by puppet.

```bash
mkdir -p /etc/puppet/nodes/{production,development,qa}/webserver
cat <<'EOF '>/etc/puppet/nodes/webserver.yaml
imports:
    - "base-role.yaml"
    - "webserver/collectd.yaml"
'EOF'
```

We'll add to this the generic role config for this role for collectd

```bash
mkdir /etc/puppet/nodes/webserver
cat <<'EOF' >/etc/puppet/nodes/webserver/collectd.yaml
collectd::core_plugins:
  'apache':
    'apache80':
      'URL': 'http://localhost/server-status?auto'
'EOF'
```

We want our parameters for various environments to potentially be different. Let's do that here:
```bash
cat <<'EOF' >/etc/puppet/nodes/production/webserver.yaml
parameters:
    monitoring_ip: "1.2.3.4"
'EOF'
cat <<'EOF' >/etc/puppet/nodes/development/webserver.yaml
parameters:
    monitoring_ip: "127.0.0.1"
'EOF'
cat <<'EOF' >/etc/puppet/nodes/qa/webserver.yaml
parameters:
    monitoring_ip: "3.4.2.1"
'EOF'
```

Managed node

/etc/facter/facts.d/node.yaml

```yaml 
role: "webserver"
environment: "production"
```

So what we've implemented here is a bottom up approach, where we first load the least specific configs, working our way backwards to more specific configs by merging them together. We have the ability to specificy different classes or parameters based on roles, environments, or specific nodes. Parameters are loaded from both hiera and facter, allowing for maximum flexibility.

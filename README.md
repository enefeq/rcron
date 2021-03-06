[![Github Version](https://badge.fury.io/gh/EvanK%2Frcron.svg)](http://badge.fury.io/gh/EvanK%2Frcron)
[![Build Status](https://travis-ci.org/EvanK/rcron.svg?branch=master)](https://travis-ci.org/EvanK/rcron)

# Rcron

Cron job redundancy and failover for a group of machines.

This is a python reimplementation of [Benjamin Pineau's rcron tool](https://code.google.com/p/rcron/), and is intended as a drop-in replacement.

### Fork notes

This repo was forked from [https://github.com/EvanK/rcron]. The functionality remains the same, but the project layout has been changed to work with ansible-galaxy roles.

### Requirements

A POSIX system with a Python 2.4+ interpreter, and a separate tool to maintain state across machines -- such as [keepalived](http://www.keepalived.org/).

### Installation

The rcron script can live anywhere in the path. For our purposes, we'll expect it to live in `/bin`:

	cp ./src/rcron.py /bin/rcron
	chown root:root /bin/rcron
	chmod 0755 /bin/rcron

It expects a configuration file, either at `/etc/rcron/rcron.conf` or specified with the `--conf` flag:

	mkdir -p /etc/rcron
	cp ./etc/rcron.conf.example /etc/rcron/rcron.conf
	chown root:root /etc/rcron/rcron.conf
	chmod 0644 /etc/rcron/rcron.conf

For Debian and Ubuntu systems, there is an included init.d script to generate a state file upon boot:

	cp ./etc/debian.initd.sh /etc/init.d/rcron
	chown root:root /etc/init.d/rcron
	chmod 0755 /etc/init.d/rcron
	update-rc.d rcron defaults

### Configuration and Usage

The rcron config file supports the following directives:

* `cluster_name`
	* An arbitrary name for your group of machines
	* Should be identical across machines within the same group
* `state_file`
	* A file indicating the current machine's state
	* Defaults to `/var/run/rcron.state`
	* For backward compatibility, falls back to a default of `/var/run/rcron/state`
* `default_state`
	* Rcron's default state if the state file cannot be read, either "active" or "passive"
	* Defaults to `active`
* `syslog_facility`
	* The [syslog facility](http://en.wikipedia.org/wiki/Syslog#Facility_levels) from which messages should be generated
	* Defaults to `LOG_CRON`
* `syslog_level`
	* The [severity level](http://en.wikipedia.org/wiki/Syslog#Severity_levels) at which messages should be generated
	* Defaults to `LOG_INFO`
* `nice_level`
	* Job priority per the [nice](http://en.wikipedia.org/wiki/Nice_%28Unix%29) utility
	* Defaults to `19`

To ensure a job is run only on the active machine in a group, prefix it with your rcron path:

	# Run daily at 3am
	0 3 * * * /bin/rcron /home/jdoe/bin/my-job

### Maintaining State

An external tool (eg, keepalived) would maintain state across machines such that only one in the group would have an "active" state and actually _run_ said job(s).

#### Two machines, one active and one passive

Both machines can have an identical rcron configuration with a default state of "passive".

The first machine should have keepalived configured with a _higher_ priority, so that it can promote itself to a MASTER state:

	# /etc/keepalived/keepalived.conf on machine 1
	vrrp_instance VI_1 {
	    state BACKUP
	    interface eth0
	    virtual_router_id 31
	    priority 100 # higher than my siblings!
	    advert_int 1
	    authentication {
	        auth_type PASS
	        auth_pass 1111
	    }
	    notify_backup "/bin/echo passive > /var/run/rcron.state"
	    notify_master "/bin/echo active > /var/run/rcron.state"
	    notify_fault  "/bin/echo passive > /var/run/rcron.state"
	    notify_stop   "/bin/echo passive > /var/run/rcron.state"
	}

The second machine should have a _lower_ priority than the first, so that it defaults to a BACKUP state:

	# /etc/keepalived/keepalived.conf on machine 2
	vrrp_instance VI_1 {
	    state BACKUP
	    interface eth0
	    virtual_router_id 31
	    priority 99 # lower than my first sibling!
	    advert_int 1
	    authentication {
	        auth_type PASS
	        auth_pass 1111
	    }
	    notify_backup "/bin/echo passive > /var/run/rcron.state"
	    notify_master "/bin/echo active > /var/run/rcron.state"
	    notify_fault  "/bin/echo passive > /var/run/rcron.state"
	    notify_stop   "/bin/echo passive > /var/run/rcron.state"
	}

#### Three or more machines

To ensure that only one machine is active, _every_ machine should be configured with a different priority.

For example, if we want to add a third machine to the example above, we would configure a _lower_ priority than the second so that it only promotes itself to a MASTER state when all higher priority machines are unavailable:

	# /etc/keepalived/keepalived.conf on machine 3
	vrrp_instance VI_1 {
	    state BACKUP
	    interface eth0
	    virtual_router_id 31
	    priority 98 # lower than my second sibling!
	    advert_int 1
	    authentication {
	        auth_type PASS
	        auth_pass 1111
	    }
	    notify_backup "/bin/echo passive > /var/run/rcron.state"
	    notify_master "/bin/echo active > /var/run/rcron.state"
	    notify_fault  "/bin/echo passive > /var/run/rcron.state"
	    notify_stop   "/bin/echo passive > /var/run/rcron.state"
	}

#### Machines across disparate networks

By default, keepalived uses [multicast](http://en.wikipedia.org/wiki/Multicast) for communication between machines. If you have multiple machines in the same group that live across two or more separate internal networks, they may not be able to communicate with each other. In such a case, you would need to use [unicast](http://en.wikipedia.org/wiki/Unicast) (added in keepalived v1.2.8), where each machine in the group is configured with the external ip of every other machine:

	# /etc/keepalived/keepalived.conf on machine 1 via unicast
	vrrp_instance VI_1 {
	    state BACKUP
	    interface eth1
	    virtual_router_id 31
	    priority 100
	    advert_int 1
	    unicast_src_ip 30.100.200.101 # this is me!
	    unicast_peer {
	        30.100.200.102 # this is my first sibling
	        30.100.200.103 # this is my second sibling
	        # et al...
	    }
	    authentication {
	        auth_type PASS
	        auth_pass 1111
	    }
	    notify_backup "/bin/echo passive > /var/run/rcron.state"
	    notify_master "/bin/echo active > /var/run/rcron.state"
	    notify_fault  "/bin/echo passive > /var/run/rcron.state"
	    notify_stop   "/bin/echo passive > /var/run/rcron.state"
	}

### Test Drive

You can use the included Vagrantfile & Ansible configuration to create a number of Ubuntu-based virtual machines, all preconfigured with rcron and keepalived v1.2.13 over unicast:

	vagrant up
	vagrant ssh rcl1 -c "sudo tail -f /var/log/syslog | grep -i --color=always keepalived"

### License

As with the original C implementation, this is released under the MIT license.

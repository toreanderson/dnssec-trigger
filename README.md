# Dnssec-Trigger

[![Travis Build Status](https://travis-ci.com/NLnetLabs/dnssec-trigger.svg?branch=master)](https://travis-ci.com/NLnetLabs/dnssec-trigger)
[![Packaging status](https://repology.org/badge/tiny-repos/dnssec-trigger.svg)](https://repology.org/project/dnssec-trigger/versions)

By Wouter Wijngaards, NLnet Labs, 2011 \
BSD license is in the LICENSE file. \
Bugs or comments: labs@nlnetlabs.nl

To install see the INSTALL instructions file.

## Intro

This package contains the dnssec-trigger tools.  It works together with
a local validating resolver (unbound) and keeps DNSSEC enabled.  It does
so by selecting DNSSEC enabled upstream caches or servers for unbound
to talk to and by modifying the DNS path on the system to 127.0.0.1.
If DNSSEC does not work because of middleboxes, the insecure option
(after a dialog window for the user) causes the DNS path to be set to
the insecure servers.

The main components are the daemon, DHCP-hooks, and a GUI-panel.
The daemon starts at bootup and runs in the background.  The DHCP hooks
tell the daemon, these are sometimes scripts depending on the system.
The GUI-panel shows a tray icon notification applet.  The GUI panel shows
the dialog to the user if insecure is the only option.  The GUI panel
has a Reprobe button, so after sigon for the hotspot the user can retry
(it makes the red ! disappear if it works).

Applications can then trust responses with the AD flag from 127.0.0.1.
But they should know that sometimes the resolv.conf contains 'bad'
insecure servers (not 127.0.0.1) and then they must not trust the AD
flag from them (and may need to send the query without the DO flag
to fallback).  Responses asked with DO flag to 127.0.0.1 and with the
returned AD flag can then be trusted.  Trusted DNS responses may help
with DANE.

The dnssec-trigger package thus runs alongside the unbound daemon.  It
provides the user with the option to go to Insecure.  It selects DNSSEC
service where possible.  This helps people run DNSSEC.

## Normal usage

The user logs in and sees a status icon in the tray.  Most of the time
it displays no ! (exclamation) but is quiet.  The icon can be ignored.

When the user connects to a new network, the DHCP hooks notify the
dnssec-trigger daemon.  This probes the network, and notifies unbound.
The user sees no change and continues to ignore the icon, unless there
is no DNSSEC.

If the daemon probe fails to find DNSSEC capability, it tells unbound
to stop talking to the network, and tells the statusicon to ask the user.
A dialog pops up out of the tray icon.  If insecure, then the resolv.conf
is changed to the insecure servers, unbound is inactive (loops to
127.0.0.127).  The user can work normally on this network connection.

For a hotspot, the probe would fail (after a second or two), then with
insecure mode the user can login to the hotspot.  With Reprobe menu
item the user can reprobe dnssec and if it works then (many hotspots
provide good access once logged in) the icon is restored to safe.  The
scripts would also reprobe on a DHCP change.


## Operations on Platforms

How the different platforms operate is described here.

### Security

There used to be a race condition where DHCP info briefly overriddes
the secure version, but this was fixed in 0.6.

### unix - NetworkManager

In /etc/NetworkManager/dispatcher.d a script sends DHCP changes to
the daemon.  The script is a networkmanager dhcp hook script and uses
dnssec-trigger-control to talk to the daemon.  The script uses nmcli
to find the DNS info.

GTK user interface.  In /etc/xdg/autostart/ a .desktop entry starts
the user-side tray icon (dnssec-trigger-panel).  The daemon is started
from /etc/rc.d like regular daemons.  The tray icon communicates with
the daemon over a persistent SSL connection over loopback (127.0.0.1).
It is possible to have multiple tray icons connected over SSL.

### unix - Netconfig

In /etc/netconfig.d a script sends DHCP changes to the daemon.  It greps
the info out of /var/run/netconfig/* files.  It sends with
dnssec-trigger-control to the daemon.

GTK like networkmanager.

### OSX

In /Library/LaunchDaemons two plist files exist.  One starts the daemon.
The other watches the /Library/Preferences/SystemConfiguration for changes
and launches a script.  This script uses ifconfig and parses plist files
and then sends the results with dnssec-trigger-control to the daemon.

The daemon changes resolv.conf but does not need to as on OSX it also
sets the network preferences for the network interfaces to the values.
These preferences survive a reboot, so the reboot is safe. There then
is no connection until unbound is started during reboot.

In /Library/LaunchAgents a plist file starts the tray icon (Cocoa app).
It uses an SSL connection to the daemon over loopback (127.0.0.1).

### Windows

The daemon is a service.  It has a thread that listens to network changes
and blocks on that, it notifies the main daemon when there is a change.
The DHCP DNS servers are picked from the registry and the override DNS
options are put in the registry as the network preferences.  These survive
a reboot, so that the system is safe during a reboot.  There then is no
DNS connection until unbound is started during reboot.

The tray icon is a native application.  It uses SSL to talk to the daemon.


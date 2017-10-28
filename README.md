Ubuntu 10.04 LTS KRACK Vulnerability Backport Patches 
=====================================================

This repository contains patches to fix the KRACK vulnerability in the
following Ubuntu 10.04 LTS packages:

* `linux 2.6.32-47` (`mac80211` module)
* `wpasupplicant 0.6.9-3ubuntu3.2`
* `hostapd 1:0.6.9-3`

Note that with a little bit of extra effort, you can easily port these
changes to other versions of these packages that differ only by the
patch level.  Also, different distributions of similar vintage are
also likely easy to port to, i.e. Debian Squeeze.

Note that patching `hostapd` is only necessary if you are running a
wireless access point, i.e. wireless router.  Thus, most systems will
not have this package installed.

**NOTE:** The patched `hostapd` package has not been tested!  The
ported patches are only included here for completeness.  Use at your
own risk.

How to use
----------

Prerequisite setup commands (should be obvious):

    sudo apt-get install build-essential

Then use the respective instructions for building and installing your
desired package with the fixes applied:

`wpasupplicant`:

    apt-get --download-only source wpasupplicant
    dpkg-source -x wpasupplicant_*.dsc
    sudo apt-get build-dep wpasupplicant
    cd wpasupplicant-0.6.9/
    patch -p1 <../deltas/wpasupplicant_0.6.9-3ubuntu3.2ap1.diff
    dpkg-buildpackage
    cd ..
    sudo dpkg -i wpasupplicant_0.6.9-*ap1_*.deb

`hostapd`:

    apt-get --download-only source hostapd
    dpkg-source -x hostapd_*.dsc
    sudo apt-get build-dep hostapd
    cd hostapd-0.6.9/
    patch -p1 <../deltas/hostapd_0.6.9-3ap1.diff
    dpkg-buildpackage
    cd ..
    sudo dpkg -i hostapd_0.6.9-*ap1_*.deb

`linux` `mac80211` module:

    # Our technique is to simply drop an update to the `mac80211'
    # module into the `updates' directory for the currently running
    # kernel.
    apt-get --download-only source linux
    dpkg-source -x linux_*.dsc
    sudo apt-get build-dep linux
    cd linux-2.6.32
    patch -p1 <../327-mac80211-accept-key-reinstall-without-changing-anyth.patch
    ln -s /usr/src/linux-headers-2.6.32-47-generic/Module.symvers \
      Module.symvers
    make oldconfig
    make prepare
    make modules_prepare
    make SUBDIRS=net/mac80211
    sudo install -m644 net/mac80211/mac80211.ko \
      /lib/modules/2.6.32-47-generic/updates/
    sudo depmod
    # Optional, remove debugging symbols to save disk space:
    sudo strip -S /lib/modules/2.6.32-47-generic/updates/mac80211.ko

    # Now reload the kernel module:
    lsmod | grep mac80211 # Find dependent modules that need to be reloaded.
    # i.e. mac80211, cfg80211, b43
    sudo modprobe -r mac80211 cfg80211 b43
    sudo modprobe mac80211 cfg80211 b43

Finally, last question.  Do patches need to be applied to the wireless
drivers in case they use hardware crypto?  If your wireless drivers
have hardware-accelerated crypto, you are going to need to disable
that until you apply the patches to fix that too.  Otherwise, you are
done at this point.

Where did the patches come from?
--------------------------------

The patches to the WPA packages are backported from the respective
patches in Debian Jessie.  The patch to the `mac80211` kernel module
is backported from the OpenWRT Linux kernel patch.

The following annotations are used on the listing of patches
originating from the Debian Jessie `wpa` source package to illustrate
the changes to apply them to the older Ubuntu 10.04 LTS.

    H = hostapd
    W = wpasupplicant
    N = WNM-Sleep Mode
    T = TDLS

Historically `hostapd` and `wpasupplicant` had separate source
packages, but in later versions, they were combined.

Note that the `wpasupplicant` patches also need to be applied to the
`hostapd` package too, since the same modified files are copied into
that package.

Also, it appears we don't need to apply as many patches for two
reasons:

* We don't have "WNM Sleep Mode"
* We don't have "TDLS"
    * Hence no "TPK" either: TDLS Peer Key

Annotated `wpa` patch listing:

```
H 0001-hostapd-Avoid-key-reinstallation-in-FT-handshake.patch
W 0002-Prevent-reinstallation-of-an-already-in-use-group-ke.patch
  * wpa_sm_drop_sa ()
  ** Only used by a WPA supplicant control interface function that is
     enabled if a "TESTING OPTIONS" config is enabled.  In other
     words, unnecessary for patching production code, hence there is
     no loss here by not patching this function that is non-existent
     in the older version.
  * wpa_wnmsleep_install_key ()
  ** Not needed!
N 0003-Extend-protection-of-GTK-IGTK-reinstallation-of-WNM-.patch
  * wpa_sm_drop_sa ()
  ** Same notes as above.
  * wpa_wnmsleep_install_key ()
  ** Not needed!
H 0004-Fix-PTK-rekeying-to-generate-a-new-ANonce.patch
T 0005-TDLS-Reject-TPK-TK-reconfiguration.patch
N 0006-WNM-Ignore-Key-Data-in-WNM-Sleep-Mode-Response-frame.patch
N 0007-WNM-Ignore-WNM-Sleep-Mode-Response-if-WNM-Sleep-Mode.patch
N 0008-WNM-Ignore-WNM-Sleep-Mode-Response-without-pending-r.patch
W 0009-FT-Do-not-allow-multiple-Reassociation-Response-fram.patch
T 0010-TDLS-Ignore-incoming-TDLS-Setup-Response-retries.patch
```

As you can see, only two patches need to be applied to
`wpasupplicant`, and only four patches need to be applied to
`hostapd`, significantly less than what is required for the later
versions of the software.

Why create these patches?
-------------------------

It is possible to simply patch wireless access points to work around
vulnerable stations/clients.  It is especially common to only run
obsolete systems that cannot be upgraded in environments "sheltered"
and firewalled from the outside world, so patching the access point to
work around station/client vulnerabilities would seem to be a
reasonable workaround for most users.  Nevertheless, I have these
patches simply because that is what I've done, being naive to
alternatives due to not having read about the KRACK vulnerability
carefully enough.  These patches might prove to be a useful starting
point for those people who want to backport security fixes to run
older kernels/software.

Other Remarks
-------------

I'm not very experienced with public contributions to Debian/Ubuntu
distribution packages.  GitHub is easiest for me.  Anyways, this is a
first effort to make some of my work on Debian/Ubuntu distribution
packages available to a broader user base.

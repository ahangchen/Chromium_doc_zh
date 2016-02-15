# Linux Chromium Packages

Some Linux distributions package up Chromium for easy installation. Please note
that Chromium is not identical to Google Chrome -- see
[chromium_browser_vs_google_chrome.md](chromium_browser_vs_google_chrome.md) --
and that distributions may (and actually do) make their own modifications.

TODO: Move away from tables.

| **Distro** | **Contact** | **URL for packages** | **URL for distro-specific patches** |
|:-----------|:------------|:---------------------|:------------------------------------|
| Ubuntu     | Chad Miller `chad.miller@canonical.com` | https://launchpad.net/ubuntu/+source/chromium-browser | https://code.launchpad.net/ubuntu/+source/chromium-browser |
| Debian     | [see package page](http://packages.debian.org/sid/chromium) | in standard repo     | [debian patch tracker](http://patch-tracker.debian.org/package/chromium-browser/) |
| openSUSE   | Raymond Wooninck  `tittiatcoke@gmail.com` | http://software.opensuse.org/search?baseproject=ALL&p=1&q=chromium | ??                                  |
| Arch       | Evangelos Foutras `evangelos@foutrelis.com` | http://www.archlinux.org/packages/extra/x86_64/chromium/ | [link](http://projects.archlinux.org/svntogit/packages.git/tree/trunk?h=packages/chromium) |
| Gentoo     | [project page](http://www.gentoo.org/proj/en/desktop/chromium/index.xml) | Available in portage, [www-client/chromium](http://packages.gentoo.org/package/www-client/chromium) | http://sources.gentoo.org/viewcvs.py/gentoo-x86/www-client/chromium/files/ |
| ALT Linux  | Andrey Cherepanov (Андрей Черепанов) `cas@altlinux.org` | http://packages.altlinux.org/en/Sisyphus/srpms/chromium | http://git.altlinux.org/gears/c/chromium.git?a=tree |
| Mageia     | Dexter Morgan `dmorgan@mageia.org` | http://svnweb.mageia.org/packages/cauldron/chromium-browser-stable/current/SPECS/ | http://svnweb.mageia.org/packages/cauldron/chromium-browser-stable/current/SOURCES/ |
| NixOS      | aszlig `"^[0-9]+$"@regexmail.net` | http://hydra.nixos.org/search?query=pkgs.chromium | https://github.com/NixOS/nixpkgs/tree/master/pkgs/applications/networking/browsers/chromium |

## Unofficial packages

Packages in this section are not part of the distro's official repositories.

| **Distro** | **Contact** | **URL for packages** | **URL for distro-specific patches** |
|:-----------|:------------|:---------------------|:------------------------------------|
| Fedora     | Tom Callaway `tcallawa@redhat.com` | http://repos.fedorapeople.org/repos/spot/chromium/ | ??                                  |
| Slackware  | Eric Hameleers `alien@slackware.com` | http://www.slackware.com/~alien/slackbuilds/chromium/ | http://www.slackware.com/~alien/slackbuilds/chromium/ |

## Other Unixes

| **System** | **Contact** | **URL for packages** | **URL for patches** |
|:-----------|:------------|:---------------------|:--------------------|
| FreeBSD    | http://lists.freebsd.org/mailman/listinfo/freebsd-chromium | http://wiki.freebsd.org/Chromium | http://trillian.chruetertee.ch/chromium |
| OpenBSD    | Robert Nagy `robert@openbsd.org` | http://openports.se/www/chromium | http://www.openbsd.org/cgi-bin/cvsweb/ports/www/chromium/patches/ |

## Updating the list

Are you packaging Chromium for a Linux distro? Is the information above out of
date? Please contact `thestig@chromium.org` with updates.

Before emailing, please note:

*   This is not a support email address
*   If you ask about a Linux distro that is not listed above, the answer will be
    "I don't know"
*   Linux distros supported by Google Chrome are listed here:
    https://support.google.com/chrome/answer/95411

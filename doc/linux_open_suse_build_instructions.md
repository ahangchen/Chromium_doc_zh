# Linux Open SUSE Build Instructions

This page includes some instruction to build Chromium on openSUSE 11.1 and 11.0.
Before reading this page you need to learn the
[Linux Build Instructions](linux_build_instructions.md).

## How to Install Dependencies:

Use zypper command to install dependencies:

(openSUSE 11.1 and higher)

    sudo zypper in subversion pkg-config python perl \
         bison flex gperf mozilla-nss-devel glib2-devel gtk-devel \
         wdiff lighttpd gcc gcc-c++ gconf2-devel mozilla-nspr \
         mozilla-nspr-devel php5-fastcgi alsa-devel libexpat-devel \
         libjpeg-devel libbz2-devel

For 11.0, use `libnspr4-0d` and `libnspr4-dev` instead of `mozilla-nspr` and
`mozilla-nspr-devel`, and use `php5-cgi` instead of `php5-fastcgi`. And need
`gtk2-devel`.

(openSUSE 11.0)

    sudo zypper in subversion pkg-config python perl \
         bison flex gperf mozilla-nss-devel glib2-devel gtk-devel \
         libnspr4-0d libnspr4-dev wdiff lighttpd gcc gcc-c++ libexpat-devel \
         php5-cgi gconf2-devel alsa-devel gtk2-devel jpeg-devel

The Ubuntu package sun-java6-fonts contains a subset of Java of the fonts used.
Since this package requires Java as a prerequisite anyway, we can do the same
thing by just installing the equivalent OpenSUSE Sun Java package:

    sudo zypper in java-1_6_0-sun

Webkit is currently hard-linked to the Microsoft fonts. To install these using zypper

    sudo zypper in fetchmsttfonts pullin-msttf-fonts

To make the fonts installed above work, as the paths are hardcoded for Ubuntu,
create symlinks to the appropriate locations:

```shell
sudo mkdir -p /usr/share/fonts/truetype/msttcorefonts
sudo ln -s /usr/share/fonts/truetype/arial.ttf /usr/share/fonts/truetype/msttcorefonts/Arial.ttf
sudo ln -s /usr/share/fonts/truetype/arialbd.ttf /usr/share/fonts/truetype/msttcorefonts/Arial_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/arialbi.ttf /usr/share/fonts/truetype/msttcorefonts/Arial_Bold_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/ariali.ttf /usr/share/fonts/truetype/msttcorefonts/Arial_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/comic.ttf /usr/share/fonts/truetype/msttcorefonts/Comic_Sans_MS.ttf
sudo ln -s /usr/share/fonts/truetype/comicbd.ttf /usr/share/fonts/truetype/msttcorefonts/Comic_Sans_MS_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/cour.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New.ttf
sudo ln -s /usr/share/fonts/truetype/courbd.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/courbi.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New_Bold_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/couri.ttf /usr/share/fonts/truetype/msttcorefonts/Courier_New_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/impact.ttf /usr/share/fonts/truetype/msttcorefonts/Impact.ttf
sudo ln -s /usr/share/fonts/truetype/times.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman.ttf
sudo ln -s /usr/share/fonts/truetype/timesbd.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/timesbi.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman_Bold_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/timesi.ttf /usr/share/fonts/truetype/msttcorefonts/Times_New_Roman_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/verdana.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana.ttf
sudo ln -s /usr/share/fonts/truetype/verdanab.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana_Bold.ttf
sudo ln -s /usr/share/fonts/truetype/verdanai.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana_Italic.ttf
sudo ln -s /usr/share/fonts/truetype/verdanaz.ttf /usr/share/fonts/truetype/msttcorefonts/Verdana_Bold_Italic.ttf
```

And then for the Java fonts:

```shell
sudo mkdir -p /usr/share/fonts/truetype/ttf-lucida
sudo find /usr/lib*/jvm/java-1.6.*-sun-*/jre/lib -iname '*.ttf' -print \
     -exec ln -s {} /usr/share/fonts/truetype/ttf-lucida \;
```

## Building the software

Please refer to the [Linux Build Instructions](linux_build_instructions.md).

Please update this page if you use different steps.

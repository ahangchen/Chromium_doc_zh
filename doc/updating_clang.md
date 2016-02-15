# Updating clang

1.  Sync your Chromium tree to the latest revision to pick up any plugin
    changes and test the new compiler against ToT
1.  Update clang revision in tools/clang/scripts/update.py, upload CL to
    rietveld
1.  Run tools/clang/scripts/package.py to create a tgz of the binary (mac and
    linux)
1.  Do a local clobber build with that clang (mac and linux). Check that
    everything builds fine and no new warnings appear. (Optional if the
    revision picked in 1 was vetted by other means already.)
1.  Upload the binaries using gsutil, they will appear at
    http://commondatastorage.googleapis.com/chromium-browser-clang/index.html
1.  Run goma package update script to push these packages to goma, send email
1.  `git cl try &&
    git cl try -m tryserver.chromium.mac -b mac_chromium_rel_ng -b
    mac_chromium_asan_rel_ng -b mac_chromium_gn_dbg -b ios_rel_device_ninja &&
    git cl try -m tryserver.chromium.linux -b linux_chromium_chromeos_dbg_ng
    -b linux_chromium_asan_rel_ng -b linux_chromium_chromeos_asan_rel_ng
    -b linux_chromium_rel_ng -b linux_chromium_msan_rel_ng &&
    git cl try -m tryserver.chromium.android -b android_clang_dbg_recipe &&
    git cl try -m tryserver.blink -b linux_blink_rel`
1.  Commit roll CL from the first step
1.  The bots will now pull the prebuilt binary, and goma will have a matching
    binary, too.

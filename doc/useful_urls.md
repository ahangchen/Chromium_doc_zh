# Useful URLs

Chromium has a lot of different pages for a lot of different things.
This page aims to be a repository of useful links that people may find useful.

## Build Status

|:---------------------------------------------|:------------------------|
| http://build.chromium.org/p/chromium/console | Main buildbot waterfall |
| http://chromium-status.appspot.com/lkgr      | Last Known Good Revision. Trybots pull this revision from trunk. |
| http://chromium-status.appspot.com/revisions | List of the last 100 potential LKGRs |
| http://build.chromium.org/p/chromium/lkgr-status/ | Status dashboard for LKGR |
| http://build.chromium.org/p/tryserver.chromium/waterfall?committer=developer@chromium.org | Trybot runs, by developer |
| http://chromium-status.appspot.com/status_viewer | Tree uptime stats       |
| http://chromium-cq-status.appspot.com        | Commit queue status     |
| http://codereview.chromium.org/search?closed=3&commit=2&limit=50 | Pending commit queue jobs |
| http://chromium-build-logs.appspot.com/      | Search for historical test failures by test name |
| http://chromium-build-logs.appspot.com/list  | Filterable list of most recent build logs |

## For Sheriffs

|:---------------------------------------------------------------------------------------------------------|:-------------------------------------------------------------------------|
| http://build.chromium.org/p/chromium.chromiumos/waterfall?show_events=true&reload=120&failures_only=true | List of failing bots for a waterfall (chromium.chromiumos as an example) |
| http://build.chromium.org/p/chromium.linux/waterfall?show_events=true&reload=120&builder=Linux%20Builder%20x64&builder=Linux%20Builder%20(dbg) | Monitor one or multiple bots (Linux Builder x64 and Linux Builder (dbg) on chromium.linux as an example) |
| http://build.chromium.org/p/chromium.win/waterfall/help                                                  | Customize the waterfall view for a waterfall (using chromium.win as an example) |
| http://chromium-sheriffing.appspot.com                                                                   | Alternate waterfall view that helps with test failure triage             |
| http://test-results.appspot.com/dashboards/flakiness_dashboard.html                                      | Lists historical test results for the bots                               |

## Release Information

|:--------------------------------------|:---------------------------------------------------|
| https://omahaproxy.appspot.com/viewer | Current release versions of Chrome on all channels |
| https://omahaproxy.appspot.com/       | Looks up the revision of a build/release version   |

## Source Information

|:------------------------|:------------|
| http://cs.chromium.org/ | Code Search |
| http://cs.chromium.org/SEARCH_TERM | Code Search for a specific SEARCH\_TERM |
| http://src.chromium.org/viewvc/chrome/ | ViewVC History Viewer |
| http://git.chromium.org/gitweb/?p=chromium.git;a=summary | Gitweb History Viewer |
| https://chromium.googlesource.com/chromium/src/+log/b6cfa6a..9a2e0a8?pretty=fuller | Git changes in revision range (also works for build numbers) |
| http://build.chromium.org/f/chromium/perf/dashboard/ui/changelog.html?url=/trunk/src&mode=html&range=SUCCESS_REV:FAILURE_REV | SVN changes in revision range |
| http://build.chromium.org/f/chromium/perf/dashboard/ui/changelog_blink.html?url=/trunk&mode=html&range=SUCCESS_REV:FAILURE_REV | Blink changes in revision range |

## Communication

|:------------------------------------------------------------------|:-------------------------|
| http://groups.google.com/a/chromium.org/group/chromium-dev/topics | Chromium Developers List |
| http://groups.google.com/a/chromium.org/group/chromium-discuss/topics | Chromium Users List      |

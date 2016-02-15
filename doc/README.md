# Chromium docs

This directory contains chromium project documentation in [Markdown].
It is automatically
[rendered by Gitiles](https://chromium.googlesource.com/chromium/src/+/master/docs/).

## Style guide

Markdown documents must follow the [style guide].

## Previewing changes

You can preview your local changes using [md_browser]:

```bash
# in chromium checkout
python tools/md_browser/md_browser.py
```

To review someone else's changes, apply them locally first:

```bash
# in chromium checkout
git cl patch <CL number or URL>
python tools/md_browser/md_browser.py
```

[Markdown]: https://gerrit.googlesource.com/gitiles/+/master/Documentation/markdown.md
[style guide]: https://github.com/google/styleguide/tree/gh-pages/docguide
[md_browser]: ../tools/md_browser/
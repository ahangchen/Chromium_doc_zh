# Closure Compilation

## I just need to fix the compile!

To locally run closure compiler like the bots, do this:

```shell
cd $CHROMIUM_SRC
# sudo apt-get install openjdk-7-jre # may be required
GYP_GENERATORS=ninja tools/gyp/gyp --depth . third_party/closure_compiler/compiled_resources.gyp
ninja -C out/Default
```

To run the v2 gyp format, change the last 2 lines to:

```shell
# notice the 2 in compiled_resources2.gyp
GYP_GENERATORS=ninja tools/gyp/gyp --depth . third_party/closure_compiler/compiled_resources2.gyp
ninja -C out/Default
```

## Background

In C++ and Java, compiling the code gives you _some_ level of protection against
misusing variables based on their type information. JavaScript is loosely typed
and therefore doesn't offer this safety. This makes writing JavaScript more
error prone as it's _one more thing_ to mess up.

Because having this safety is handy, Chrome now has a way to optionally
typecheck your JavaScript and produce compiled output with
[Closure Compiler](https://developers.google.com/closure/compiler/).
The type information is
[annotated in comment tags](https://developers.google.com/closure/compiler/docs/js-for-compiler)
that are briefly described below.

See also:
[the design doc](https://docs.google.com/a/chromium.org/document/d/1Ee9ggmp6U-lM-w9WmxN5cSLkK9B5YAq14939Woo-JY0/edit).

## Assumptions

A working Chrome checkout. See here:
http://www.chromium.org/developers/how-tos/get-the-code

## Typechecking Your Javascript

So you'd like to compile your JavaScript!

Maybe you're working on a page that looks like this:

```html
<script src="other_file.js"></script>
<script src="my_product/my_file.js"></script>
```

Where `other_file.js` contains:

```javascript
var wit = 100;

// ... later on, sneakily ...

wit += ' IQ';  // '100 IQ'
```

and `src/my_product/my_file.js` contains:

```javascript
/** @type {number} */ var mensa = wit + 50;
alert(mensa);  // '100 IQ50' instead of 150
```

In order to check that our code acts as we'd expect, we can create a

    my_project/compiled_resources.gyp

with the contents:

```
# Copyright 2015 The Chromium Authors. All rights reserved.
# Use of this source code is governed by a BSD-style license that can be
# found in the LICENSE file.
{
  'targets': [
    {
      'target_name': 'my_file',  # file name without ".js"

      'variables': {  # Only use if necessary (no need to specify empty lists).
        'depends': [
          'other_file.js',  # or 'other_project/compiled_resources.gyp:target',
        ],
        'externs': [
          '<(CLOSURE_DIR)/externs/any_needed_externs.js'  # e.g. chrome.send(), chrome.app.window, etc.
        ],
      },

      'includes': ['../third_party/closure_compiler/compile_js.gypi'],
    },
  ],
}
```

You should get results like:

```
(ERROR) Error in: my_project/my_file.js
## /my/home/chromium/src/my_project/my_file.js:1: ERROR - initializing variable
## found   : string
## required: number
## /** @type {number} */ var mensa = wit + 50;
##                                   ^
```

Yay! We can easily find our unexpected type errors and write less error-prone
code!

## Continuous Checking

To compile your code on every commit, add a line to
/third_party/closure_compiler/compiled_resources.gyp
like this:

```
{
  'targets': [
    {
      'target_name': 'compile_all_resources',
      'dependencies': [
         # ... other projects ...
++       '../my_project/compiled_resources.gyp:*',
      ],
    }
  ]
}
```

and the
[Closure compiler bot](http://build.chromium.org/p/chromium.fyi/builders/Closure%20Compilation%20Linux)
will [re-]compile your code whenever relevant .js files change.

## Using Compiled JavaScript

Compiled JavaScript is output in
`src/out/<Debug|Release>/gen/closure/my_project/my_file.js` along with a source
map for use in debugging. In order to use the compiled JavaScript, we can create
a

    my_project/my_project_resources.gpy

with the contents:

```
# Copyright 2015 The Chromium Authors. All rights reserved.
# Use of this source code is governed by a BSD-style license that can be
# found in the LICENSE file.

{
  'targets': [
    {
      # GN version: //my_project/resources
      'target_name': 'my_project_resources',
      'type': 'none',
      'variables': {
        'grit_out_dir': '<(SHARED_INTERMEDIATE_DIR)/my_project',
        'my_file_gen_js': '<(SHARED_INTERMEDIATE_DIR)/closure/my_project/my_file.js',
      },
      'actions': [
        {
          # GN version: //my_project/resources:my_project_resources
          'action_name': 'generate_my_project_resources',
          'variables': {
            'grit_grd_file': 'resources/my_project_resources.grd',
            'grit_additional_defines': [
              '-E', 'my_file_gen_js=<(my_file_gen_js)',
            ],
          },
          'includes': [ '../build/grit_action.gypi' ],
        },
      ],
      'includes': [ '../build/grit_target.gypi' ],
    },
  ],
}
```

The variables can also be defined in an existing .gyp file if appropriate. The
variables can then be used in to create a

    my_project/my_project_resources.grd

with the contents:

```
<?xml version="1.0" encoding="utf-8"?>
<grit-part>
  <include name="IDR_MY_FILE_GEN_JS" file="${my_file_gen_js}" use_base_dir="false" type="BINDATA" />
</grit-part>
```

In your C++, the resource can be retrieved like this:
```
base::string16 my_script =
    base::UTF8ToUTF16(
        ResourceBundle::GetSharedInstance()
            .GetRawDataResource(IDR_MY_FILE_GEN_JS)
            .as_string());
```

## Debugging Compiled JavaScript

Along with the compiled JavaScript, a source map is created:
`src/out/<Debug|Release>/gen/closure/my_project/my_file.js.map`

Chrome DevTools has built in support for working with source maps:
https://developer.chrome.com/devtools/docs/javascript-debugging#source-maps

In order to use the source map, you must first manually edit the path to the
'sources' in the .js.map file that was generated. For example, if the source map
looks like this:

```
{
"version":3,
"file":"/tmp/gen/test_script.js",
"lineCount":1,
"mappings":"A,aAAA,IAAIA,OAASA,QAAQ,EAAG,CACtBC,KAAA,CAAM,OAAN,CADsB;",
"sources":["/tmp/tmp70_QUi"],
"names":["fooBar","alert"]
}
```

sources should be changed to:

```
...
"sources":["/tmp/test_script.js"],
...
```

In your browser, the source map can be loaded through the Chrome DevTools
context menu that appears when you right click in the compiled JavaScript source
body. A dialog will pop up prompting you for the path to the source map file.
Once the source map is loaded, the uncompiled version of the JavaScript will
appear in the Sources panel on the left. You can set break points in the
uncompiled version to help debug; behind the scenes Chrome will still be running
the compiled version of the JavaScript.

## Additional Arguments

`compile_js.gypi` accepts an optional `script_args` variable, which passes
additional arguments to `compile.py`, as well as an optional `closure_args`
variable, which passes additional arguments to the closure compiler. You may
also override the `disabled_closure_args` for more strict compilation.

For example, if you would like to specify multiple sources, strict compilation,
and an output wrapper, you would create a

```
my_project/compiled_resources.gyp
```

with contents similar to this:
```
# Copyright 2015 The Chromium Authors. All rights reserved.
# Use of this source code is governed by a BSD-style license that can be
# found in the LICENSE file.
{
  'targets' :[
    {
      'target_name': 'my_file',
      'variables': {
        'source_files': [
          'my_file.js',
          'my_file2.js',
        ],
        'script_args': ['--no-single-file'], # required to process multiple files at once
        'closure_args': [
          'output_wrapper=\'(function(){%output%})();\'',
          'jscomp_error=reportUnknownTypes',     # the following three provide more strict compilation
          'jscomp_error=duplicate',
          'jscomp_error=misplacedTypeAnnotation',
        ],
        'disabled_closure_args': [], # remove the disabled closure args for more strict compilation
      },
      'includes': ['../third_party/closure_compiler/compile_js.gypi'],
    },
  ],
}
```

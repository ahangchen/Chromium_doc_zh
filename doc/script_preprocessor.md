# Using the Chrome Devtools JavaScript preprocessing feature

The Chrome Devtools JavaScript preprocessor intercepts JavaScript just before it
enters V8, the Chrome JS system, allowing the JS to be transcoded before
compilation.  In combination with page injected JavaScript, the preprocessor
allows a complete synthetic runtime to be constructed in JavaScript. Combined
with other functions in the `chrome.devtools` extension API, the preprocessor
allows new more sophisticated  JavaScript-related developer tools to be created.

## API

To use the script preprocessor, write a
[chrome devtools extension](http://developer.chrome.com/extensions/devtools.inspectedWindow.html#method-reload)
that reloads the Web page with the preprocessor installed:

```javascript
chrome.devtools.inspectedWindow.reload({
    ignoreCache: true,
    injectedScript: runThisFirst,
    preprocessorScript: preprocessor
});
```

where `preprocessorScript` is source code (string) for a JavaScript function
taking three string arguments, the source to preprocess, the URL of the source,
and a function name if the source is an DOM event handler. The
`preprocessorerScript` function should return a string to be compiled by Chrome
in place of the input source. In the case that the source is a DOM event
handler, the returned source must compile to a single JS function.

The
[Chrome Preprocessor Example](http://developer.chrome.com/extensions/samples.html)
illustrates the API call in a simple chrome devtools extension. Download and
unpack the .zip file, use `chrome://extensions` in Developer Mode and load the
unpacked extension. Then open or reopen devtools. The Preprocessor panel has a
**reload** button that triggers a simple preprocessor.

The preprocessor runs in an isolated world similar to the environment of Chrome
content scripts. A `window` object is available but it shares no properties with
the Web page `window` object.  DOM calls in the preprocessor environment will
operate on the Web page, but developers should be cautious about operating on
the DOM in the preprocessor. We do not test such operations though we expect the
result to resemble calls from the outer function of `<script>` tags.

In some applications the developer may coordinate runtime initialization using
the `injectedScript` property in the object passed to the `reload()` call. This
is also JavaScript source code; it is compiled into the page ahead of any Web
page scripts and thus before any JavaScript is preprocessed.

The preprocessor is compiled once just before the first JavaScript appears. It
remains active until the page is reloaded or otherwise navigated. Navigating the
Web page back and then forward will result in no preprocessing. Closing devtools
will leave the preprocessor in place.

## Use Cases

The script preprocessor supports transcoding input source to JavaScript. Use cases include:

*   Adding write barriers for Querypoint debugging,
*   Supporting feature-specific debugging of next generation EcmaScript using eg Traceur,
*   Integration of development tools like coverage analysis.
*   Analysis of call sequences for performance tuning.

Several JavaScript compilers support transcoding, including
[Traceur](https://github.com/google/traceur-compiler#readme) and
[Esprima](http://esprima.org/).

## Implementation

The implementation relies on the Devtools front-end hosting an extension
supplying the preprocessor script; the front end communicates with the browser
backend over eg web sockets.

The devtools extension function call issues a postMessage() event from the
devtools extension iframe to the devtools main frame. The event is handled in
`ExtensionServer.js` which forwards it over the
[devtools remote debug protocol](https://developers.google.com/chrome-developer-tools/docs/protocol/1.0/page#command-reload).
(See [Bug 229971](https://crbug.com/229971) for this part of the implementation
and its status).

When the preprocessor script arrives in the back end,
`InspectorPageAgent::reload` stores the preprocessor script in
`m_pendingScriptPreprocessor`. After the browser begins the reload operation, it
calls `PageDebuggerAgent::didClearWindowObjectInWorld` which moves the processor
source into the `scriptDebugServer()`.

Next the browser prepares the page environment and calls
`PageDebuggerAgent::didClearWindowObjectInWorld`. This function clears the
preprocessor object pointer and if it is not recreated during the page load, no
scripts will be preprocessed. At this point we only store the preprocessor
source, delaying the compilation of the preprocessor until just before its first
use. This helps ensure that the JS environment we use is fully initialized.

Source to be preprocessed comes from three different places:

1.  Web page `<script>` tags,
1.  DOM event-listener attributes, eg `onload`,
1.  JS `eval()` or `new Function()` calls.

When the browser encounters either a `<script>` tag
(`ScriptController::executeScriptInMainWorld`) or an  element attribute script
(`V8LazyEventListener::prepareListenerObject`) we call a corresponding function
in InspectorInstrumentation. This function has a fast inlined return path in the
case that the debugger is not attached.

If the debugger is attached, InspectorInstrumentation will call the matching
function in PageDebuggerAgent (see core/inspector/InspectorInstrumentation.idl).
It checks to see if the preprocessor is installed. If not, it returns.

The preprocessor source is stored in PageScriptDebugServer.
If the preprocessor is installed, we check to see if it is compiled. If not, we
create a new `ScriptPreprocessor` object.  The constructor uses
`ScriptController::executeScriptInIsolatedWorld` to compile the preprocessor in
a new isolated world associated with the Web page's main world.  If the
compilation and outer script execution succeed and if the result is a JavaScript
function, we store the resulting function as a `ScopedPersistent<v8::Function>`
member of the preprocessor.

If the `PageScriptDebugServer::preprocess()` has a value for the preprocessor
function, it applies the function to the web page source using
`V8ScriptRunner::callAsFunction()`. This calls the compiled JS function in the
ScriptPreprocessor's isolated world and retrieves the resulting string.

When the preprocessed JavaScript source runs it may call `eval()` or
`new Function()`.  These calls cause the V8 runtime to compile source.
Immediately before compiling, V8 issues a beforeCompile event which triggers
`ScriptDebugServer::handleV8DebugEvent()`. This code is only called if the
debugger is active. In the handler we call `ScriptDebugServer::preprocessEval()`
to examine the ScriptCompilationTypeInfo, a marker set by V8, to see if we are
compiling dynamic code. Only dynamic code is preprocessed in this function and
only if we are not executing the preprocessor itself.

During the browser operation, API generation code, debugger console
initialization code, injected page script code, debugger information extraction
code, and regular web page code enter this function.  There is currently no way
to distinguish internal or system code from the web page code. However the
internal code is all static. By limiting our preprocessing to dynamic code in
the beforeCompile handler, we know we are only operating on Web page code. The
static Web page code is preprocessed as described above.

## Limitations

We currently do not support preprocessing of WebWorker source code.

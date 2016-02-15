# What is chrome://settings?

Chrome (version 10 and above) uses WebUI settings by default for all platforms.
Access it via the wrench menu ("Preferences" on Mac and Linux; "Options" on
Windows and ChromeOS), or by typing chrome://settings into the address bar.

One advantage of chrome://settings over platform-native dialogs is that it is
shared by all platforms; therefore, it is easier to add new options UI and to
keep all platforms in sync.

Note that at the time of this writing, DOMUI is being renamed to WebUI. The two
terms will be used interchangeably herein.

## Moving parts

### String resources

Strings live in `chrome/app/generated_resources.grd`. There are several rules
governing the format of strings:

*   the **casing of button text** depends on the platform. If your string will
    be displayed on a button, you need to add it twice, in sentence case and
    title case. Follow examples inside `<if expr="pp_ifdef('use_titlecase')">`
    blocks. Do this even if your string is a single word because it may not be a
    single word in another locale.
*   strings that are associated with form controls (buttons, checkboxes,
    dropdowns, etc.) should NOT have **trailing punctuation**. Standalone
    strings, such as sectional descriptive text, should have trailing
    punctuation.
*   strings may be different between Google Chrome and chromium. If they differ
    only in **product name**, put them in `generated_resources.grd` and use
    product name placeholders; if they differ more substantially, use
    `chromium_strings.grd` and `google_chrome_strings.grd`.

### WebUI Handlers

The C++ WebUI handler classes for chrome://settings live in
`chrome/browser/ui/webui/options/`. These objects both handle messages from and
dispatch messages to the page. Each handler is a logical grouping of related
functionality. Hence `ContentSettingsHandler` is both the delegate and
controller of the content settings subpage.

A handler sends a message to the page via `dom_ui_->CallJavascriptFunction()`
and registers for messages from the page via
`dom_ui_->RegisterMessageCallback()`.

### HTML/CSS/JS

The HTML, CSS, and JS resources live in `chrome/browser/resources/options`. They
are compiled into one resource at compile time by grit, via `<link>`, `<src>`
and `<include>` tags in `options.html`. There is no need to add new files to any
`.gyp` file.

Some things that are good to know:

*   JS objects are bound to HTML nodes via `decorate()`.  * Javascript calls
    into C++ via `chrome.send`.  * Some automatic preference handling is
    provided in `preferences.js` and `pref_ui.js`.  * Strings defined in a WebUI
    handler are available via `localStrings.getString()` and friends.

## Example: adding a simple pref

Suppose you want to add a pref controlled by a **checkbox**.

### 1. Getting UI approval

Ask one of the UI leads where your option belongs. First point of contact should
be Alex Ainslie <ainslie at chromium>.

### 2. Adding Strings

Add the string to `chrome/app/generated_resources.grd` near related strings. No
trailing punctuation; sentence case.

### 3. Changing WebUI Handler

For simple prefs, the UI is kept in sync with the value automagically (see
`CoreOptionsHandler`). Find the handler most closely relevant to your pref. The
only action we need take here is to make the page aware of the new string.
Locate the method in the handler called `GetLocalizedStrings` and look at its
body for examples of `SetString` usage. The first parameter is the javascript
name for the string.

### 4. HTML/CSS/JS

An example of the form a checkbox should take in html:

```html
<label class="checkbox">
  <input id="clear-cookies-on-exit"
       pref="profile.clear_site_data_on_exit" type="checkbox">
  <span i18n-content="clearCookiesOnExit"></span>
</label>
```

Of note:

*   the `checkbox` class allows us to style all checkboxes the same via CSS
*   the class and ID are in dash-form * the i18n-content value is in camelCase
*   the pref attribute matches that which is defined in
    `chrome/common/pref_names.cc` and allows the prefs framework to
    automatically keep it in sync with the value in C++
*   the `i18n-content` value matches the string we defined in the WebUI handler.
    The `textContent` of the `span` will automatically be set to the associated
    text.

In this example, no additional JS or CSS are needed.

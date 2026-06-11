# Content scripts &nbsp;|&nbsp; Chrome for Developers

> Source: https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts
> Cached: 2026-06-11T13:31:12.150Z

---

Home
        
  
  
      
    
  
  
  
    
      
      
    
    
    
      
        
  
    
          Docs
        
  
  
      
    
  
  
  
    
      
      
    
    
    
      
        
  
    
          Chrome Extensions
        
  
  
      
    
  
  
  
    
      
      
    
    
    
      
        
  
    
          Develop
        
  
  
      
    
  
  

    
      
    
    
  
    
  
  
    
      Content scripts

    
    
      
    

    
      
      Stay organized with collections
    
    
      
      Save and categorize content based on your preferences.
    
  
    
  
      
    
  
  

  
  
  
    
  

  
    
    

Content scripts are files that run in the context of web pages. Using the standard Document
Object Model (DOM), they are able to read details of the web pages the browser visits, make
changes to them, and pass information to their parent extension.

## Understand content script capabilities

Content scripts can access the following extension APIs directly:

- [`dom`](/docs/extensions/reference/api/dom)

- [`i18n`](/docs/extensions/reference/api/i18n)

- [`storage`](/docs/extensions/reference/api/storage)

- [`runtime.connect()`](/docs/extensions/reference/api/runtime#method-connect)

- [`runtime.getManifest()`](/docs/extensions/reference/api/runtime#method-getManifest)

- [`runtime.getURL()`](/docs/extensions/reference/api/runtime#method-getURL)

- [`runtime.id`](/docs/extensions/reference/api/runtime#property-id)

- [`runtime.onConnect`](/docs/extensions/reference/api/runtime#event-onConnect)

- [`runtime.onMessage`](/docs/extensions/reference/api/runtime#event-onMessage)

- [`runtime.sendMessage()`](/docs/extensions/reference/api/runtime#method-sendMessage)

Content scripts are unable to access other APIs directly. But they can access them indirectly by [exchanging messages](/docs/extensions/develop/concepts/messaging) with other parts of your extension.

You can also access other files in your extension from a content script, using
APIs like `fetch()`. To do this, you need to declare them as
[web-accessible resources](/docs/extensions/reference/manifest/web-accessible-resources). Note that this also exposes the resources to any
first-party or third-party scripts running on the same site.

## Work in isolated worlds

Content scripts live in an isolated world, allowing a content script to make changes to its
JavaScript environment without conflicting with the page or other extensions' content scripts.

**Key term:** An **isolated world** is a private execution environment that isn't accessible to the page or other
extensions. A practical consequence of this isolation is that JavaScript variables in an extension's
content scripts are not visible to the host page or other extensions' content scripts. The concept
was originally introduced with the initial launch of Chrome, providing isolation for browser tabs.

An extension may run in a web page with code similar to the following example.

webPage.html

```
<html>
  <button id="mybutton">click me</button>
  <script>
    var greeting = "hello, ";
    var button = document.getElementById("mybutton");
    button.person_name = "Bob";
    button.addEventListener(
        "click", () => alert(greeting + button.person_name + "."), false);
  </script>
</html>

```

That extension could inject the following content script using one of the techniques outlined in the
[Inject scripts](#functionality) section.

content-script.js

```
var greeting = "hola, ";
var button = document.getElementById("mybutton");
button.person_name = "Roberto";
button.addEventListener(
    "click", () => alert(greeting + button.person_name + "."), false);

```

With this change, both alerts appear in sequence when the button is clicked.

**Note:** Not only does each extension run in its own isolated world, but content scripts and the web page do
too. This means that none of these (web page, content scripts, and any running extensions) can
access the context and variables of the others.

## Inject scripts

Content scripts can be [declared statically](#static-declarative), declared
dynamically, or [programmatically injected](#programmatic).

### Inject with static declarations

Use static content script declarations in manifest.json for scripts that should be automatically
run on a well known set of pages.

Statically declared scripts are registered in the manifest under the `"content_scripts"` key.
They can include JavaScript files, CSS files, or both. All auto-run content scripts must specify
[match patterns](/docs/extensions/develop/concepts/match-patterns).

manifest.json

```
{
 "name": "My extension",
 ...
 "content_scripts": [
   {
     "matches": ["https://*.nytimes.com/*"],
     "css": ["my-styles.css"],
     "js": ["content-script.js"]
   }
 ],
 ...
}

```

  
    
      Name
      Type
      Description
    
    
      `matches`
      array of strings
      *Required.* Specifies which pages this content script will be injected into. See [Match Patterns](/docs/extensions/develop/concepts/match-patterns) for details on the syntax of these strings
        and [Match patterns and globs](#matchAndGlob) for information on how to exclude
        URLs.
    
    
      `css`
      array of strings
      *Optional.* The list of CSS files to be injected into matching pages. These are
        injected in the order they appear in this array, before any DOM is constructed or displayed
        for the page.
    
    
      `js`
      
        array of strings
      
      *Optional.* The list of JavaScript files to be injected into matching pages. Files
        are injected in the order they appear in this array. Each string in this list must contain
        a relative path to a resource in the extension's root directory. Leading slashes (`/`) are
        automatically trimmed.
    
    
      `run_at`
      [RunAt](/docs/extensions/reference/api/extensionTypes#type-RunAt)
      *Optional.* Specifies when the script should be injected into the page. Defaults to
        `document_idle`.
    
    
      `match_about_blank`
      boolean
      *Optional.* Whether the script should inject into an `about:blank` frame
        where the parent or opener frame matches one of the patterns declared in
        `matches`. Defaults to false.
    
    
      `match_origin_as_fallback`
      boolean
      
        *Optional.* Whether the script should inject in frames that were
        created by a matching origin, but whose URL or origin may not directly
        match the pattern. These include frames with different schemes, such as
        `about:`, `data:`, `blob:`, and
        `filesystem:`. See also
        [Injecting in related frames](#injecting-in-related-frames).
      
    
    
      `world`
      [ExecutionWorld](/docs/extensions/reference/api/scripting#type-ExecutionWorld)
      
        *Optional.* The JavaScript world for a script to execute within. Defaults to `ISOLATED`. See also
        [Work in isolated worlds](#isolated_world).
      
    
  

Within a given [stage](#run_at) of the document lifecycle, content scripts
declared statically in the manifest are the first to be injected, before content
scripts registered in any other way. They are injected in the order in which
they are specified in the manifest.

### Inject with dynamic declarations

Dynamic content scripts are useful when the match patterns for content scripts are
not well known or when content scripts shouldn't always be injected on known hosts.

Introduced in Chrome 96, dynamic declarations are similar to static
declarations, but the content script object is registered with Chrome using
methods in the [`chrome.scripting` namespace](/docs/extensions/reference/api/scripting) rather than in
[manifest.json](/docs/extensions/reference/manifest). The Scripting API also allows extension developers
to:

- [Register](/docs/extensions/reference/api/scripting#method-registerContentScripts) content scripts.

- [Get a list of](/docs/extensions/reference/api/scripting#method-getRegisteredContentScripts) registered content scripts.

- [Update](/docs/extensions/reference/api/scripting#method-updateContentScripts) the list of registered content scripts.

- [Remove](/docs/extensions/reference/api/scripting#method-unregisterContentScripts) registered content scripts.

Like static declarations, dynamic declarations can include JavaScript files, CSS files, or both.

service-worker.js

```
chrome.scripting
  .registerContentScripts([{
    id: "session-script",
    js: ["content.js"],
    persistAcrossSessions: false,
    matches: ["*://example.com/*"],
    runAt: "document_start",
  }])
  .then(() => console.log("registration complete"))
  .catch((err) => console.warn("unexpected error", err))

```

service-worker.js

```
chrome.scripting
  .updateContentScripts([{
    id: "session-script",
    excludeMatches: ["*://admin.example.com/*"],
  }])
  .then(() => console.log("registration updated"));

```

service-worker.js

```
chrome.scripting
  .getRegisteredContentScripts()
  .then(scripts => console.log("registered content scripts", scripts));

```

service-worker.js

```
chrome.scripting
  .unregisterContentScripts({ ids: ["session-script"] })
  .then(() => console.log("un-registration complete"));

```

### Inject programmatically

Use programmatic injection for content scripts that need to run in response to events or on specific
occasions.

To inject a content script programmatically, your extension needs [host permissions](/docs/extensions/reference/permissions) for
the page it's trying to inject scripts into. Host permissions can either be granted by
requesting them as part of your extension's manifest or temporarily using [`"activeTab"`](/docs/extensions/develop/concepts/activeTab).

The following are different versions of an activeTab-based extension.

manifest.json:

```
{
  "name": "My extension",
  ...
  "permissions": [
    "activeTab",
    "scripting"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "action": {
    "default_title": "Action Button"
  }
}

```

Content scripts can be injected as files.

content-script.js

```

document.body.style.backgroundColor = "orange";

```

service-worker.js:

```
chrome.action.onClicked.addListener((tab) => {
  chrome.scripting.executeScript({
    target: { tabId: tab.id },
    files: ["content-script.js"]
  });
});

```

Or, a function body can be injected and executed as a content script.

service-worker.js:

```
function injectedFunction() {
  document.body.style.backgroundColor = "orange";
}

chrome.action.onClicked.addListener((tab) => {
  chrome.scripting.executeScript({
    target : {tabId : tab.id},
    func : injectedFunction,
  });
});

```

Be aware that the injected function is a copy of the function referenced in the
`chrome.scripting.executeScript()` call, not the original function itself. As a result, the function's
body must be self contained; references to variables outside of the function will cause the content
script to throw a [`ReferenceError`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/ReferenceError).

When injecting as a function, you can also pass arguments to the function.

service-worker.js

```
function injectedFunction(color) {
  document.body.style.backgroundColor = color;
}

chrome.action.onClicked.addListener((tab) => {
  chrome.scripting.executeScript({
    target : {tabId : tab.id},
    func : injectedFunction,
    args : [ "orange" ],
  });
});

```

### Exclude matches and globs

To customize specified page matching, include the following fields in a declarative
registration.

  
    
      Name
      Type
      Description
    
    
      `exclude_matches`
      array of strings
      *Optional.* Excludes pages that this content script would otherwise be injected
        into. See [Match Patterns](/docs/extensions/develop/concepts/match-patterns) for details of the syntax of
        these strings.
    
    
      `include_globs`
      array of strings
      *Optional.* Applied after `matches` to include only those URLs that also
        match this glob. This is intended to emulate the [`@include`](https://wiki.greasespot.net/Metadata_Block#.40include)
        Greasemonkey keyword.
    
    
      `exclude_globs`
      array of string
      *Optional.* Applied after `matches` to exclude URLs that match this
        glob. Intended to emulate the [`@exclude`](https://wiki.greasespot.net/Metadata_Block#.40exclude)
        Greasemonkey keyword.
    
  

The content script will be injected into a page if both of the following are true:

- Its URL matches any `matches` pattern and any `include_globs` pattern.

The URL doesn't also match an `exclude_matches` or `exclude_globs` pattern.
Because the `matches` property is required, `exclude_matches`, `include_globs`, and `exclude_globs`
can only be used to limit which pages will be affected.

The following extension injects the content script into `https://www.nytimes.com/health`
but not into `https://www.nytimes.com/business` .

manifest.json

```
{
  "name": "My extension",
  ...
  "content_scripts": [
    {
      "matches": ["https://*.nytimes.com/*"],
      "exclude_matches": ["*://*/*business*"],
      "js": ["contentScript.js"]
    }
  ],
  ...
}

```

service-worker.js

```
chrome.scripting.registerContentScripts([{
  id : "test",
  matches : [ "https://*.nytimes.com/*" ],
  excludeMatches : [ "*://*/*business*" ],
  js : [ "contentScript.js" ],
}]);

```

Glob properties follow a different, more flexible syntax than [match patterns](/docs/extensions/develop/concepts/match-patterns). Acceptable glob
strings are URLs that may contain "wildcard" asterisks and question marks. The asterisk (`*`)
matches any string of any length, including the empty string, while the question mark (`?`) matches
any single character.

For example, the glob `https://???.example.com/foo/\*` matches any of the following:

- `https://www.example.com/foo/bar`

- `https://the.example.com/foo/`

However, it does *not* match the following:

- `https://my.example.com/foo/bar`

- `https://example.com/foo/`

- `https://www.example.com/foo`

This extension injects the content script into `https://www.nytimes.com/arts/index.html` and
`https://www.nytimes.com/jobs/index.htm*`, but not into
`https://www.nytimes.com/sports/index.html`:

manifest.json

```
{
  "name": "My extension",
  ...
  "content_scripts": [
    {
      "matches": ["https://*.nytimes.com/*"],
      "include_globs": ["*nytimes.com/???s/*"],
      "js": ["contentScript.js"]
    }
  ],
  ...
}

```

This extension injects the content script into `https://history.nytimes.com` and
`https://.nytimes.com/history`, but not into `https://science.nytimes.com` or
`https://www.nytimes.com/science`:

manifest.json

```
{
  "name": "My extension",
  ...
  "content_scripts": [
    {
      "matches": ["https://*.nytimes.com/*"],
      "exclude_globs": ["*science*"],
      "js": ["contentScript.js"]
    }
 

... [Content truncated]
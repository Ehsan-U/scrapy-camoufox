# scrapy-camoufox: Camoufox integration for Scrapy
[![version](https://img.shields.io/pypi/v/scrapy-camoufox.svg)](https://pypi.python.org/pypi/scrapy-camoufox)
[![pyversions](https://img.shields.io/pypi/pyversions/scrapy-camoufox.svg)](https://pypi.python.org/pypi/scrapy-camoufox)


A [Scrapy](https://github.com/scrapy/scrapy) Download Handler which performs requests using
[Camoufox](https://github.com/daijro/camoufox).
It can be used to handle pages that require JavaScript (among other things),
while adhering to the regular Scrapy workflow (i.e. without interfering
with request scheduling, item processing, etc).


## Requirements

After the release of [version 2.0](https://docs.scrapy.org/en/latest/news.html#scrapy-2-0-0-2020-03-03),
which includes [coroutine syntax support](https://docs.scrapy.org/en/2.0/topics/coroutines.html)
and [asyncio support](https://docs.scrapy.org/en/2.0/topics/asyncio.html), Scrapy allows
to integrate `asyncio`-based projects such as `Camoufox`.


## Installation

`scrapy-camoufox` is available on PyPI and can be installed with `pip`:

```
pip install scrapy-camoufox
```

`camoufox` is defined as a dependency so it gets installed automatically,
however it might be necessary to install the browser that will be
used:

```
camoufox fetch
```


## Activation

### Download handler

Replace the default `http` and/or `https` Download Handlers through
[`DOWNLOAD_HANDLERS`](https://docs.scrapy.org/en/latest/topics/settings.html):

```python
# settings.py
DOWNLOAD_HANDLERS = {
    "http": "scrapy_camoufox.handler.ScrapyCamoufoxDownloadHandler",
    "https": "scrapy_camoufox.handler.ScrapyCamoufoxDownloadHandler",
}
```

Note that the `ScrapyCamoufoxDownloadHandler` class inherits from the default
`http/https` handler. Unless explicitly marked (see [Basic usage](#basic-usage)),
requests will be processed by the regular Scrapy download handler.


### Twisted reactor

[Install the `asyncio`-based Twisted reactor](https://docs.scrapy.org/en/latest/topics/asyncio.html#installing-the-asyncio-reactor):

```python
# settings.py
TWISTED_REACTOR = "twisted.internet.asyncioreactor.AsyncioSelectorReactor"
```

This is the default in new projects since [Scrapy 2.7](https://github.com/scrapy/scrapy/releases/tag/2.7.0).


## Basic usage

Set the [`camoufox`](#camoufox) [Request.meta](https://docs.scrapy.org/en/latest/topics/request-response.html#scrapy.http.Request.meta)
key to download a request using Camoufox:

```python
import scrapy

class AwesomeSpider(scrapy.Spider):
    name = "awesome"

    def start_requests(self):
        # GET request
        yield scrapy.Request("https://httpbin.org/get", meta={"camoufox": True})
        # POST request
        yield scrapy.FormRequest(
            url="https://httpbin.org/post",
            formdata={"foo": "bar"},
            meta={"camoufox": True},
        )

    def parse(self, response, **kwargs):
        # 'response' contains the page as seen by the browser
        return {"url": response.url}
```

### `CAMOUFOX_LAUNCH_OPTIONS`
Type `dict`, default `{}`

A dictionary with options to be passed as keyword arguments when launching the
Browser. See the docs for
[`AsyncNewBrowser`](https://camoufox.com/python/usage/)
for a list of supported keyword arguments.


#### Device Rotation
Camoufox will generate device information for you based on the following parameters.

```python
CAMOUFOX_LAUNCH_OPTIONS = {
    # Operating system to use for the fingerprint generation. 
    # available os: ['windows', 'macos', 'linux']
    "os": ["macos"],
    # Fonts to load into Camoufox, in addition to the default fonts for the target os
    "fonts": ["Arial", "Helvetica", "Times New Roman"],
    # Constrains the screen dimensions of the generated fingerprint.
    # from browserforge.fingerprints import Screen
    "screen": Screen(max_width=1920, max_height=1080),
    # Use a specific WebGL vendor/renderer pair. Passed as a tuple of (vendor, renderer). The vendor & renderer combination must be supported for the target os or this will cause leaks.
    "webgl_config": ("Apple", "Apple M1, or similar"),
}
```
Note: While this sets the screen dimensions, it has very light impact on the size of the window. Sometimes the window will be larger, sometimes smaller.


#### Configuration
Extra feature configuration and quality of life options.

```python
CAMOUFOX_LAUNCH_OPTIONS = {
    # Humanize the cursor movement. Takes either True, or the MAX duration in seconds of the cursor movement.
    "humanize": True,
    # Defaults to False. If you are running linux, passing 'virtual' will use Xvfb.
    "headless": True,
    # List of Firefox addons to use. Must be paths to extracted addons.
    "addons": ["/path/to/addon1", "/path/to/addon2"],
    # Set the window size in (width, height) pixels. This will also set the window.screenX and window.screenY properties to position the window at the center of the generated screen.
    "window": (1282, 955),
    # Whether to cache previous pages, requests, etc. Disabled by default as it uses more memory.
    "enable_cache": True,
    # persistent context
    "persistent_context": True,
    "user_data_dir": '/path/to/profile/dir',
}
```
Note: Camoufox will automatically generate a window size for you. Using a fixed window size can lead to fingerprinting. Do not set the window size to a fixed value unless for debugging purposes.


#### Location & Language

Prevent proxy detection by matching your geolocation & locale with your target IP. This will populate the Geolocation & Intl properties for you.

```python
CAMOUFOX_LAUNCH_OPTIONS = {
    # Calculates longitude, latitude, timezone, country, & locale based on the IP address. Pass the target IP address to use, or True to find the IP address automatically.
    "geoip": "203.0.113.0",
    "proxy": ...,
    # Locale(s) to use in Camoufox. Can be a list of strings, or a single string separated by a comma. The first locale in the list will be used for the Intl API.
    "locale": "en-US",
}
```

#### Toggles

Shortcuts for common Firefox preferences and security toggles.


```python
CAMOUFOX_LAUNCH_OPTIONS = {
    "block_images": True,
    "block_webrtc": True,
    # Whether to block WebGL. To prevent leaks, only use this for special cases.
    "block_webgl": True,
}
```
Note: Camoufox will warn you if your passed configuration might cause leaks.


### `CAMOUFOX_CONTEXTS`
Type `dict[str, dict]`, default `{}`

A dictionary which defines Browser contexts to be created on startup.
It should be a mapping of (name, keyword arguments).

```python
CAMOUFOX_CONTEXTS = {
    "foobar": {
        "context_arg1": "value",
        "context_arg2": "value",
    },
    "default": {
        "context_arg1": "value",
        "context_arg2": "value",
    },
    "persistent": {
        "user_data_dir": "/path/to/dir",  # will be a persistent context
        "context_arg1": "value",
    },
}
```

See the section on [browser contexts](#browser-contexts) for more information.
See also the docs for [`Browser.new_context`](https://playwright.dev/python/docs/api/class-browser#browser-new-context).

### `CAMOUFOX_MAX_CONTEXTS`
Type `Optional[int]`, default `None`

Maximum amount of allowed concurrent Camoufox contexts. If unset or `None`,
no limit is enforced. See the [Maximum concurrent context count](#maximum-concurrent-context-count)
section for more information.

```python
CAMOUFOX_MAX_CONTEXTS = 8
```

### `CAMOUFOX_DEFAULT_NAVIGATION_TIMEOUT`
Type `Optional[float]`, default `None`

Timeout to be used when requesting pages by Camoufox, in milliseconds. If
`None` or unset, the default value will be used (30000 ms at the time of writing).
See the docs for [BrowserContext.set_default_navigation_timeout](https://playwright.dev/python/docs/api/class-browsercontext#browser-context-set-default-navigation-timeout).

```python
CAMOUFOX_DEFAULT_NAVIGATION_TIMEOUT = 10 * 1000  # 10 seconds
```

### `CAMOUFOX_RESTART_DISCONNECTED_BROWSER`
Type `bool`, default `True`

Whether the browser will be restarted if it gets disconnected, for instance if the local
browser crashes or a remote connection times out.
Implemented by listening to the
[`disconnected` Browser event](https://playwright.dev/python/docs/api/class-browser#browser-event-disconnected),
for this reason it does not apply to persistent contexts since
[BrowserType.launch_persistent_context](https://playwright.dev/python/docs/api/class-browsertype#browser-type-launch-persistent-context)
returns the context directly.

### `CAMOUFOX_MAX_PAGES_PER_CONTEXT`
Type `int`, defaults to the value of Scrapy's `CONCURRENT_REQUESTS` setting

Maximum amount of allowed concurrent Camoufox pages for each context.
See the [notes about leaving unclosed pages](#receiving-page-objects-in-callbacks).

```python
CAMOUFOX_MAX_PAGES_PER_CONTEXT = 4
```

### `CAMOUFOX_ABORT_REQUEST`
Type `Optional[Union[Callable, str]]`, default `None`

A predicate function (or the path to a function) that receives a
[`playwright.async_api.Request`](https://playwright.dev/python/docs/api/class-request)
object and must return `True` if the request should be aborted, `False` otherwise.
Coroutine functions (`async def`) are supported.

Note that all requests will appear in the DEBUG level logs, however there will
be no corresponding response log lines for aborted requests. Aborted requests
are counted in the `camoufox/request_count/aborted` job stats item.

```python
def should_abort_request(request):
    return (
        request.resource_type == "image"
        or ".jpg" in request.url
    )

CAMOUFOX_ABORT_REQUEST = should_abort_request
```

### General note about settings
For settings that accept object paths as strings, passing callable objects is
only supported when using Scrapy>=2.4. With prior versions, only strings are
supported.


## Supported [`Request.meta`](https://docs.scrapy.org/en/latest/topics/request-response.html#scrapy.http.Request.meta) keys

### `camoufox`
Type `bool`, default `False`

If set to a value that evaluates to `True` the request will be processed by Camoufox.

```python
return scrapy.Request("https://example.org", meta={"camoufox": True})
```

### `camoufox_context`
Type `str`, default `"default"`

Name of the context to be used to download the request.
See the section on [browser contexts](#browser-contexts) for more information.

```python
return scrapy.Request(
    url="https://example.org",
    meta={
        "camoufox": True,
        "camoufox_context": "awesome_context",
    },
)
```

### `camoufox_context_kwargs`
Type `dict`, default `{}`

A dictionary with keyword arguments to be used when creating a new context, if a context
with the name specified in the `camoufox_context` meta key does not exist already.
See the section on [browser contexts](#browser-contexts) for more information.

```python
return scrapy.Request(
    url="https://example.org",
    meta={
        "camoufox": True,
        "camoufox_context": "awesome_context",
        "camoufox_context_kwargs": {
            "ignore_https_errors": True,
        },
    },
)
```

### `camoufox_include_page`
Type `bool`, default `False`

If `True`, the [Camoufox page](https://playwright.dev/python/docs/api/class-page)
that was used to download the request will be available in the callback at
`response.meta['camoufox_page']`. If `False` (or unset) the page will be
closed immediately after processing the request.

**Important!**

This meta key is entirely optional, it's NOT necessary for the page to load or for any
asynchronous operation to be performed (specifically, it's NOT necessary for `PageMethod`
objects to be applied). Use it only if you need access to the Page object in the callback
that handles the response.

For more information and important notes see
[Receiving Page objects in callbacks](#receiving-page-objects-in-callbacks).

```python
return scrapy.Request(
    url="https://example.org",
    meta={"camoufox": True, "camoufox_include_page": True},
)
```

### `camoufox_page_event_handlers`
Type `Dict[Str, Callable]`, default `{}`

A dictionary of handlers to be attached to page events.
See [Handling page events](#handling-page-events).

### `camoufox_page_init_callback`
Type `Optional[Union[Callable, str]]`, default `None`

A coroutine function (`async def`) to be invoked for newly created pages.
Called after attaching page event handlers & setting up internal route
handling, before making any request. It receives the Camoufox page and the
Scrapy request as positional arguments. Useful for initialization code.
Ignored if the page for the request already exists (e.g. by passing
`camoufox_page`).

```python
async def init_page(page, request):
    await page.add_init_script(path="./custom_script.js")

class AwesomeSpider(scrapy.Spider):
    def start_requests(self):
        yield scrapy.Request(
            url="https://httpbin.org/headers",
            meta={
                "camoufox": True,
                "camoufox_page_init_callback": init_page,
            },
        )
```

**Important!**

`scrapy-camoufox` uses `Page.route` & `Page.unroute` internally, avoid using
these methods unless you know exactly what you're doing.

### `camoufox_page_methods`
Type `Iterable[PageMethod]`, default `()`

An iterable of [`scrapy_camoufox.page.PageMethod`](#pagemethod-class)
objects to indicate actions to be performed on the page before returning the
final response. See [Executing actions on pages](#executing-actions-on-pages).

### `camoufox_page`
Type `Optional[playwright.async_api.Page]`, default `None`

A [Camoufox page](https://playwright.dev/python/docs/api/class-page) to be used to
download the request. If unspecified, a new page is created for each request.
This key could be used in conjunction with `camoufox_include_page` to make a chain of
requests using the same page. For instance:

```python
from playwright.async_api import Page

def start_requests(self):
    yield scrapy.Request(
        url="https://httpbin.org/get",
        meta={"camoufox": True, "camoufox_include_page": True},
    )

def parse(self, response, **kwargs):
    page: Page = response.meta["camoufox_page"]
    yield scrapy.Request(
        url="https://httpbin.org/headers",
        callback=self.parse_headers,
        meta={"camoufox": True, "camoufox_page": page},
    )
```

### `camoufox_page_goto_kwargs`
Type `dict`, default `{}`

A dictionary with keyword arguments to be passed to the page's
[`goto` method](https://playwright.dev/python/docs/api/class-page#page-goto)
when navigating to an URL. The `url` key is ignored if present, the request
URL is used instead.

```python
return scrapy.Request(
    url="https://example.org",
    meta={
        "camoufox": True,
        "camoufox_page_goto_kwargs": {
            "wait_until": "networkidle",
        },
    },
)
```

### `camoufox_security_details`
Type `Optional[dict]`, read only

A dictionary with [security information](https://playwright.dev/python/docs/api/class-response#response-security-details)
about the give response. Only available for HTTPS requests. Could be accessed
in the callback via `response.meta['camoufox_security_details']`

```python
def parse(self, response, **kwargs):
    print(response.meta["camoufox_security_details"])
    # {'issuer': 'DigiCert TLS RSA SHA256 2020 CA1', 'protocol': 'TLS 1.3', 'subjectName': 'www.example.org', 'validFrom': 1647216000, 'validTo': 1678838399}
```

### `camoufox_suggested_filename`
Type `Optional[str]`, read only

The value of the [`Download.suggested_filename`](https://playwright.dev/python/docs/api/class-download#download-suggested-filename)
attribute when the response is the binary contents of a
[download](https://playwright.dev/python/docs/downloads) (e.g. a PDF file).
Only available for responses that only caused a download. Can be accessed
in the callback via `response.meta['camoufox_suggested_filename']`

```python
def parse(self, response, **kwargs):
    print(response.meta["camoufox_suggested_filename"])
    # 'sample_file.pdf'
```

## Receiving Page objects in callbacks

Specifying a value that evaluates to `True` in the
[`camoufox_include_page`](#camoufox_include_page) meta key for a
request will result in the corresponding `playwright.async_api.Page` object
being available in the `playwright_page` meta key in the request callback.
In order to be able to `await` coroutines on the provided `Page` object,
the callback needs to be defined as a coroutine function (`async def`).

**Caution**

Use this carefully, and only if you really need to do things with the Page
object in the callback. If pages are not properly closed after they are no longer
necessary the spider job could get stuck because of the limit set by the
`CAMOUFOX_MAX_PAGES_PER_CONTEXT` setting.

```python
from playwright.async_api import Page
import scrapy

class AwesomeSpiderWithPage(scrapy.Spider):
    name = "page_spider"

    def start_requests(self):
        yield scrapy.Request(
            url="https://example.org",
            callback=self.parse_first,
            meta={"camoufox": True, "camoufox_include_page": True},
            errback=self.errback_close_page,
        )

    def parse_first(self, response):
        page: Page = response.meta["camoufox_page"]
        return scrapy.Request(
            url="https://example.com",
            callback=self.parse_second,
            meta={"camoufox": True, "camoufox_include_page": True, "camoufox_page": page},
            errback=self.errback_close_page,
        )

    async def parse_second(self, response):
        page: Page = response.meta["camoufox_page"]
        title = await page.title()  # "Example Domain"
        await page.close()
        return {"title": title}

    async def errback_close_page(self, failure):
        page: Page = failure.request.meta["camoufox_page"]
        await page.close()
```

**Notes:**

* When passing `camoufox_include_page=True`, make sure pages are always closed
  when they are no longer used. It's recommended to set a Request errback to make
  sure pages are closed even if a request fails (if `camoufox_include_page=False`
  pages are automatically closed upon encountering an exception).
  This is important, as open pages count towards the limit set by
  `CAMOUFOX_MAX_PAGES_PER_CONTEXT` and crawls could freeze if the limit is reached
  and pages remain open indefinitely.
* Defining callbacks as `async def` is only necessary if you need to `await` things,
  it's NOT necessary if you just need to pass over the Page object from one callback
  to another (see the example above).
* Any network operations resulting from awaiting a coroutine on a Page object
  (`goto`, `go_back`, etc) will be executed directly by Camoufox, bypassing the
  Scrapy request workflow (Scheduler, Middlewares, etc).


## Browser contexts

Multiple [browser contexts](https://playwright.dev/python/docs/browser-contexts)
to be launched at startup can be defined via the
[`CAMOUFOX_CONTEXTS`](#camoufox_contexts) setting.

### Choosing a specific context for a request

Pass the name of the desired context in the `camoufox_context` meta key:

```python
yield scrapy.Request(
    url="https://example.org",
    meta={"camoufox": True, "camoufox_context": "first"},
)
```

### Default context

If a request does not explicitly indicate a context via the `camoufox_context`
meta key, it falls back to using a general context called `default`. This `default`
context can also be customized on startup via the `CAMOUFOX_CONTEXTS` setting.

### Persistent contexts

Pass a value for the `user_data_dir` keyword argument to launch a context as
persistent. See also [`BrowserType.launch_persistent_context`](https://playwright.dev/python/docs/api/class-browsertype#browser-type-launch-persistent-context).

Note that persistent contexts are launched independently from the main browser
instance, hence keyword arguments passed in the
[`CAMOUFOX_LAUNCH_OPTIONS`](#camoufox_launch_options)
setting do not apply.

### Creating contexts while crawling

If the context specified in the `camoufox_context` meta key does not exist, it will be created.
You can specify keyword arguments to be passed to
[`Browser.new_context`](https://playwright.dev/python/docs/api/class-browser#browser-new-context)
in the `camoufox_context_kwargs` meta key:

```python
yield scrapy.Request(
    url="https://example.org",
    meta={
        "camoufox": True,
        "camoufox_context": "new",
        "camoufox_context_kwargs": {
            "java_script_enabled": False,
            "ignore_https_errors": True,
        },
    },
)
```

Please note that if a context with the specified name already exists,
that context is used and `camoufox_context_kwargs` are ignored.

### Closing contexts while crawling

After [receiving the Page object in your callback](#receiving-page-objects-in-callbacks),
you can access a context though the corresponding [`Page.context`](https://playwright.dev/python/docs/api/class-page#page-context)
attribute, and await [`close`](https://playwright.dev/python/docs/api/class-browsercontext#browser-context-close) on it.

```python
def parse(self, response, **kwargs):
    yield scrapy.Request(
        url="https://example.org",
        callback=self.parse_in_new_context,
        errback=self.close_context_on_error,
        meta={
            "camoufox": True,
            "camoufox_context": "awesome_context",
            "camoufox_include_page": True,
        },
    )

async def parse_in_new_context(self, response):
    page = response.meta["camoufox_page"]
    title = await page.title()
    await page.close()
    await page.context.close()
    return {"title": title}

async def close_context_on_error(self, failure):
    page = failure.request.meta["camoufox_page"]
    await page.close()
    await page.context.close()
```

### Avoid race conditions & memory leaks when closing contexts
Make sure to close the page before closing the context. See
[this comment](https://github.com/scrapy-plugins/scrapy-playwright/issues/191#issuecomment-1548097114)
in [#191](https://github.com/scrapy-plugins/scrapy-playwright/issues/191)
for more information.

### Maximum concurrent context count

Specify a value for the `CAMOUFOX_MAX_CONTEXTS` setting to limit the amount
of concurent contexts. Use with caution: it's possible to block the whole crawl
if contexts are not closed after they are no longer used (refer to
[this section](#closing-contexts-while-crawling) to dinamically close contexts).
Make sure to define an errback to still close contexts even if there are errors.

## Executing actions on pages

A sorted iterable (e.g. `list`, `tuple`, `dict`) of `PageMethod` objects
could be passed in the `camoufox_page_methods`
[Request.meta](https://docs.scrapy.org/en/latest/topics/request-response.html#scrapy.http.Request.meta)
key to request methods to be invoked on the `Page` object before returning the final
`Response` to the callback.

This is useful when you need to perform certain actions on a page (like scrolling
down or clicking links) and you want to handle only the final result in your callback.

### `PageMethod` class

#### `scrapy_camoufox.page.PageMethod(method: str | callable, *args, **kwargs)`:

Represents a method to be called (and awaited if necessary) on a
`playwright.page.Page` object (e.g. "click", "screenshot", "evaluate", etc).
It's also possible to pass callable objects that will be invoked as callbacks
and receive Playwright Page as argument.
`method` is the name of the method, `*args` and `**kwargs`
are passed when calling such method. The return value
will be stored in the `PageMethod.result` attribute.

For instance:
```python
def start_requests(self):
    yield Request(
        url="https://example.org",
        meta={
            "camoufox": True,
            "camoufox_page_methods": [
                PageMethod("screenshot", path="example.png", full_page=True),
            ],
        },
    )

def parse(self, response, **kwargs):
    screenshot = response.meta["camoufox_page_methods"][0]
    # screenshot.result contains the image's bytes
```

produces the same effect as:
```python
def start_requests(self):
    yield Request(
        url="https://example.org",
        meta={"camoufox": True, "camoufox_include_page": True},
    )

async def parse(self, response, **kwargs):
    page = response.meta["camoufox_page"]
    screenshot = await page.screenshot(path="example.png", full_page=True)
    # screenshot contains the image's bytes
    await page.close()
```

### Passing callable objects

If a `PageMethod` receives a callable object as its first argument, it will be
called with the page as its first argument. Any additional arguments are passed
to the callable after the page.

```python
async def scroll_page(page: Page) -> str:
    await page.wait_for_selector(selector="div.quote")
    await page.evaluate("window.scrollBy(0, document.body.scrollHeight)")
    await page.wait_for_selector(selector="div.quote:nth-child(11)")
    return page.url


class MySpyder(scrapy.Spider):
    name = "scroll"

    def start_requests(self):
        yield Request(
            url="https://quotes.toscrape.com/scroll",
            meta={
                "camoufox": True,
                "camoufox_page_methods": [PageMethod(scroll_page)],
            },
        )
```

### Supported Playwright methods

Refer to the [upstream docs for the `Page` class](https://playwright.dev/python/docs/api/class-page)
to see available methods.

### Impact on Response objects

Certain `Response` attributes (e.g. `url`, `ip_address`) reflect the state after the last
action performed on a page. If you issue a `PageMethod` with an action that results in
a navigation (e.g. a `click` on a link), the `Response.url` attribute will point to the
new URL, which might be different from the request's URL.


## Handling page events

A dictionary of Page event handlers can be specified in the `camoufox_page_event_handlers`
[Request.meta](https://docs.scrapy.org/en/latest/topics/request-response.html#scrapy.http.Request.meta) key.
Keys are the name of the event to be handled (e.g. `dialog`, `download`, etc).
Values can be either callables or strings (in which case a spider method with the name will be looked up).

Example:

```python
from playwright.async_api import Dialog

async def handle_dialog(dialog: Dialog) -> None:
    logging.info(f"Handled dialog with message: {dialog.message}")
    await dialog.dismiss()

class EventSpider(scrapy.Spider):
    name = "event"

    def start_requests(self):
        yield scrapy.Request(
            url="https://example.org",
            meta={
                "camoufox": True,
                "camoufox_page_event_handlers": {
                    "dialog": handle_dialog,
                    "response": "handle_response",
                },
            },
        )

    async def handle_response(self, response: PlaywrightResponse) -> None:
        logging.info(f"Received response with URL {response.url}")
```

See the [upstream `Page` docs](https://playwright.dev/python/docs/api/class-page)
for a list of the accepted events and the arguments passed to their handlers.

### Notes about page event handlers

* Event handlers will remain attached to the page and will be called for
  subsequent downloads using the same page unless they are
  [removed later](https://playwright.dev/python/docs/events#addingremoving-event-listener).
  This is usually not a problem, since by default requests are performed in
  single-use pages.
* Event handlers will process Camoufox objects, not Scrapy ones. For example,
  for each Scrapy request/response there will be a matching Camoufox
  request/response, but not the other way: background requests/responses to get
  images, scripts, stylesheets, etc are not seen by Scrapy.


## Memory usage extension

The default Scrapy memory usage extension
(`scrapy.extensions.memusage.MemoryUsage`) does not include the memory used by
Playwright because the browser is launched as a separate process. The
scrapy-camoufox package provides a replacement extension which also considers
the memory used by Camoufox. This extension needs the
[`psutil`](https://pypi.org/project/psutil/) package to work.

Update the [EXTENSIONS](https://docs.scrapy.org/en/latest/topics/settings.html#std-setting-EXTENSIONS)
setting to disable the built-in Scrapy extension and replace it with the one
from the scrapy-camoufox package:

```python
# settings.py
EXTENSIONS = {
    "scrapy.extensions.memusage.MemoryUsage": None,
    "scrapy_camoufox.memusage.ScrapyPlaywrightMemoryUsageExtension": 0,
}
```

Refer to the
[upstream docs](https://docs.scrapy.org/en/latest/topics/extensions.html#module-scrapy.extensions.memusage)
for more information about supported settings.

### Windows support

Just like the [upstream Scrapy extension](https://docs.scrapy.org/en/latest/topics/extensions.html#module-scrapy.extensions.memusage), this custom memory extension does not work
on Windows. This is because the stdlib [`resource`](https://docs.python.org/3/library/resource.html)
module is not available.


## Examples

**Click on a link, save the resulting page as PDF**

```python
class ClickAndSavePdfSpider(scrapy.Spider):
    name = "pdf"

    def start_requests(self):
        yield scrapy.Request(
            url="https://example.org",
            meta=dict(
                camoufox=True,
                camoufox_page_methods={
                    "click": PageMethod("click", selector="a"),
                    "pdf": PageMethod("pdf", path="/tmp/file.pdf"),
                },
            ),
        )

    def parse(self, response, **kwargs):
        pdf_bytes = response.meta["camoufox_page_methods"]["pdf"].result
        with open("iana.pdf", "wb") as fp:
            fp.write(pdf_bytes)
        yield {"url": response.url}  # response.url is "https://www.iana.org/domains/reserved"
```

**Scroll down on an infinite scroll page, take a screenshot of the full page**

```python
class ScrollSpider(scrapy.Spider):
    name = "scroll"

    def start_requests(self):
        yield scrapy.Request(
            url="http://quotes.toscrape.com/scroll",
            meta=dict(
                camoufox=True,
                camoufox_include_page=True,
                camoufox_page_methods=[
                    PageMethod("wait_for_selector", "div.quote"),
                    PageMethod("evaluate", "window.scrollBy(0, document.body.scrollHeight)"),
                    PageMethod("wait_for_selector", "div.quote:nth-child(11)"),  # 10 per page
                ],
            ),
        )

    async def parse(self, response, **kwargs):
        page = response.meta["camoufox_page"]
        await page.screenshot(path="quotes.png", full_page=True)
        await page.close()
        return {"quote_count": len(response.css("div.quote"))}  # quotes from several pages
```

## Credits 
[daijro](https://github.com/daijro) for [Camoufox](https://github.com/daijro/camoufox)  <br>
[elacuesta](https://github.com/elacuesta) for [Scrapy-Playwright](https://github.com/scrapy-plugins/scrapy-playwright)


# Cuprite public API review

State: `main` at `a0e6c48` (after `Driver#console`), Ferrum 0.20.0, Capybara driver contract from the installed
capybara gem.

Method: diffed `Driver`/`Node` public methods against `Capybara::Driver::Base`/`Capybara::Driver::Node`, diffed
`Browser`/`Page` against `Ferrum::Browser`/`Ferrum::Page`, grepped lib/spec/docs for every extra, and called the
suspicious ones.

Deprecation warnings say the method will be removed in 0.21 (`0864fa3`).

Verdicts: **Keep**, **Fix** (broken or inconsistent), **Deprecate** (warn in 0.20, remove in 0.21), **Hide** (make private
or move out of the public surface), **Delete** (dead code, nothing to deprecate).

Status: ✅ done (with commit), ⏳ not started. *differs:* notes say where the outcome differs from the verdict. `docs/` was
updated for all deprecations in `a9cc52c`.

---

## 1. Broken or dead code

| What | Where | Finding | Verdict | Status |
|---|---|---|---|---|
| `Driver#source` | `driver.rb:68`, `browser.rb:166` | Calls `Browser#source`, which is `raise NotImplementedError`. Always raises. Poltergeist returned the page source before JS ran | **Fix** (return the main document's response body from `network`) or **Delete** | ✅ `25fea71` deleted. Capybara's `Session#source` is an alias of `html` and never calls the driver's |
| `Capybara::Cuprite::Cookie` | `lib/capybara/cuprite/cookie.rb` | Never required anywhere since the initial commit (2018). `Driver#cookies` returns Ferrum cookies | **Delete** | ✅ `dc1acf7`, README now names `Ferrum::Cookies::Cookie` |
| `Page#frame_name` | `page.rb:152` | Unused in lib and spec | **Delete** | ✅ `aa99336` |
| `Browser#browser_error` | `browser.rb:162` | Only used by one spec to trigger a JS error (`_cuprite.browserError()`) | **Delete**, inline the JS in the spec | ✅ `19758b6`, also removed `browserError()` from `index.js` |
| `Node#as_json` / `Node#to_json` | `node.rb:268-276` | `@api private`, has a `FIXME: Where is this method used`. Not used in lib or spec | **Delete** | ✅ `aafb699` |
| README `page.driver.reset` | `README.md:181` | Method doesn't exist, it's `reset!` | **Fix** docs | ✅ `1655423` |

## 2. Poltergeist leftovers on `Driver`

Everything below is Cuprite-only (not in Capybara's driver contract) and dates back to Poltergeist.

**Policy** (agreed after §2.1 and §2.3): `Driver` keeps only (1) Capybara's driver contract and (2) what needs
Capybara or driver context that Ferrum lacks: node conversion, frames and windows by handle, modals, per-test
`reset!` of Cuprite options, `Capybara.save_path` defaults. Plain conveniences over Ferrum get deprecated in favor of
`page.driver.browser.*`. When a wrapper only hides a Ferrum sharp edge, fix Ferrum first, then deprecate the wrapper.

Checked that `reset!` is no reason to keep a wrapper: headers, cookies, `network.authorize`, `intercept` +
`on(:request)`, `evaluate_on_new_document`, `offline_mode`, `network.traffic` and console messages set through
`page.driver.browser` are all gone after `reset!` (the browser context is disposed). Only Browser-level `timeout=`
survives, and `Driver#timeout=` behaves the same.

*Correction:* `body`, `back`/`forward` and `window_handle` are not Poltergeist leftovers. Until `65083d9` (2021,
"Delegate methods to browser") `Driver` only had Capybara's names as wrappers; that refactor delegated Ferrum's names
and turned Capybara's into aliases, which made the Ferrum names public by accident.

### 2.1 Duplicate names

| Methods | Finding | Verdict | Status |
|---|---|---|---|
| `body` + `html` | `html` is Capybara's, `body` is a Ferrum delegate | **Deprecate** `body` | ✅ `098251b`, adds the `Driver.deprecate` helper |
| `back`/`forward` + `go_back`/`go_forward` | Capybara uses `go_*` | **Deprecate** `back`/`forward` | ✅ `8798e0c` |
| `window_handle` + `current_window_handle` | Capybara uses `current_window_handle` | **Deprecate** `window_handle` | ✅ `7b86558`; `a1e7d89` renamed `Browser#window_handle` to `current_window_handle`, `d439402` added the deprecated fallback |
| `render` + `save_screenshot` | `render` is Poltergeist's alias | **Deprecate** `render` | ✅ `23aecc7` |
| `resize` + `resize_window` + `resize_window_to` | Capybara uses `resize_window_to(handle, w, h)`. Two Poltergeist aliases on top | **Keep** `resize` (no handle, documented), **Deprecate** `resize_window` | ✅ `b81b288`, *differs:* both deprecated in favor of `page.current_window.resize_to`. `resize` was documented in `docs/9-windows-frames.md` (not the README); Capybara's `Window#resize_to` covers it and also waits for a stable size |
| `basic_authorize` + `authorize` | Same method twice | **Deprecate** `authorize` | ✅ `f8c804a`, *differs:* `authorize` kept (Ferrum's name), `basic_authorize` deprecated. Capybara has no API for it; both names come from Rack::Test |
| `pause` + `debug` | `debug` opens DevTools and can drop into a binding. `pause` is the Poltergeist "press enter or SIGCONT" loop, 35 lines with a thread and a signal trap | **Deprecate** `pause` | ✅ `dded317`, *differs:* removed outright; `binding.irb` (with `headless: false`) or `debug` replace it |

### 2.2 Value wrappers vs Ferrum objects

New APIs return Ferrum objects (`downloads`, `console`). The old ones wrap Ferrum and return plain values, so the
two halves of the API look different and the old half hides most of Ferrum:

| Driver method | Returns | Ferrum object it wraps |
|---|---|---|
| `headers`, `headers=`, `add_headers`, `add_header` | `Hash` | `browser.headers` (`Ferrum::Headers`) |
| `cookies`, `set_cookie`, `remove_cookie`, `clear_cookies` | `Hash` of `Ferrum::Cookies::Cookie` | `browser.cookies` (`Ferrum::Cookies`) |
| `network_traffic`, `clear_network_traffic`, `clear_memory_cache`, `wait_for_network_idle` | `Array`/values | `browser.network` (`Ferrum::Network`) |
| `response_headers`, `status_code` | values | `browser.network`, but both are Capybara's contract |
| `downloads`, `console` | Ferrum objects | `browser.downloads`, `browser.console` |

Quirks in the wrappers:

- `network_traffic` with no argument drops blocked requests; you need `network_traffic(:all)` to get everything.
  Surprising default, and different from `browser.network.traffic`.
- `set_cookie(name, value, options)` lets `options[:name]`/`options[:value]` override the positional arguments
  (`||=`).
- `remove_cookie` fills in the default domain only when no options at all are passed, so `remove_cookie("a",
  path: "/x")` silently has no domain.
- `page.driver.headers` (a `Hash`) and `page.driver.browser.headers` (a `Ferrum::Headers`) share a name but are
  different things.

**Verdict** (replaces "expose `network`/`headers`/`cookies` as Ferrum objects on the driver in 1.0", which would have
added three more delegates the policy says not to have):

| Methods | Verdict | Replacement |
|---|---|---|
| `headers`, `headers=`, `add_headers`, `add_header` | **Deprecate** | `browser.headers.get` / `.set` / `.add`. ✅ `4cbf112`; `headers.add` needs braces (`add({ "A" => "b" })`), otherwise Ruby 3 reads the hash as keywords |
| `cookies`, `set_cookie`, `remove_cookie`, `clear_cookies` | **Deprecate** | `browser.cookies.all` / `.set` / `.remove` / `.clear`. ✅ `9bc6de9`; docs pass `domain:` explicitly (see below) |
| `network_traffic`, `clear_network_traffic` | **Deprecate** | `browser.network.traffic`, `.clear(:traffic)`. ✅ `c7641f6` |
| `clear_memory_cache`, `wait_for_network_idle` | **Deprecate** | `browser.network.clear(:cache)`, `.wait_for_idle`. ✅ `c7641f6` |
| `response_headers`, `status_code` | **Keep** | Capybara's contract |
| `downloads`, `console` | **Delete** before release | `browser.downloads`, `browser.console`. Both are unreleased (CHANGELOG Unreleased/Added), so no deprecation needed; the `save_path` default for downloads is set in `Driver#initialize` and stays. ✅ `a772ce1` |

The quirks go away with the wrappers, nothing to fix separately. Deprecated methods live in
`lib/capybara/cuprite/deprecations.rb` (`8e4de05`), included into `Driver`, so 1.0 deletes one file.

Cookie domain: Ferrum's `cookies.set`/`remove` default `domain:` to the host of the `:base_url` option, not the
current page, and Cuprite doesn't set `base_url`, so without `domain:` Chrome answers "Invalid parameters".
`cookies.set(url: ...)` doesn't help either: Ferrum still sends `domain: nil` and Chrome rejects it. Ferrum
candidates: skip `domain` when `url` is given, and fall back to the current page's host.
✅ `c056532`, *differs:* fixed in Cuprite instead, Ferrum unchanged: `Driver` sets `browser.base_url` from `Capybara.app_host` (or the server) on start and `reset!`, and to the URL on every `visit`.

**Status:** ✅ (`8e4de05`, `4cbf112`, `9bc6de9`, `c7641f6`)

### 2.3 Screenshot state on the driver

- `zoom_factor=` only sets `scale:` for later screenshots. The name suggests page zoom.
- `paper_size=` stores `{width:, height:}` for later PDFs. Ferrum's `pdf` already takes `paperWidth`/`paperHeight`
  and `format:`.
- Both are hidden driver state, reset in `reset!`, duplicating options that `save_screenshot`/`render_base64`
  already accept.

**Verdict:** **Deprecate** both; pass `scale:` and the paper options to `save_screenshot`/`render_base64`.

**Status:** ✅ `e0b39a9`, deprecated in favor of `scale:`, `paper_width:`/`paper_height:` (or `format: :A4`) on
`save_screenshot`. No issue, PR or public code found using either setter.

Found along the way:

- `format: :pdf` (`save_screenshot(path, format: :pdf)`, `render_base64(:pdf)`) worked before Ferrum, broke with the
  Ferrum 0.6 bump (`da05cd5`, Oct 2019), when Ferrum started reading `format:` as a paper size (`KeyError`). Nobody
  reported it. ✅ `344b340` removed it; only a `.pdf` path renders a PDF, and `render_base64` lost its dead PDF branch.
- `render_base64` never applied `zoom_factor=`. Not fixed on purpose, both are deprecated.

### 2.4 Other Poltergeist-only methods

| Method | Finding | Verdict | Status |
|---|---|---|---|
| `click(x, y)` | Clicks page coordinates. Capybara has no equivalent. `browser.mouse.click(x:, y:)` does the same | **Deprecate** in favor of `browser.mouse.click` | ✅ `4d67d3c` |
| `scroll_to(left, top)` | Window scroll. Name collides with Capybara's `Session#scroll_to(element/position)`, which goes through `Node#scroll_to` and has different arguments | **Deprecate**, `browser.mouse.scroll_to` covers it | ✅ `4d67d3c`, *differs:* replacement is Capybara's `page.scroll_to` |
| `set_proxy(...)` | Only merges into `@options`, so it has an effect only before the browser starts (documented in `docs/15-proxy.md`). A runtime-looking setter that isn't one | **Deprecate** in favor of the `:proxy` option | ✅ `8942a5a` |
| `render_base64(format, options)` | Positional format plus options hash, unlike `save_screenshot(path, options)` | **Keep**, accept `format:` as a keyword too | ✅ `503c79e`, *differs:* deprecated in favor of `page.driver.browser.screenshot` / `.pdf`, which return Base64 without a `path:` |
| `refresh` | Capybara's contract (`Session#refresh` calls `driver.refresh`) | **Keep** | — |
| `wait_for_reload` | Thin Ferrum delegate, documented | **Deprecate**, `browser.wait_for_reload` | ✅ `b2fdad1` |
| `timeout`, `timeout=` | Thin Ferrum delegates, documented. Not reset by `reset!` either way | **Deprecate**, `browser.timeout` / `browser.timeout=` (or the `:timeout` option) | ✅ `b2fdad1` |
| `device_pixel_ratio`, `viewport_size` | Ferrum delegates, used in specs, undocumented | **Deprecate**, `browser.device_pixel_ratio` / `browser.viewport_size` (was: keep and document) | ✅ `b2fdad1` |
| `authorize(user, password)` | Exists only because `browser.network.authorize` requires a block (raises `AUTHORIZE_BLOCK_MISSING` unless something already handles `Fetch.requestPaused`), and the obvious block `{ \|r\| r.continue }` breaks once Rubocop turns it into `&:continue` (Ferrum calls it with 3 arguments) | **Fix Ferrum** (block optional, continue by default), then **Deprecate** in favor of `browser.network.authorize(user:, password:)`. The `basic_authorize` deprecation (`f8c804a`) is unreleased, so point it straight at the Ferrum call so users migrate once | ✅ `788e52f`, *differs:* Ferrum unchanged (the required block is intended), both names deprecated, docs show the 3-argument block |
| `debug` | Thin delegate to `Ferrum::Browser#debug`, but it's the documented debugging entry point (`docs/4-debugging.md`) and what `pause` users were sent to in `dded317` | **Keep** as a deliberate exception | — |
| `restart`, `quit` | Delegates. `quit` is called by `Capybara::Session#quit`; `restart` isn't called by Capybara | **Keep** `quit`, **Deprecate** `restart` | ✅ `857fe65`, `restart` deprecated in favor of `browser.restart` |
| `within_window(handle)` | Public by accident through the delegate list. Capybara's `within_window` uses `switch_to_window`; only `resize_window_to`, `window_size` and `fullscreen_window` call it | **Hide** | ✅ `e2a047c` made private, `6d3a22c` moved the body from `Browser` into `Driver` |
| `options`, `screen_size`, `app` | Public readers. `options` is the mutable hash that `set_proxy` mutates | **Hide** `options` (or freeze a copy) once `set_proxy` is gone | ✅ `7dca253`, *differs:* only `screen_size` made private; `options` and `app` stay public like Selenium's and RackTest's drivers, `options` stays mutable |
| `find(method, selector)` | Public helper behind `find_css`/`find_xpath` | **Hide** | ✅ `7dca253` |

## 3. `Node`

| Method | Finding | Verdict |
|---|---|---|
| `command(name, *args)` | Public dispatcher that does `browser.send(name, node, ...)`. Internal | **Hide** ✅ `7dca253` |
| `node`, `description` | Ferrum node and its CDP description. `node` is the useful escape hatch | **Keep** `node`, **Hide** `description` ✅ `7dca253` |
| `send_key` | Poltergeist alias of `send_keys` | **Deprecate** ✅ `50e419d` |
| `parents` | Poltergeist-only, used by one spec. Capybara has `ancestor` | **Deprecate** ✅ `398fa77` |
| `property(name)` | Not in Capybara's contract, same as Ferrum's `node.property` | **Deprecate** ✅ `32aff0f` (documented in `989fc91`, then undocumented) |
| `attributes` | Not in Capybara's contract, Ferrum has no method returning the whole hash | **Keep**, document ✅ `2665ee6` (briefly deprecated in `32aff0f`) |
| `find(method, selector)` | Internal helper behind `find_css`/`find_xpath` | **Hide** ✅ `e2a047c` |
| `browser` | Delegate to the driver, `page.driver.browser` is the same object | **Deprecate** ✅ `857fe65`; internal calls use `driver.browser` |
| `drag_by(x, y, **)` | Poltergeist-only, no Ferrum counterpart, documented | **Keep**; docs example fixed to `.native.drag_by` in `e8db986` |
| `set(value, **options)` | Warns and ignores every option. Selenium supports `clear: :backspace / :none / [keys]` and `rapid:` | **Fix**: support at least `clear:` (see §5). *Out of scope:* cleanup only, no new features |
| `style(styles)` | Not implemented, falls through to `Capybara::Driver::Node#style`, which raises `NotImplementedError` | **Fix** (see §5). *Out of scope:* cleanup only, no new features |

## 4. `Browser`/`Page`: internals on Ferrum's public surface

`Capybara::Cuprite::Browser` adds **52 public methods** to `Ferrum::Browser`'s 93. Most are node helpers that exist
only so `Node#command` can call `browser.send(name, node, ...)`:

`all_text attribute attributes before_click delete_text disabled? drag drag_by drag_html5 drop find find_within
hover obscured? parents path property select select_file send_keys set trigger value visible? visible_text`

plus the modal plumbing (`accept_confirm accept_prompt dismiss_confirm dismiss_prompt find_modal reset_modals`) and
`browser_error` (✅ removed in `19758b6`).

Problems:

- Anyone using `page.driver.browser` as "a Ferrum browser" sees `browser.value(node)`, `browser.set(node, v)`,
  `browser.select(node, v)` next to Ferrum's own API, with nothing marking them as internal.
- `Browser#evaluate_async(script, wait, *args)` overrides Ferrum's, with callback semantics written on top of the new
  JS API. Ferrum's own legacy runtime has the same signature and semantics, so there are two implementations of one
  contract. ✅ `779adfd` renamed it to the private `evaluate_callback`, so `browser.evaluate_async` is Ferrum's again
  (with Ferrum's own deprecation warning).
- `Page` overrides `at_css at_xpath css xpath body current_url current_title evaluate execute execution_id` to route
  them to the active frame. That's intentional for frame switching, but it means `page.driver.browser.page.css(...)`
  quietly searches a different document than `Ferrum::Page#css` would.

**Verdict:** **Hide**. Move the node helpers into a module that `Node` calls (or into `Node` itself, most are a single
`node.evaluate("_cuprite.x(this)")`), and make the modal plumbing private or `@api private`. No user-facing change if
done before anyone depends on them, and it shrinks `Browser` to what it actually adds over Ferrum: frames, windows,
URL filtering and modals.

**Status:** ✅ `597c5b9` removed unused `reset_modals`; `835e061` made the node helpers private on `Browser` (`Node#command`
uses `send`; `Page` keeps its methods public because Forwardable warns when delegating to private ones); `c362816` made
`Driver` call modal and frame methods on `browser.page` and dropped those `Browser` delegates.

Second pass, methods on `Browser` that only served `Driver`:

| Method | Verdict | Status |
|---|---|---|
| `visit` | **Deprecate**, Ferrum's `goto`; `Driver#visit` calls `browser.goto` | ✅ `6dd6d74` |
| `status_code` | **Deprecate**, Ferrum's `network.status`; `Driver#status_code` calls it | ✅ `6dd6d74` |
| `window_handle` | **Deprecate**, renamed to `current_window_handle` in `a1e7d89` | ✅ `d439402` |
| `find` | **Hide**, `Driver#find` uses `send` | ✅ `6d3a22c` |
| `active_element` | **Delete**, `Driver#active_element` evaluates `document.activeElement` | ✅ `6d3a22c` |
| `within_window` | **Delete**, moved into `Driver` as private | ✅ `6d3a22c` |
| `evaluate_async` | **Hide**, renamed to `evaluate_callback` | ✅ `779adfd` |

`Browser`'s public additions over Ferrum are now: windows (`current_window_handle`, `window_handles`,
`switch_to_window`, `close_window`), URL filtering (`url_allowlist`/`url_blocklist` and the old names), modal settings
(`raise_on_unhandled_modal`), and overrides of `command`, `reset`, `resize`, `quit`. The `Page` frame overrides stay
as they are.

## 5. Missing

| Feature | Evidence | Suggestion |
|---|---|---|
| `Node#style` | `spec/spec_helper.rb` skips Capybara's `#assert_matches_style` and `#has_css? :style option should support Hash` shared specs | `getComputedStyle(this)` for the requested properties |
| `Node#set` `clear:` / `rapid:` | `node.rb:97` warns and ignores options | `clear: :backspace` via `delete_text` + keys, `clear: :none` to append |
| `response_headers` in Capybara's suite | `spec/features/session_spec.rb:3` passes `capybara_skip: %i[response_headers]` | Find out why it's skipped (header name casing?) and either fix or note it |
| `#has_field with valid`, `#has_element?` | Skipped in `spec/spec_helper.rb` with no reason given | Investigate, add the reason next to the skip |
| A documented escape hatch | `page.driver.browser` was only briefly introduced in `docs/1-introduction.md` ("Ferrum under the hood"), and the policy makes it the main API beyond Capybara | One docs section: "`page.driver.browser` is a `Ferrum::Browser`, everything in Ferrum's docs applies", written before the §2.2/§2.4 deprecations so their warnings can point at it. ✅ `066ad80`, expanded that section |
| Docs for undocumented extras | `Node#property`, `Node#attributes` (`device_pixel_ratio`/`viewport_size` are deprecated instead, §2.4) | Superseded: both deprecated (`attributes` in `98b586d`, docs entry removed) |

## 6. Order

1. **0.20 (Unreleased):** dead code deleted (§1) ✅, unreleased `downloads`/`console` dropped ✅, internals hidden
   (§2.4, §3, §4) ✅, `page.driver.browser` documented ✅, every deprecation above warns that it goes in 0.21 ✅.
2. **0.21:** delete every `*/deprecations.rb` (`driver/`, `node/`, `browser/`) and `deprecation.rb`, the `@zoom_factor`/
   `@paper_size` handling in `Driver#reset!`/`save_screenshot`, and the `parents`/`getAttributes` JS helpers.
   `Extensions` modules stay. The old `0.21` branch (`fbfd80c`) was dropped; redo the removal from `main` at release.
   ✅ `115f990`, also the docs notes and the `_json` alias that only `getAttributes` used.

## 7. Left

| Item | Status |
|---|---|
| Full `session_spec.rb` run | Skipped (user), last full green run at `908428a` (1691 examples, 0 failures); `158230f`, `812f6ca` and `eed662b`..`8caf847` checked with `driver_spec.rb` + filtered session specs, full run on CI |
| Push | Skipped (user), `main` is 6 commits ahead of `origin` (`eed662b`..`8caf847`); `98b586d`..`812f6ca` pushed |
| `Page#set`, `select`, `trigger`, `hover`, `send_keys`, `before_click` | ✅ moved into `Node` in `908428a`, gone from `Page` and `Browser` |
| `Page#accept_confirm`, `dismiss_confirm`, `accept_prompt`, `dismiss_prompt`, `find_modal`, `switch_to_frame` | Now in `Page::Modals`/`Page::Frames` (`e7d0368`), still public since `Driver` calls them on `browser.page` |
| `Browser#active_element`, `Browser#within_window` | Removed without deprecation in `6d3a22c` although public in 0.19, listed under Changed like `835e061`. Revisit only if a deprecation is preferred |
| `url_whitelist`/`url_blacklist` on `Browser` | ⏳ in `Browser::Extensions`, depends on §8 step 3 |
| `Driver#debug`, `Node#drag_by` | Keep (decided), in `Extensions` |
| `Node#attributes` | Deprecated without replacement in `4a912cb`, removed in 0.21 |
| `Node#style`, `Node#set(clear:)` | Out of scope, cleanup only |
| §5 skipped specs (`response_headers`, `#has_field with valid`, `#has_element?`) | Declined |
| `Page` frame overrides of `css`/`at_css`/`evaluate`/... | Keep, intentional for frame switching (`Page::Frames`) |

## 8. Refactoring

Rule: `Driver`, `Node`, `Browser` and `Page` classes hold Capybara's contract and what it needs; deprecated methods go
to `<Class>::Deprecations`, supported extras to `<Class>::Extensions` (`c95956c`).

| # | Step | Status |
|---|---|---|
| 1 | `Node` runs `_cuprite` helpers on its own Ferrum node and owns `set`, `select`, `trigger`, `hover`, `send_keys`, `before_click`, `drag`, `drop`; shared finding in `Finder` | ✅ `908428a` |
| 2 | `Driver` stops calling `Browser` privates with `send`: `find` and `CALLBACK_JS` move into `Driver` | ✅ `db87904` |
| 3 | `options.rb` reopens `Ferrum::Browser::Options` (`url_blacklist`/`url_whitelist`, `raise_on_unhandled_modal`, `mobile`, `scale_factor`, `device_user_agent`, `default_user_agent`, `reset_window_size`). Either move URL filtering and `emulate:`/`Devices` into Ferrum, or keep a small Cuprite settings object handed to `Page` | ⏳ needs a decision: Ferrum or Cuprite |
| 4 | Split `Page` into `Page::Frames` and `Page::Modals`, `Page` keeps Ferrum overrides only | ✅ `e7d0368` |
| 5a | `Driver#open_new_window` and `Browser#attach_page` both build `Page.new(target.client, ...)`; keep one factory | ✅ `eed662b`, `Browser#open_new_window` reuses `attach_page`; `Driver#open_new_window` stays a zero-arity method because Capybara checks its arity |
| 5b | `ClosedPage` null object exists only so `Browser#page` raises after closing a window; replace with a flag or `nil` + raise | ✅ `eed662b`, `@page_closed` flag, `Page#closed?` removed |
| 5c | `Driver#reset!` re-applies URL filters and the modal flag every test only because runtime setters exist | ⏳ depends on step 3 |
| 5d | Mixed DOM paths in `Node` (`driver.execute_script` vs `node.evaluate` vs `command`) | ✅ `908428a` + `048706f` (`shadow_root` via `browser.evaluate` since `Ferrum::Node#evaluate` returns values, `rect` via `node.evaluate`) |
| 5e | Two `MouseEventFailed` paths: `before_click` raises it from a JS error, `Node#command` rebuilds it by regex-parsing the `"Cuprite.MouseEventFailed"` message | ✅ `cf560f1`, the `"Cuprite.MouseEventFailed"` branch was dead (and would raise `ArgumentError`), JS `MouseEventFailed` → `before_click` is the only path |
| 5f | `CUPRITE_MODAL_WAIT` (`Page::Modals`) and `CUPRITE_TRIGGER_CLICK_WAIT` (`Node`) are ENV-read constants, documented in `docs/2-customization.md`; make them driver options or drop them (user-facing) | ⏳ |
| 5g | `Node#set` for `type="color"` still interpolates `value` into JS (`this.setAttribute('value', '#{value}')`), pass it as an argument like the rest | ✅ `21d7c4e` |
| 5h | `Node#before_click` swallows every `Ferrum::JavaScriptError` except `MouseEventFailed` (kept as is in `908428a`); re-raise or document why | ✅ `cf560f1`, re-raised |
| 5i | `Page::Modals#handle_javascript_dialog` calls `client.command` to skip the pending-error wrapper; the reason lost its comment with the removed `ferrum_command` alias | ✅ `54ba999` |
| 5j | `node.evaluate`/`browser.evaluate` with fixed arguments use named keywords instead of `arguments[n]`; `Node#js` and `Finder` stay positional (variadic / optional `within`) | ✅ `8caf847` |

## 9. Next session

State on 2026-10-30: `main` at `8caf847`, 6 commits ahead of `origin` (`eed662b`..`8caf847`, not pushed). Last full
green `session_spec.rb` at `908428a`; everything after was checked with `driver_spec.rb` + filtered session specs only,
so let CI run the full suite once pushed.

Open, needs a decision:

1. **§8 step 3**: `options.rb` reopens `Ferrum::Browser::Options`. Move URL filtering and `emulate:`/`Devices` into
   Ferrum, or keep them in a small Cuprite settings object handed to `Page`. Unblocks 5c and the `url_whitelist`/
   `url_blacklist` methods in `Browser::Extensions` (keep or deprecate).
2. **§8 5f**: `CUPRITE_MODAL_WAIT`/`CUPRITE_TRIGGER_CLICK_WAIT` ENV constants, documented in
   `docs/2-customization.md`: turn into driver options or drop (user-facing).

At 0.21 release: redo the removal from `main` (§6.2), the old `0.21` branch was dropped.

Gotchas learned:

- `Ferrum::Node#evaluate` returns by value, so anything returning DOM nodes (`find`, `shadowRoot`, `parents`) must go
  through `browser.evaluate(..., node: node)`.
- `Driver#open_new_window` must stay a zero-arity method; Capybara checks its arity and passes `kind` otherwise.
- Prefer named `evaluate` keywords over `arguments[n]`; only variadic `Node#js` and `Finder` stay positional.

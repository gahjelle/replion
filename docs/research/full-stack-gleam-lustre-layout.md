# Full-stack Gleam/Lustre project layout

Research for [#2](https://github.com/gahjelle/replion/issues/2). Question: what is the current, idiomatic way to lay out Replion's three Gleam projects (`shared/`, `server/`, `client/`, see ADR 0001), reference `shared/` from both, build the Lustre client and serve it from the server, run the dev loop, and avoid target-specific pitfalls in `shared/`?

Researched 2026-09-17 against these versions (latest on Hex / GitHub on that date):

| Package | Version |
|---|---|
| Gleam compiler | v1.18.1 |
| `gleam_stdlib` | 1.0.5 |
| `lustre` | 5.7.1 |
| `lustre_dev_tools` | 2.3.6 |
| `wisp` | 2.2.2 |
| `mist` | 6.0.3 |
| `gleam_json` | 3.1.0 |
| `gleam_http` | 4.4.0 |
| `gleam_erlang` | 1.3.0 |
| `rsvp` | 2.0.0 |

## Short answer

- **Layout is the one Lustre officially documents.** Three sibling projects created with `gleam new`, which is exactly ADR 0001. Lustre's own guide says a single codebase with several targets isn't the way to go: "the language isn't designed to support a single codebase with different applications for different targets". [L6]
- **`shared/` is a path dependency** in both `client/gleam.toml` and `server/gleam.toml`: `shared = { path = "../shared" }`. Leave `target` unset in `shared/`. Set `target = "javascript"` in `client/`. `server/` can leave it unset, because the default is `erlang`. [GT, L6]
- **Build the client with `lustre_dev_tools`** (a dev-dependency of `client/`): `gleam run -m lustre/dev build`, with `outdir = "../server/priv/static"` set in `[tools.lustre.build]`. The server serves that directory with `wisp.serve_static` and finds it with `wisp.priv_directory("server")`. [L6, TR, W]
- **Dev loop:** run two processes. `server/` runs with `gleam run` on a fixed port. `client/` runs `gleam run -m lustre/dev start` on port 1234, which proxies `/api` to the server and live-reloads. Add `watch = ["../shared/src"]` so edits in `shared/` also trigger a rebuild. [L6, TR, DT]
- **Watch out for these in `shared/`:** keep it pure (only `gleam_stdlib` and `gleam_json`, no FFI and no IO). Remember that `decode.int` and `decode.float` behave differently on Erlang and JavaScript. `gleam_json` needs Erlang/OTP 27 or later on the server. JavaScript Ints lose precision above 2^53.

## 1. Project layout

```
replion/
  justfile
  shared/   gleam.toml: no target; deps: gleam_stdlib, gleam_json
  server/   gleam.toml: (target erlang by default); deps: shared (path), wisp, mist, gleam_http, gleam_erlang, gleam_json
    priv/static/   <- client build output (gitignore it)
  client/   gleam.toml: target = "javascript"; deps: shared (path), lustre, rsvp, gleam_json, gleam_http; dev-deps: lustre_dev_tools
```

Lustre's full-stack guide builds exactly this monorepo: `gleam new client`, `gleam new server`, `gleam new shared`. [L6] The deployment guide assumes the same three directories too. [L7]

### Path dependency

From the `gleam.toml` reference: "Gleam packages on the local file system can be added as dependencies using the `path` field", written as `my_other_project = { path = "../my_other_project" }`. [GT] The Lustre guide adds this to both `client/gleam.toml` and `server/gleam.toml`:

```toml
[dependencies]
shared = { path = "../shared" }
```

and then runs `gleam deps update`. [L6] Each project builds its own copy of `shared` into its own `build/` directory, for its own target.

### `target`

`target` is "the default compilation target to use when compiling or running Gleam code… If this is not provided then the default value `"erlang"` is used." [GT] The guide sets `target = "javascript"` in `client/gleam.toml`. [L6] `lustre_dev_tools` always runs `gleam build --target javascript` for the client either way. [DT `bin/gleam.gleam`] Don't set a target in `shared/`: it is compiled by whichever project depends on it.

## 2. Building the client and serving it from the server

### Build command and config

```sh
cd client
gleam add lustre rsvp gleam_json gleam_http
gleam add --dev lustre_dev_tools
gleam run -m lustre/dev build            # add --minify for a release build
```

You can put the flags in `client/gleam.toml` under `tools.lustre.*` instead of typing them each time ("any flags passed to the command line will always take precedence"): [TR]

```toml
[tools.lustre.build]
outdir = "../server/priv/static"   # default is ./dist

[tools.lustre.dev]
proxy = { from = "/api", to = "http://localhost:<server-port>/api" }
watch = ["../shared/src"]

[tools.lustre.html]
title = "Replion"
stylesheets = [{ href = "/replion.css" }]   # file lives in client/assets/
```

What `build` does (from `lustre/dev.gleam` in 2.3.6): [DT]

1. It runs `gleam build --target javascript`.
2. It writes a small entry module that imports `main` from `build/dev/javascript/client/client.mjs`. Bun bundles it into `<outdir>/client.js`. `lustre_dev_tools` downloads its own Bun unless you set `tools.lustre.bin.bun = "system"`.
3. It builds Tailwind if it finds a `src/client.css` entry. Turn this off with `no_tailwind`.
4. It writes `<outdir>/index.html` unless `no_html = true`. The page has `<div id="app"></div>` and `<script type="module" src="/client.js">`. The script path is **absolute from the site root**.
5. It copies everything in `client/assets/` into `outdir`, keeping the relative paths.

### Two ways to serve it

**Option A: server renders `index.html` (what the Lustre guide does).** Build with `--no-html`. Serve `priv/static` under `/static`, and have the server render an HTML shell that points to `/static/client.js`. [L6] Choose this when the server needs to put data into the page (hydration).

**Option B: use the generated `index.html` (simpler for Replion).** Keep the generated HTML and serve `priv/static` from the root. Send every other GET that isn't an API route to `index.html`. This fits Replion best:

- Replion loads its data through the HTTP API, so it doesn't need hydration.
- The dev server (`lustre/dev start`) creates its own HTML from the same `[tools.lustre.html]` config. The page therefore looks the same in dev and in the built app.

With Option A, you would have to keep the server-rendered page and the dev-server page in sync by hand.

```gleam
// server: sketch (wisp 2.2.2 / mist 6.0.3)
pub fn main() {
  wisp.configure_logger()
  let assert Ok(priv) = wisp.priv_directory("server")
  let static_dir = priv <> "/static"
  let assert Ok(_) =
    handle_request(_, static_dir)
    |> wisp_mist.handler(wisp.random_string(64))
    |> mist.new
    |> mist.port(3000)          // mist defaults: port 4000, interface "localhost"
    |> mist.start
  process.sleep_forever()
}

fn handle_request(req: wisp.Request, static_dir: String) -> wisp.Response {
  use <- wisp.log_request(req)
  use <- wisp.rescue_crashes
  use req <- wisp.handle_head(req)
  case wisp.path_segments(req) {
    ["api", ..rest] -> api.handle(req, rest)
    _ -> {
      use <- wisp.serve_static(req, under: "/", from: static_dir)
      // not a file: fall back to the SPA shell
      wisp.response(200)
      |> wisp.set_body(wisp.File(path: static_dir <> "/index.html", offset: 0, limit: option.None))
      |> response.set_header("content-type", "text/html; charset=utf-8")  // gleam/http/response
    }
  }
}
```

Notes from the sources:

- `wisp.serve_static(req, under:, from:, next)` only handles `GET`. It removes the prefix, strips `..`, sets the MIME type from the file extension, and handles ETag and Range headers. For anything that isn't a regular file (a missing path or a directory), it calls `next`. [W `wisp.gleam` L1416–1460] That is why the fallback works.
- `wisp.priv_directory` is `gleam_erlang`'s `application.priv_directory`. [W L1699] It points to `server/priv` both under `gleam run` and in an `erlang-shipment`. The Lustre deployment guide builds the client into `../server/priv/static` and then runs `gleam export erlang-shipment`. [L7]
- `mist.new` listens on `localhost` over IPv4 by default, on port 4000. `mist.bind` and `mist.port` change these. [M `mist.gleam` L422–490] For Replion, "localhost only" and a fixed port are already the defaults. Only the port number needs picking.
- The fallback above is a sketch. wisp 2.2.2 does have `wisp.response`, `wisp.set_body` and the `File(path:, offset:, limit:)` body variant. [W L61–98] It isn't compiled, though, so the idea to rely on is "serve_static first, then serve `index.html`".

## 3. Dev loop

Run two terminals (or two `just` recipes):

```sh
# terminal 1: backend (restart on server changes; no hot reload)
cd server && gleam run

# terminal 2: frontend dev server with live reload on http://localhost:1234
cd client && gleam run -m lustre/dev start
```

- **Proxy.** The dev server runs on a different port from the backend. Instead of setting up CORS, the guide uses the dev tools' proxy: `proxy = { from = "/api", to = "http://localhost:3000/api" }`. [L6, PX] The source removes the `from` prefix and joins the rest onto the path of `to`. With that config, `/api/sessions` is forwarded to `http://localhost:3000/api/sessions`. [DT `dev/proxy.gleam` L99–110]
  - The TOML reference prose says "`/api/users` would be forwarded to `http://localhost:3000/users`". That doesn't match its own example config, or the source. Trust the source.
  - You can give a list of proxies.
- **Client code always calls relative `/api/...` URLs.** This works in dev (through the proxy) and in the built app (same origin), with no per-environment code.
- **What gets watched.** The dev server always watches `src/` and `assets/`. Add more directories with `watch`. [TR, DT] A change in a watched directory runs `gleam build --target javascript` again and reloads the browser. [DT `dev/watcher.gleam`] `shared/` is outside `client/`, so add `watch = ["../shared/src"]`. Otherwise edits to shared types won't show up until the next client change.
- **`watch_mode`.** Options are `"events"` (the default), `"polling"` and `"none"`. The docs say file-system events don't fire reliably for some editors (helix, nvim). [TR] If reloads get missed, try `"polling"`. Synced folders such as Dropbox are a likely cause too, though that's our guess rather than something the docs say.
- **Static CSS.** Put it in `client/assets/` and list it under `[tools.lustre.html] stylesheets`. The dev server hot-reloads CSS added this way. [AS]
- **Production-like run (`just serve`).** Build the client into `server/priv/static`, then run the server:

  ```sh
  cd client && gleam run -m lustre/dev build --minify
  cd server && gleam run
  ```

  This matches the guide's "Running our app" and "Building for production" sections. [L6]

## 4. Target-specific pitfalls in `shared/`

`shared/` is compiled for Erlang (by `server/`) and for JavaScript (by `client/`), so all of its code has to work on both.

1. **Keep it dependency-light and pure.**
   - Depend only on `gleam_stdlib` and `gleam_json`. Both support both targets: `gleam_json` ships both `gleam_json_ffi.erl` and `gleam_json_ffi.mjs`. [GJ]
   - Anything with IO or runtime-specific code belongs in `server/` or `client/`: `simplifile`, `gleam_erlang`, `gleam_otp`, `wisp`, `lustre`'s DOM parts, `plinth`. This matches the functional-core / imperative-shell rule in map #1.
   - An `@external` function needs an implementation for every target it is compiled for. Otherwise "the compiler will return an error". [TX] Avoid FFI in `shared/` entirely.
2. **Ints and floats decode differently.** `decode.int` "will not coerce float values into int values, so on platforms with distinct runtime int and float types (Erlang, not JavaScript) it will fail, even if the float is a whole number (e.g. 1.0)". `decode.float` fails for ints on Erlang, and the docs call out JSON decoding as a place this happens. [SL `decode.gleam` L694–735]
   - This matters for Adapter decoders, which run on the server over Harness JSON (token counts, timestamps, durations). Use `decode.one_of` or a decoder that accepts both when a field could be either.
   - The problem only shows up on Erlang. Tests of the shared codecs should therefore run on the Erlang target, which is the default for `gleam test` in `shared/`. Ideally they also run with `--target javascript`.
3. **Large integers.** JavaScript `Int` is a JS number and is exact only up to 2^53. That's fine for millisecond timestamps and token counts. Nanosecond timestamps or large IDs should stay strings in the shared JSON.
4. **OTP version.** `gleam_json` 3.x uses the `json` module that Erlang/OTP 27 added, and fails at runtime with `erlang_otp_27_required` on older OTP. [GJ `gleam_json_ffi.erl`] The server machine needs OTP ≥ 27.
5. **Encoders and decoders.** The Lustre guide keeps the type, `*_decoder` and `*_to_json` side by side in the shared module, "largely auto-generated by using the respective code actions" of the Gleam language server. [L6] Do the same for Session, Event and Session Entry.
6. **Name modules under the package name**, as in `shared/src/shared/session.gleam` → `import shared/session`. [L6] A more specific package name (e.g. `replion_shared`) is fine too, but the guide uses `shared`.

## Sources

- **[L6]** Lustre guide "06 Full stack applications", lustre main, `pages/guide/06-full-stack-applications.md`: https://github.com/lustre-labs/lustre/blob/main/pages/guide/06-full-stack-applications.md (also at https://hexdocs.pm/lustre/guide/06-full-stack-applications.html; retrieved via Context7 `/lustre-labs/lustre` and raw GitHub)
- **[L7]** Lustre guide "07 Full-stack deployments": https://github.com/lustre-labs/lustre/blob/main/pages/guide/07-full-stack-deployments.md
- **[TR]** lustre_dev_tools TOML reference: https://github.com/lustre-labs/dev-tools/blob/main/pages/toml-reference.md (https://hexdocs.pm/lustre_dev_tools/toml-reference.html)
- **[PX]** lustre_dev_tools "API Proxying": https://github.com/lustre-labs/dev-tools/blob/main/pages/proxying.md
- **[AS]** lustre_dev_tools "Handling assets": https://github.com/lustre-labs/dev-tools/blob/main/pages/assets.md
- **[DT]** lustre_dev_tools 2.3.6 source from the Hex tarball (`src/lustre/dev.gleam`, `src/lustre_dev_tools/bin/gleam.gleam`, `dev/proxy.gleam`, `dev/watcher.gleam`, `build/html.gleam`): https://hex.pm/packages/lustre_dev_tools/2.3.6
- **[W]** wisp 2.2.2 source, `src/wisp.gleam` (`serve_static`, `priv_directory`): https://hex.pm/packages/wisp/2.2.2 / https://hexdocs.pm/wisp/
- **[M]** mist 6.0.3 source, `src/mist.gleam` (`new`, `port`, `bind`, `start`): https://hex.pm/packages/mist/6.0.3
- **[GT]** Gleam `gleam.toml` reference (path dependencies, `target`): https://gleam.run/documentation/gleam-toml-reference/
- **[TX]** Gleam language tour, "Multi-target externals": https://tour.gleam.run/advanced-features/multi-target-externals/
- **[SL]** gleam_stdlib 1.0.5 source, `src/gleam/dynamic/decode.gleam`: https://hex.pm/packages/gleam_stdlib/1.0.5
- **[GJ]** gleam_json 3.1.0 source and README: https://hex.pm/packages/gleam_json/3.1.0
- Versions: Hex API (`https://hex.pm/api/packages/<name>`) and GitHub releases for gleam-lang/gleam (v1.18.1, 2026-08-01).

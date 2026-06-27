# Sub-Store OpenWrt Integration Design

Date: 2026-06-27
Repository: `openwrt-daede`

## Purpose

Add Sub-Store support to the OpenWrt feed in a way that fits the existing
`openwrt-daede` package style without making `luci-app-daede` heavy or tightly
coupled to a Node.js service.

The target user can install an optional Sub-Store service package, manage it
from LuCI, and use its subscription output as an input to the existing
dae/daed import and conversion flow.

## Current Context

The local repository already contains:

- `daed/`: OpenWrt package for the dae-wing based `daed` service.
- `luci-app-daede/`: LuCI application for managing dae/daed, including
  configuration, dashboard embedding, subscription conversion, updates, logs,
  and daed subscription auto-update scripts.
- `luci-app-daede/htdocs/luci-static/resources/view/daede/daed-web.js`: an
  iframe-based pattern for exposing a backend web UI from LuCI.
- `luci-app-daede/htdocs/luci-static/resources/view/daede/converter.js` and
  `clash-converter.js`: existing browser-side import and conversion paths for
  Clash-style YAML and share links.
- `luci-app-daede/root/usr/share/luci-app-daede/daed-sub-update.sh`: an
  existing shell-managed path for updating daed subscriptions through GraphQL.

Sub-Store upstream is a Node.js-backed subscription manager rather than a pure
LuCI frontend. It should be treated as a service with a LuCI control surface,
not as code to inline into the existing LuCI JavaScript.

## Building

Build two optional packages in this feed:

1. `sub-store`: an OpenWrt service package that installs and runs the upstream
   Sub-Store backend bundle.
2. `luci-app-sub-store`: a LuCI package that manages the service and exposes
   the Sub-Store web UI, with a small bridge into `luci-app-daede`.

The packages may live in this repository next to `daed/` and `luci-app-daede/`.
They should not be mandatory dependencies of `luci-app-daede`.

## Not Building

This design does not:

- Rewrite Sub-Store as native LuCI JavaScript.
- Vendor Sub-Store into the `luci-app-daede` package.
- Make Node.js a dependency of `luci-app-daede`.
- Replace the current `luci-app-daede` converter.
- Require 192.168.3.252 during initial implementation; device validation runs
  during the device-check phase when the test device is reachable again.
- Build a full bidirectional sync system between daed and Sub-Store.

## Chosen Approach

Use a sidecar service model.

`sub-store` owns the runtime, data directory, init script, UCI config, and
network listener. `luci-app-sub-store` owns service controls, settings, logs,
and the browser entry point. `luci-app-daede` keeps its current import and
conversion responsibilities, with only a narrow optional integration that can
consume Sub-Store output.

This keeps the current daede packages installable on small systems while still
letting users opt into the larger Sub-Store capability.

## Rejected Alternatives

### Inline Sub-Store Into `luci-app-daede`

Rejected because Sub-Store is service-oriented and depends on a Node runtime.
Inlining it would make the lightweight LuCI app pull a backend runtime and
would blur ownership between dae/daed management and subscription management.

### Pure External Documentation

Rejected because it leaves users without OpenWrt-native service management.
It also prevents a clean LuCI workflow for opening Sub-Store and feeding its
output into daede.

## Package Layout

Add:

```text
sub-store/
  Makefile
  files/
    sub-store.config
    sub-store.init

luci-app-sub-store/
  Makefile
  root/etc/config/sub-store
  root/usr/share/luci/menu.d/luci-app-sub-store.json
  root/usr/share/rpcd/acl.d/luci-app-sub-store.json
  root/usr/share/luci-app-sub-store/
    pkg-info.sh
  htdocs/luci-static/resources/view/sub-store/
    config.js
    dashboard.js
    log.js
    status.js
```

If the implementation proves small enough, `status.js`, `config.js`, and
`dashboard.js` may be a single `sub-store.js` view. The package boundary stays
the same.

## `sub-store` Package

The `sub-store` package installs:

- the upstream backend bundle at `/usr/share/sub-store/sub-store.bundle.js`;
- a UCI default config at `/etc/config/sub-store`;
- a procd init script at `/etc/init.d/sub-store`;
- a writable data directory, preferably `/etc/sub-store`, because OpenWrt users
  expect service configuration and persistent subscription data to survive
  reboot and package upgrades.

Default UCI shape:

```text
config sub_store 'config'
  option enabled '0'
  option listen_addr '127.0.0.1'
  option port '3001'
  option data_dir '/etc/sub-store'
  option log_file '/var/log/sub-store.log'
```

The service listens on loopback by default. LuCI can show a warning before the
user binds it to LAN.

Runtime dependencies:

- `node` or the OpenWrt package name available in the target feed;
- `ca-bundle`, because subscription fetching commonly needs HTTPS.

The implementation must verify the actual OpenWrt Node package name and the
minimum compatible Node version during build work. If the feed cannot provide a
compatible Node runtime, the package should be gated behind a config option or
documented as unavailable for that target.

## Service Behavior

The procd init script should:

- read `/etc/config/sub-store`;
- exit cleanly when disabled;
- create the data directory before start;
- run the backend bundle with `node`;
- export Sub-Store environment variables from UCI values;
- redirect stdout/stderr to the configured log file;
- use procd respawn with conservative limits;
- stop the service without deleting user data.

The service must not listen on `0.0.0.0` by default.

## LuCI Package

`luci-app-sub-store` should provide:

- status display: installed, running, PID, listen URL, data directory;
- settings: enabled, listen address, port, data directory, log file;
- commands: start, stop, restart, enable, disable;
- dashboard page: iframe when the browser can load it safely, and "open in new
  tab" fallback when mixed-content or browser restrictions apply;
- log page: read the configured log file with the same style as daede logs.

The UI should follow the existing `luci-app-daede` patterns:

- use LuCI JS views;
- use `rpcd` ACLs for exact command/file access;
- use UCI for persistent settings;
- use service status from ubus where possible;
- keep pages utilitarian and compact.

## Daede Integration

The first integration should be deliberately narrow:

1. Add a LuCI entry or action that helps users open Sub-Store from the daede
   subscription converter area when `sub-store` is installed.
2. Let users paste or fetch Sub-Store output as Clash/Mihomo YAML or share
   links into the existing converter.
3. Avoid writing directly into Sub-Store's internal storage from daede.

This direction reuses the current daede conversion surface and avoids coupling
to Sub-Store private data formats.

The data flow is:

```text
Sub-Store source subscriptions
  -> Sub-Store processing/output endpoint
  -> Clash/Mihomo YAML or share links
  -> luci-app-daede converter
  -> dae config or daed subscription import
```

## Security

Default security posture:

- Sub-Store binds to `127.0.0.1`.
- LuCI dashboard embedding uses the router host and configured port.
- Binding to LAN is allowed only after explicit user configuration.
- `rpcd` ACL grants only the exact files and init commands needed by
  `luci-app-sub-store`.
- Logs are readable through LuCI, but subscription data files are not exposed
  directly through CGI.
- No default password or token is generated in the package unless upstream
  requires one. If upstream supports an auth secret, generate it in
  `uci-defaults` and store it under `/etc/config/sub-store`.

## Upgrade And Release Strategy

The `sub-store` Makefile should fetch a pinned upstream release artifact with a
fixed hash. Do not fetch "latest" at build time.

Package versioning should make the upstream Sub-Store version obvious, for
example:

```text
PKG_NAME:=sub-store
PKG_VERSION:=<upstream-version>
PKG_RELEASE:=1
```

When updating Sub-Store:

- update the artifact URL;
- update `PKG_HASH`;
- record the upstream version in the commit message;
- run the package build;
- test service start on 252 or another OpenWrt device.

## Error Handling

Expected failure cases and behavior:

- Node missing or incompatible: service start fails with a clear log line;
  LuCI shows "runtime missing or incompatible".
- Bundle missing: package build or install should fail; LuCI shows "not
  installed" if the package is absent.
- Port conflict: service fails to start; LuCI log page shows the bind error.
- Invalid listen address or port: LuCI validation prevents save where possible;
  init script refuses unsafe or malformed values.
- Sub-Store unreachable from iframe: LuCI shows an open-in-new-tab fallback.
- Sub-Store output invalid for daede converter: existing converter validation
  reports parse/import errors without changing dae/daed config.

## Testing

Static checks:

- `make package/sub-store/{clean,compile} V=s` in an OpenWrt buildroot.
- `make package/luci-app-sub-store/{clean,compile} V=s`.
- `sh -n` for shell scripts.
- JSON validation for LuCI menu and rpcd ACL files.

Device checks when 192.168.3.252 is reachable:

- install `sub-store` and `luci-app-sub-store`;
- verify `/etc/init.d/sub-store start`, `stop`, `restart`, `enable`, and
  `disable`;
- verify service binds to loopback by default;
- verify LuCI status, config save, dashboard fallback, and log page;
- verify LAN binding only after explicit config change;
- generate or copy a Sub-Store output URL and import it through the existing
  daede converter;
- restart router or service and confirm Sub-Store data persists.

Regression checks for existing daede:

- existing daed dashboard page still loads;
- existing subscription converter still accepts Clash YAML;
- existing daed subscription update scripts still run;
- installing `luci-app-daede` alone does not install Node or Sub-Store.

## Rollback

Rollback is simple because Sub-Store is isolated:

- uninstall `luci-app-sub-store` to remove the LuCI control surface;
- stop and disable `/etc/init.d/sub-store`;
- uninstall `sub-store`;
- leave `/etc/sub-store` in place by default so user data is not lost;
- remove `/etc/sub-store` only on explicit user request.

No migration of existing dae/daed config is required.

## Fragile Assumption

This design assumes the target OpenWrt feeds can provide a Node runtime
compatible with the upstream Sub-Store release bundle. If that assumption fails,
the sidecar package is still the right boundary, but the implementation must
either pin an older compatible Sub-Store release, provide an architecture-gated
package, or stop before merging runtime support.

## Approval State

Approved direction: sidecar `sub-store` service package plus optional
`luci-app-sub-store` control package, with a narrow bridge back into
`luci-app-daede` through existing converter flows.

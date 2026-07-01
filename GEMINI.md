# GEMINI.md

Agent context for working on **lima-net** — a single-file Textual TUI, shipped
as a Lima **CLI plugin**, that maps Lima VMs to their networks (vSwitches),
NICs, and IPs.

## What this project is

One executable file, `limactl-net` (no extension — that's the Lima plugin
naming convention; `limactl net` runs `limactl-net` from PATH). No package, no
build step. Self-bootstrapping via a `#!/usr/bin/env -S uv run --script`
shebang with PEP 723 inline deps (`textual`, `pyyaml`). Target environment is a
no-sudo Apple Silicon Mac; do not introduce steps that need root or a system
package manager.

**Plugin contract (do not break):**
- Line 1 MUST be a shebang (`#!…`). Lima only extracts the description from
  files that start with `#!`.
- Keep the `# <limactl-desc>…</limactl-desc>` comment near the top — it's what
  `limactl --help` and `limactl info` display.
- The file stays named `limactl-net` and executable (`chmod +x`).
- argparse `prog` is `limactl net` so `--help` reads correctly under Lima.

## Layout of the single file (top to bottom)

- **Data model** — `Nic`, `VSwitch` dataclasses.
- **Pure parsers** (no subprocess, unit-testable): `parse_instances`,
  `parse_guest_ifaces` (returns an ordered **list** of real NICs),
  `parse_networks_yaml`, `parse_network_table`, `merge_network_list`,
  `_instance_networks`, `_match_subnet_label`, `_label_for`, `_cidr`.
- **Backends** — `RealBackend` (shells out to `limactl`, with an on-disk
  `lima.yaml` fallback in `list_instances`) and `DemoBackend` (canned data).
  They share an implicit interface: `list_instances()`, `guest_addr_json(name)`,
  `network_defs()`.
- **`build_topology(backend, keep_virtual)`** — the core per-VM join.
- **Rendering** — `render_by_switch`, `render_by_vm` (Rich renderables).
- **`LimaNetApp`** — Textual app; data collection runs in a `@work(thread=True)`
  worker and updates the UI via `call_from_thread`.
- **`main()`** — argparse (incl. `--dump` diagnostic) + backend selection.

## Run / smoke-test

There is no pytest suite; the demo path is the smoke test. Run these after a
change before declaring done (use `python limactl-net` in a sandbox without uv,
or `./limactl-net` to also exercise the uv shebang):

```sh
python limactl-net --print --demo            # vSwitch view
python limactl-net --print --demo --by-vm    # VM view
python limactl-net --print --demo --all-ifaces   # CNI NOT filtered (contrast)
python limactl-net --dump --demo             # diagnostic: config network[] vs guest NICs
./limactl-net --print --demo                 # exercises the uv-shebang plugin path
```

Headless TUI check (boots app, runs worker, exercises the `t` toggle):

```sh
python - <<'PY'
import sys, importlib.util, importlib.machinery, asyncio
spec = importlib.util.spec_from_loader("limactl_net",
        importlib.machinery.SourceFileLoader("limactl_net", "limactl-net"))
m = importlib.util.module_from_spec(spec)
sys.modules["limactl_net"] = m          # needed so @dataclass can resolve the module
spec.loader.exec_module(m)
async def t():
    app = m.LimaNetApp(m.DemoBackend())
    async with app.run_test() as pilot:
        await app.workers.wait_for_complete(); await pilot.pause(0.1)
        app.query_one('#diagram').render()
        await pilot.press('t')
        await app.workers.wait_for_complete(); await pilot.pause(0.1)
        app.query_one('#diagram').render()
    print('ok', app.by_vm)
asyncio.run(t())
PY
```

The filename has a hyphen and no `.py`, so load it with a `SourceFileLoader` and
register it in `sys.modules` (as above) rather than a plain `import`.

## Invariants — do not break these

- **The NIC→network match is a 4-step pipeline, in this order:** (1) MAC against
  the VM's configured `network[]`, (2) **interface name** against `network[]`,
  (3) IP subnet against known CIDRs, (4) else default usermode NAT. Step 2 is
  load-bearing: it places an interface that has *no IPv4 yet* (e.g. `lima1` on
  an egress net), which neither MAC-less config nor subnet matching could do.
  Do not reduce this back to "MAC only" — that regresses the kube-ovn case.
- **Read the singular key.** `limactl list --json` emits the per-instance array
  under `network` (singular), not `networks`. `_instance_networks()` handles
  `network` → `networks` → `config.networks` → on-disk `lima.yaml`. Reading only
  `networks` makes every NIC fall through to the NAT bucket.
- **Subnet matching is a last resort and can be ambiguous.** Overlapping CIDRs
  exist in real setups (`shared`/`user-v2-egress` share `192.168.105.0/24`;
  `host`/`user-v2-node` share `192.168.106.0/24`). Only the config (MAC/iface)
  disambiguates. Never promote subnet matching above config matching.
- **CNI filtering is structural.** A real NIC has `link_type == ether`, is not
  `lo`, and has **no `linkinfo.info_kind`**. Keep it that way; do not switch to
  a name-based blocklist (`veth*`, `cni0`, `ovn0`, …) — new CNIs/overlays would
  leak through. `geneve`/`vxlan`/`bridge`/`veth`/`dummy` all carry an info_kind.
- **Every defined network shows**, even with zero VMs (empty panel). That's a
  feature, not clutter.
- **Usermode slirp NAT is per-VM**, not a shared bus — render it as a NAT panel
  (`shared=False`), and keep it distinct from any *named* `user-v2` network.
- The two backends must stay interface-compatible so `DemoBackend` keeps the
  TUI testable without Lima. `DemoBackend.INSTANCES` must use the singular
  `network` key (mirrors reality and guards against the bug regressing).

## Known upstream bugs (already worked around — keep the workarounds)

Two separate `limactl` quirks, both with no upstream tracking issue at the time
of writing:

1. **`network list --json` drops the name** (it's the map key, dropped by
   `json.Marshal` in `cmd/limactl/network.go`). The table still has `NAME`.
   `RealBackend.network_defs()` / `merge_network_list` zip the table names onto
   the JSON rows positionally (same sorted slice → safe). Without this, the
   `user-v2*` variants are indistinguishable. `networks.yaml` is the fallback.

2. **`list --json` uses the singular key `network`** for the per-instance array
   (the `Instance` struct in `pkg/limatype/lima_instance.go` tags it
   `json:"network"`), while `lima.yaml` uses plural `networks`. This is the bug
   that put every NIC in the NAT bucket. Handled in `_instance_networks()`.

If either is fixed upstream, the corresponding workaround can be simplified, but
keep them until then.

## Conventions

- Keep it a single file; resist splitting into a package.
- Pure parsers take/return plain strings + dicts so they can be tested without
  subprocess or a real Lima.
- Rendering uses Rich primitives (`Panel`, `Table.grid`, `Text`) inside a Static
  widget; no business logic in the render functions.
- New external data goes through a `parse_*` function first, then into the
  dataclasses — never parse inside `build_topology` or the renderers.

## Likely next tasks

- IPv6 column (currently IPv4 only; filter is in `parse_guest_ifaces` /
  `_ip_text`).
- Per-VM grouping for the NAT panel.
- A `cheat/cheatsheets`-style entry generated from `--print` output.

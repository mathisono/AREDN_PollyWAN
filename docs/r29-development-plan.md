# PollyWAN r29 Development Plan

## Release target

PollyWAN r29 is the final planned package release targeting AREDN `4.26.7.0`.
It remains a development package until live validation is complete.

- Stable source tag: `4.26.7.0`
- Stable source commit: `93ad9ea94fb2c0edd829c513305ffbaa90c07858`
- Test node: `KJ6DZB-WSB-hub5`
- Board and target: `mikrotik,hap-ac2`, `ipq40xx/mikrotik`
- Architecture: `arm_cortex-a7_neon-vfpv4`
- Observed stable kernel: `6.12.94`

Development builds and live validation must use this stable ABI. Current AREDN
`main` is a compatibility-audit target, not the r29 build target. PollyWAN r30
will start the nightly-targeted development line.

## Forward-compatibility baseline

The authoritative upstream repository is `https://github.com/aredn/aredn.git`.
Do not use the `mathisono/aredn` fork's `origin/main` as evidence of current
AREDN nightly state.

Initial nightly audit commit:

```text
eb2a89c10adf2ae22165ae2b5589f2d44531cbe6
```

Refresh and record this commit whenever the compatibility audit is repeated.
Do not silently change the audit baseline during an r29 test cycle.

## Compatibility gates

| Area | Stable 4.26.7.0 | Audited nightly behavior | r29 requirement |
| --- | --- | --- | --- |
| Local selected route | PollyWAN tables 26 and 27 | AREDN retains table 27 and adds more routing controls | Preserve PollyWAN's stable behavior; detect incompatible ownership before changing routes |
| Mesh-exported default | PollyWAN exclusively qualifies and manages table 28 | AREDN `wan_monitor.uc` also adds and removes table-28 routes | Adapt the native monitor to PollyWAN's qualification and damping contract; it must be the sole table-28 owner on nightly |
| Remote Mesh WAN | Table 22 | Table 23 is inserted ahead of table 22 for a local default learned over DtD | Treat table 23 as a distinct future input; do not silently classify it as table 22 |
| Internet health | Source-bound HTTPS with primary and optional secondary endpoint | AREDN monitor uses source-interface ping targets | Do not weaken PollyWAN qualification to gateway reachability or ping-only monitoring |
| Babel lifecycle | PollyWAN dampens restarts and restarts only when required | Network option changes can explicitly request Babel and WAN-monitor restarts | Preserve idempotence and identify external restarts in telemetry/tests |
| RF topology | Stable per-release network generation | PR #2816 moves RF modes onto shared `br-wifi` | Never classify `br-wifi` or `br-fast` as an Internet WAN candidate |
| Firewall | Stable zone layout | PR #2817 assigns `wifi` and `fast` to the `wifi` zone | Never move logical networks `wifi` or `fast` into the WAN zone |
| Port management | Stable advanced-port behavior | Nightly changes port migration and phantom-change handling | Keep role changes isolated from XLinks and retain rollback/Cancel semantics |

## r29 work sequence

1. Preserve and review all post-r28 uncommitted work already present in both
   PollyWAN trees.
2. Complete the stable-to-nightly compatibility matrix for network generation,
   firewall, Babel, LQM, ports, UCode, OpenWrt, kernel, and APK packaging.
3. On stable `4.26.7.0`, retain the r28 route contract and all r28 GUI,
   telemetry, damping, XLink, and Cancel behavior.
4. Keep the preserved stable release at `0.1.0-r29`; corrective work uses APK
   version `0.1.0-r29.5-r1` for product release R29.5 while it is reviewed,
   built, and validated on stable firmware.
5. Run standalone verification, integration synchronization/check, and
   integration verification. Record root-only verification as pending unless it
   actually succeeds.
6. Refresh or reconstruct the stable hAP ac2 build tree before compiling. Never
   use a nightly build tree for the stable r29 APK.
7. Back up hub5 and validate the development APK on its stable firmware.
8. After r29 validation, freeze a fresh authoritative nightly commit and begin
   r30. Do not mix the nightly source adaptation into the stable r29 APK.

## r29.5 release scope

Development for these corrective changes continues on `release/r29.5`. The
explicit Ethernet and Selection Policy apply controls are R29.5 changes; they
are not part of the published R29 APK.

### R29.5 pre-build checkpoint

- explicit Save/apply controls and visible `Done` warnings are implemented;
- ordered local/Remote Mesh WAN routing and recovery damping are implemented;
- Remote Mesh WAN exit-node telemetry is present in the dashboard and both the
  Connection Speed Test card and dialog;
- the initialization commit-order bug is fixed and required defaults are
  verified after the persistent commit;
- runtime telemetry and the dashboard identify product release `R29.5`; the APK
  uses product version `0.1.0-r29.5` plus integer APK package revision `r1`;
- standalone verification, stable-tree synchronization/build, and independent
  live validation on both hAP ac2 nodes remain release gates.

### Ordered Route Policy Setup

R29.5 replaces the Manual/Automatic and speed-ranked policy with one ordered
route list. WAN 1, WAN 2, Android USB tether, and Remote Mesh WAN each appear
exactly once in the preference order and have independent enable switches.
Failover moves downward immediately after the failure threshold; recovery moves
upward only after the configured success count and hold-down. Speed is a local
eligibility floor and never reorders the list.

The operator-facing page is renamed `Route Policy Setup`, has no Advanced
disclosure, and removes expected HTTP response-code fields, result lifetime,
automatic test scheduling, manual selection mode, and the return-to-preferred
toggle. It retains only settings useful to the routing decision: enabled
routes, route order, clear local and mesh-sharing data-rate floors, health URLs
and timing, recovery damping, and local-route mesh export.

Remote Mesh WAN status in R29.5 resolves the originator ID of the installed
Babel table-22 default to the originating node's primary address and AREDN node
name. The dashboard shows that exit node, origin address, next hop, interface,
and route availability. Connection Speed Test includes a read-only Remote Mesh
WAN row with the same exit status; it does not claim that a local speed test can
measure the remote gateway's own Internet link.

### Preserve and verify initialization

Live validation on `KP4DJT-HAP-AC2-VAN` found the R29 APK registered as
installed while the expected `aredn.multiwan` UCI section was absent and the
service was inactive. Running the packaged initializer again created the
section only after its conflicting runtime commit was removed, with
`enabled=0`.

The failure was reproduced under shell tracing. The initializer stages defaults
through `uci -c /etc/config.mesh`, but `cleanup_old_proxy_state()` then runs
`uci commit aredn` against the runtime configuration. On AREDN that commit
synchronizes runtime state back over the mesh configuration and erases the
staged `multiwan` defaults before the final mesh commit. The script nevertheless
returns zero. The R29 GUI then cannot save Enable because its `uciMesh` handler
assumes the missing `aredn.multiwan` section already exists and does not report
set or commit failures.

The package currently stores its section inside `/etc/config.mesh/aredn`.
AREDN can regenerate that file after installation, so a later configuration
save may discard the package-owned section. R29.5 must:

- make the post-install initializer fail when any required UCI write or commit
  fails instead of returning success unconditionally;
- remove or reorder the runtime `uci commit aredn` in
  `cleanup_old_proxy_state()` so it cannot overwrite staged mesh defaults, and
  add a regression test for this exact commit-order failure;
- verify after installation that `aredn.multiwan` exists and contains all
  required defaults;
- make every GUI save handler create or repair the `multiwan` section before
  setting options, verify the mesh commit, and show a visible error when a set,
  commit, or apply operation fails;
- keep the Ethernet Port Roles Save and Apply-with-rollback controls directly
  beneath the port-role table instead of below the unrelated XLinks editor and
  help content, so applying a role change is visible at the point of editing;
- keep the PollyWAN Selection Policy apply control directly beneath its form
  and label it `Save and apply policy`, matching the handler's existing commit
  and controller-restart behavior;
- state beside both apply controls that the dialog's `Done` button only closes
  the window and never saves or applies configuration;
- verify the saved persistent value is synchronized to runtime before starting
  or restarting PollyWAN services;
- preserve the section across AREDN configuration regeneration, or move
  package-owned state to a dedicated persistent UCI config with a compatible
  migration;
- add an idempotent repair path for an installed package whose section is
  missing;
- test first install, reinstall, firmware/config regeneration, reboot, and
  uninstall on both hAP ac2 validation nodes;
- keep repair and migration disabled by default and prove they do not change
  radios, ports, GPS, WAN routes, time/location, or USB power.

## Native WAN manager direction

AREDN stable `4.26.7.0` does not contain the new native `wan_monitor.uc`, so the
stable r29 package continues to use `wan-sla`. The native-manager work begins
with r30 and must not be copied into the stable r29 build tree.

PollyWAN r30 will target a recorded AREDN nightly baseline. Its native manager
must become the single table-28 owner and adopt PollyWAN's source-bound HTTPS
qualification, immediate withdrawal, export recovery count, hold-down, route
idempotence, and cached telemetry contract. The r30 handoff must inspect active
route managers and route ownership and must never leave both `wan-sla` and the
native WAN manager able to mutate table 28.

The nightly manager must not merely reproduce PollyWAN's implementation
language-for-language. It must reproduce the behavioral contract:

- distinguish interface, route, gateway, upstream, selection, and export state;
- qualify Internet access with a source-bound HTTPS request;
- use an optional secondary endpoint only after primary failure;
- withdraw table 28 on the first confirmed upstream failure;
- keep local selection hysteresis separate from mesh-export recovery;
- restore export only after the configured recovery count and hold-down;
- change table 28 only when the desired route differs from current state;
- avoid unnecessary Babel restarts;
- publish internal status and cache-only public telemetry;
- coordinate tables 22 and 23 without treating them as interchangeable;
- remain inert when PollyWAN is disabled or ownership has not been granted.

## Stop conditions

Stop implementation or deployment when any of these is true:

- another active service can add or remove PollyWAN-owned table-28 routes;
- table 23 is present and its ownership or policy precedence is unknown;
- generated network or firewall configuration would move RF paths into a WAN
  role;
- a route change can strand management access without a tested rollback;
- the build ABI differs from the stable test-node architecture or kernel ABI;
- verification or integration synchronization fails.

Root-only verification remains pending until it is actually run successfully.

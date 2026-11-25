# UniApp MVP Requirements — Host Foundation

> Version: v2.2 (2026-11-25) — MVP converged to **host foundation + one sample sub-app**, starting Android-only.
> **Overall goal and scope**: Build the minimal host foundation and have the single `:feature-sample` complete registration, home-screen display, page entry, shared-service invocation, and its own data persistence. The sample is the acceptance fixture and the template to copy from; it carries no real product business. **The second sub-app is not required, and no extension capability (LLM, generic database factory, backup, etc.) is prebuilt**; other sub-apps get their own requirements when proposed.
> Platform strategy: **Android-only, no KMP structure** (§7); `:shared` keeps only one portability discipline.
> Companion document: [architecture-decisions.md](architecture-decisions.md)

---

## 1. Positioning

**One sentence**: a single-APK "sub-app host" for personal use — a microkernel platform inside one app.

- **Design direction**: one installable package holding every small tool I build from now on; tools stay fully isolated from each other, and deleting any one leaves the foundation intact.
- Personal use, **not published to any store** (this premise reserves the freedom for future sub-apps to use platform-privileged capabilities; the foundation itself does not presume where it will be used).
- Single user, Android 11 / API 30+.

**The product's inherent duality**: the foundation has two classes of "users" — the end user (opening and using it daily) and the sub-app developer (also me). This document serves both sides: **the usage experience** and **the integration experience**.

## 2. Core scenarios

1. **Use**: open the app, see the aggregated home screen of registered sub-apps, tap any one to use it; unwanted entries can be turned off in settings.
2. **Mount**: developing a new sub-app = create a Gradle module → provide an entry and a public integration API → register the entry, DI, and routes at the host's central assembly point → it appears on the home screen. **Module declarations, dependencies, and the assembly list may change; the generic home screen, navigation mechanism, and shared services never need to.**
3. **Unmount**: deleting a sub-app = remove the module + unregister the entry. The build passes, the foundation and the remaining sub-apps are intact, no dangling references.

## 3. MVP scope (P0 = the minimal set needed to accept the foundation)

### FR-1 Host shell and aggregated home screen
Single Activity (Compose + NavHost). The home screen aggregates enabled sub-app entries (icon / name / status); a tap routes through the registry to the sub-app entry. With no sub-app registered, show an empty state with guidance.
**Acceptance**: register only the sample entry → one entry appears on the home screen → tapping enters the sample; after disabling it, the home screen shows the empty state.

### FR-2 Sub-app registration mechanism
**A compile-time Gradle module is the sub-app**, with a code-level registry (no declarative manifest file). Registry entries carry UI-free metadata; route mounting and DI come from the sub-app's public integration API and are invoked at the host assembly point:

```
SubAppEntry (minimal MVP field set):
  id / display name / entry route
  permission requirements (for the home screen's permission status display and guidance)
  shared capabilities consumed (for host initialization and troubleshooting)
```

**Acceptance**: the sample integrates only through module dependency and the central assembly list; no reference to the sample's internals appears in the generic home screen, the navigation mechanism, or the shared services.

### FR-3 Shared service set (minimal viable)
Contains only services with a **real consumer** during the MVP — the consumer is the FR-4 sample sub-app; nothing "possibly useful later" is prebuilt:

| Service | Purpose |
|---|---|
| DI container (Koin) | Service registry; the single access point for all shared services |
| Typed event bus (SharedFlow) | sub-app ↔ host and sub-app ↔ sub-app notifications, zero mutual holding |
| Encrypted credential store (Keystore) | The standard home for all future key-like data |
| KV config store (DataStore) | The standard home for non-sensitive configuration |

**Acceptance**: each of the four services is genuinely exercised once inside the sample sub-app (store/retrieve, send/receive, encrypt, read).

### FR-4 The single sample sub-app
This round implements only `:feature-sample`, as the acceptance fixture for the foundation and the template to copy from. It contains a minimal Compose page that shows the results of shared-service invocations and provides one simple input to verify saving and reading of its own configuration; no concrete product business is introduced.

- **UI and page logic**: live inside `:feature-sample`; the host only provides entry aggregation and navigation.
- **Own data**: config keys use the sample namespace; entities, repositories, storage code, and migrations (if any) belong to the sample's own module. The simplest possible KV persistence suffices for the MVP; Room is not required.
- **Foundation data**: enable/disable state, global config, and credentials are managed by the corresponding `:shared` services; the sample does not reimplement these common services.
- **Storage extensions**: no Room build factory or unified backup hook is prebuilt; when a future sub-app genuinely needs a database, the independent-database and storage-foundation decision is made then.

**Acceptance**: the sample is reachable from the home screen; shared-service invocations show visible results; editing its own config survives an app restart; its own storage implementation never enters the host.

### FR-5 Foundation settings page
Enable/disable switches for registered sub-apps (off = no route mounted, no entry shown, entry record retained), a permission status overview with jump links, and the credential management entry.
**Acceptance**: turn the sample off → its home-screen entry disappears and the route is unreachable; turn it back on and everything restores.
Note: a sub-app's **own** settings (some tool's preference items) are the sub-app's own page and route (`xxx/settings`) inside its own module — not part of the foundation settings page. The foundation settings page manages foundation assets only (enable/disable, permissions, credentials).

### FR-6 Sample integration and removal acceptance (a cross-cutting standard, not a new feature)
- **Mount**: verify with the sample only that "module integration → registration → home-screen visibility → page entry" works; no second sub-app is required, and no "half-day integration" time metric is set;
- **Unmount**: temporarily remove the sample module's dependency and registration; verify the host still builds, runs, and shows the empty state with no dangling references; restore the sample after acceptance — the final delivery keeps exactly this one sub-app.

## 4. P1 — deferred items (outside the MVP; boundary, not backlog)

Nothing in this table is an MVP deliverable or a scheduled task. Each row records only its trigger; when a trigger fires, the row moves into its own sub-app requirements or Issue at that moment — not before.

| Item | Trigger |
|---|---|
| WebView container capability (Web-type sub-app integration: a URL + name becomes a sub-app; zero bridge, no credential injection) | when the first Web-type sub-app is proposed |
| Home-screen polish (sorting / grouping / badge contract) | when sub-apps ≥ 3 |
| A cross-sub-app data-sharing convention (who owns, who reads) | when the first cross-sub-app data-sharing need appears |

## 5. P2 — deferred capabilities (outside the MVP; always consumer-triggered)

Nothing here is built or scheduled up front. A P2 item starts existing as work only when its first consumer is proposed, then follows the rule of three.

- **iOS version** (if proposed: evaluate via the §7 migration path first; no commitment)
- Network egress channel, databases, and other shared-service expansions — **nothing prebuilt**; whoever needs it first pulls it in and promotes it per the rule of three
- Host-side support for system-overlay sub-apps (WindowManager tier, see architecture-decisions §4) — when the first overlay sub-app is proposed

## 6. Non-functional requirements

| Dimension | Requirement |
|---|---|
| Distribution | Single APK, personal install, no store-compliance burden |
| Offline | The foundation is fully offline; no integration requirement may force networking |
| Privacy | Credentials pass only through the encrypted store (never plaintext on disk or in logs); the foundation collects no data |
| Footprint | This round creates exactly 3 Gradle modules: `:composeApp` / `:shared` / `:feature-sample`; no empty future-extension modules |

## 7. Platform strategy: start Android-only, pay no KMP tax upfront

- **Decision**: no KMP three-layer source sets (commonMain/androidMain/iosMain), no expect/actual, no platform-availability declarations. Rationale:
  1. Platform-privileged capabilities (screenshots, screen reading, overlays) are forbidden on iOS; the ceiling for cross-platform value is low;
  2. Whether future sub-apps lean utility or privileged is unknown now — **don't bet on unknown needs**;
  3. The KMP structural tax (every class passing an "does this exist on iOS?" audit, dual-platform implementations, dual test matrices) is pure cost before the second platform appears.
- **Portability discipline (the only retained constraint)**: `:shared` has no UI/Compose dependency; platform-sensitive points (Keystore credentials, file paths) concentrate in a few classes. This discipline is unrelated to cross-platform — it is what a microkernel demands of its contract layer anyway.
- **If iOS is proposed later**: migration = add the KMP plugin to `:shared`, create the three-layer directories, move pure logic to commonMain, move Keystore to androidMain and add a Keychain actual — contract and dependency iron rules unchanged, sub-apps untouched — a **directory-shuffling chore, not a teardown refactor**. Only then does the registry entry reintroduce a "platform availability" field.

## 8. Milestones

| Stage | Content | Exit criteria |
|---|---|---|
| M0 Skeleton | Shell + registry + sample visible and enterable | FR-1/FR-2 minimal closed loop |
| M1 Services & storage | Four shared services + the sample's own config persistence | FR-3/FR-4 accepted; config survives restart |
| M2 Control & acceptance | Settings switches + permission overview + sample mount/unmount verification | FR-5/FR-6 accepted; sample finally restored |
| M3 (optional, no commitment) | iOS version (if proposed) | scheduled only after the §7 migration-path evaluation passes |

**MVP Definition of Done**: the host foundation and the single `:feature-sample` run stably on a real device; every acceptance item in §3 passes; the sample's entry, shared services, and own-config persistence loop works end to end. After the temporary-removal verification, the sample is restored — **neither a second sub-app nor any shared business capability is a delivery condition**.

## 9. Explicit non-goals (red lines)

- ❌ **Defining any sub-app's concrete requirements or features** (this document's charter is to not overstep)
- ❌ Prebuilding abstractions and services without consumers (network/database/overlay support all wait for the first consumer)
- ❌ **Paying the KMP structural tax for an imaginary iOS** (evaluate when iOS is actually proposed)
- ❌ Runtime pluginization, dynamic dex loading, plugin marketplaces
- ❌ Declarative manifest files, API version freezing, process isolation
- ❌ Account systems, cloud sync, multi-device
- ❌ Store publication

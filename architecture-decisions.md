# UniApp Architecture Decisions — Host Foundation

> Version: v2.2 (2026-11-25) | Companion: [mvp-requirements.md](mvp-requirements.md)
> Goal this round: **minimal host foundation + the single `:feature-sample`**, validating entry, shared services, and own-config persistence. No second sub-app; no prebuilt shared business capabilities. This document constrains the integration surface; future extension rules do not represent current construction items.

## 1. Selection conclusions

| # | Decision | Conclusion | Rationale |
|---|---|---|---|
| D1 | Overall shape | Plain microkernel: shell + island modules + a few shared services | No ecosystem distribution, no runtime swapping — heavier shapes are pure maintenance cost |
| D2 | Sub-app tier | A compile-time Gradle module is the sub-app + a code-level registry | Mount = new module + one entry; unmount = remove module + unregister; no runtime machinery |
| D3 | Page UI | Single Activity + Compose NavHost | The officially recommended direction, no legacy baggage |
| D4 | Integration shape | Native-page tier (in the MVP); WebView and system-overlay tiers are consumer-triggered | Three tiers cover the future shape spectrum; only tier 1 lands now |
| D5 | Cross-platform | **No KMP; start Android-only** | Privileged capabilities cannot be replicated on iOS, sub-app shapes unknown — don't bet; `:shared` keeps only the portability discipline (§5) |
| D6 | DI container | Koin | For service assembly and interface injection; per-platform implementations and initialization are still handled separately |

## 2. Frame of reference

**In spirit we take DSH's three disciplines; in mechanism we do not take VSCode's; in scale we are far smaller than DSH.**

- Taken: ① service registry (obtained through the container, never importing someone else's implementation) ② typed events (events as extension points) ③ reversible registration (every registration comes with rollback).
- Not taken: zero-privilege kernel, layered patches, hot-plugging (no "unchangeable official core" to route around); declarative manifests, API freezing, process isolation (a sub-app is compile-time code we own — **the Gradle module is the manifest**, registry entries add fields as needed).
- This is not a compromise but a third thing — a plain microkernel. This project's biggest risk is **over-engineering**: in a single-person repo, every extra layer of indirection is pure maintenance cost.

## 3. Module layout (Android-only)

```
UniApp/
├── :composeApp       # application: MainActivity, home screen, host settings, navigation, startup assembly
├── :shared           # Android library: registry/events/credentials/settings/di
└── :feature-sample   # Android library: sample pages, page logic, own-config persistence

# Created only when a real need appears: :feature-*, :integration-*
```

Note: `:shared` is demoted from KMP three-layer source sets to a plain module — **cross-platform is merely its side duty; contract isolation plus the service container are its primary duty**, which a pure-Android project needs regardless (the landing point of the dependency iron rules).

**Dependency direction**: `:composeApp → :feature-sample → :shared`, plus `:composeApp → :shared`.

- The host's **assembly point** explicitly depends on the sample's public integration API and registers the entry, installs DI modules, and mounts routes; the home screen and generic navigation never touch the sample's internals.
- The sample does not depend on the host; future sub-apps never import each other, and may depend on shared capability libraries with a genuine need. `:shared` never depends on the host or sub-apps, nor on Compose.
- Adding or removing the sample may change Gradle module declarations, dependencies, and the central assembly list; **the generic home screen, navigation mechanism, and shared services require no changes**. "The foundation stays put" does not include the assembly list.
- Host settings (enable/disable, permission overview, credential entry) live in `:composeApp`; the sample's pages, page logic, and own settings live in `:feature-sample`.

**Entry integration: stay type-safe**. `SubAppEntry` carries only UI-free metadata — id, name, route. The sample exposes a function registering its Compose routes, invoked by the host assembly point; Compose types exist only at those two ends. **No `Any` or opaque-factory casts** to smuggle Compose dependencies; if multiple sub-apps later genuinely need a unified UI interface, a separate UI contract module is considered then — not built this round.

**Where shared business capabilities are planned (LLM as the example)**. The foundation `:shared` holds only host-necessary contracts and services, **no business capabilities**. Capabilities like LLM that "multiple sub-apps might share" follow the rule of three:

1. The first user appears → the capability is **hard-coded inside that user's own feature module** (no new module);
2. A second user appears → create the `:integration-llm` module and sink the capability out of the first module; the interface lives in that integration module, never in `:shared` (keeping the foundation from fattening on business capabilities);
3. Hard constraint: **secrets always go through `:shared`'s `CredentialStore`** (no reinvented storage); heavy dependencies like HTTP clients hang only on the integration module, never entering `:shared`.

Dependency direction: `:feature-a ──► :integration-llm ──► :shared`. Integration modules are peers of feature modules (both are library modules); integration never depends on any feature.

**Feature vs. integration is a role convention**: a sub-app with a user entry is called a feature; a shared capability library being consumed is called an integration. Both are ordinary library modules in Gradle — **naming neither dissolves dependencies nor decides plugin composition**. Removing any depended-on module must handle its consumers; this round only forbids sub-apps depending on each other and implements no runtime dependency resolution. If a future LLM gains model management, testing, and user pages, it can offer capability and UI at once; the split decision comes then. Not created this round.

**Data storage: unambiguous ownership, implement on demand**.

1. **Foundation data** → `:shared`: registry enable/disable state, global config, credentials;
2. **The sample's own config** → `:feature-sample`: config keys in the sample namespace, persistence wrappers in its own `data/` package, free to reuse `:shared`'s KV service. The MVP verifies one input surviving a restart; Room not required;
3. **Future business data** → whichever module owns the data: entities, DAOs, repositories, and migrations follow the data owner. When Room appears, prefer per-ownership database files over independent sub-apps sharing business tables; a separate database is this project's default strategy, not Room's only supported shape.

No prebuilt `storage/` package, Room factory, unified backup, or orphan-data cleanup; extract infrastructure free of business knowledge only when duplication actually appears. Removing a module and deleting user data are distinct operations; on-disk data is never auto-deleted. Future cross-module access goes through the data owner's exported query/export contracts; the event bus only notifies — it never substitutes for a persistent data source.

## 4. Three integration tiers

| Tier | Shape | When enabled |
|---|---|---|
| ① Native page | Compose page + NavHost route | In the MVP (the sample is this tier) |
| ② WebView | An entry with a URL becomes a sub-app; zero bridge, no credential injection | when the first Web-type sub-app is proposed |
| ③ System overlay | WindowManager hosts the cross-app overlay; pick View or ComposeView then, and handle permissions and lifecycle | when the first overlay sub-app is proposed |

Only tier ① is implemented this round. WebView content trust, permissions, and isolation policy, plus overlay-page interplay, are designed after a real consumer appears.

## 5. Platform strategy: start Android-only

- **Not doing**: KMP three-layer source sets, expect/actual, platform-availability fields. Rationale: ① privileged capabilities are iOS-forbidden, so the cross-platform ceiling is low; ② future sub-app shapes are unknown — don't bet; ③ the structural tax is pure cost before the second platform arrives.
- **Portability discipline (the only retained rule)**: `:shared` has no UI/Compose dependency; platform-sensitive points (Keystore, file paths) concentrate in a few classes. This discipline is what a microkernel demands of its contract layer — it has nothing to do with KMP.
- **If iOS is proposed**: re-evaluate platform APIs, dependencies, storage, and UI portability, then decide on KMP source sets and platform implementations. The current isolation helps migration but promises neither "just shuffle directories" nor "sub-apps need no changes".
- The UI approach (SwiftUI shell vs. Compose Multiplatform) is decided independently then.

## 6. Technology stack

| Purpose | Choice | Notes |
|---|---|---|
| Language | Kotlin 2.x | |
| Page UI | Jetpack Compose + Navigation | Single Activity |
| DI | Koin | Service registry |
| Events | StateFlow / SharedFlow | Typed events |
| Config | DataStore | Non-sensitive config |
| Credentials | Android Keystore + encrypted at rest | Keystore manages the encryption keys; ciphertext at rest; plaintext persistence or logging forbidden |
| Files | Chosen on demand | The sample's KV verification adds no file abstraction |
| Serialization | kotlinx-serialization on demand | Not added without a real encoding need |
| Network, databases, etc. | **Nothing prebuilt** | Pulled in by the first consumer |

## 7. Mapping the DSH three disciplines

| DSH discipline | Landing in this project |
|---|---|
| ① Service registry | Koin container + `:shared` contract interfaces; consumers depend on interfaces only |
| ② Typed events | Host-level SharedFlow event contracts; subscribers declare subscriptions, zero cross-module holding |
| ③ Reversible registration | Subscriptions cancel with their coroutine scope; external listeners and resources are released explicitly in `onDispose`/`finally`. Removing a Compose page does not automatically roll back system registrations |

## 8. Guardrails (matter more than the architecture)

1. **Extract on actual reuse**: implement a first need in its owning module; evaluate extraction only when a second real user appears. Host-generic services go to `:shared`, shared business capabilities to `:integration-*`; code never auto-sinks into the foundation.
2. **No empty modules**: exactly three modules exist; no UI-contract, database-factory, backup, or future-extension placeholder modules.
3. **Run one sample through first**: registration, pages, all four shared services, and own-config persistence; sub-apps carry no independent application, and no second sub-app is required.
4. **Central assembly, isolated internals**: integration changes are confined to module declarations, dependencies, and the assembly list; the generic host UI and shared services never follow business changes.
5. **Acceptance closed loop**: after temporarily removing the sample, the host builds, runs, and shows the empty state; restore it afterwards; the final delivery always keeps the single sample — consistent with FR-6 of the requirements document.

## 9. Negative list (explicitly not introduced)

- ❌ dex dynamic loading, runtime plugin frameworks (RePlugin/VirtualAPK, etc.)
- ❌ Declarative plugin manifests (JSON manifest), API freezing, activation-event systems
- ❌ Sub-app processes
- ❌ Hilt (Koin is lighter; its DSL matches the registry mindset)
- ❌ **KMP source-set structure and expect/actual** (re-evaluated when iOS is proposed)
- ❌ Kotlin/Wasm / Web target
- ❌ Prebuilt consumer-less shared services

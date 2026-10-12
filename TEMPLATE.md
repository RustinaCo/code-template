# Rustinaz Template — The Complete Guide

**A developer and AI reference for the `Rustinaz/code-template` workspace.**

This document is the companion to [README.md](README.md). The README explains *what* the
template is and how to get it running; **this guide explains how it is built and how to write
code inside it** — the crate-by-crate contract, the invariants a change must not break, and a
set of task recipes.

It is written so that **a person and an AI coding assistant can both work from it without
guessing**: every path, crate and public API it mentions exists in this repository.

- 🇮🇷 **نسخهٔ فارسی این راهنما:** [TEMPLATE.fa.md](TEMPLATE.fa.md)
- 🇬🇧 **Getting started (English):** [README.md](README.md)
- 🇮🇷 **شروع سریع (فارسی):** [README.fa.md](README.fa.md)

> Author: **Sina Khanzadeh** · <khanzadeh.1377@gmail.com> · [sina-khanzadeh.ir](https://sina-khanzadeh.ir)
> Instagram / Telegram / Discord: **programmer_1998**
> Licence: **MIT OR Apache-2.0**

---

## Table of contents

1. [How to use this document](#1-how-to-use-this-document)
2. [What the template is](#2-what-the-template-is)
3. [Architecture at a glance](#3-architecture-at-a-glance)
4. [The complete file tree](#4-the-complete-file-tree)
5. [The crates, one by one](#5-the-crates-one-by-one)
6. [The UI layer in detail](#6-the-ui-layer-in-detail)
7. [The apps](#7-the-apps)
8. [The wire contract and the bundled backend](#8-the-wire-contract-and-the-bundled-backend)
9. [Feature flags and the `cfg` matrix](#9-feature-flags-and-the-cfg-matrix)
10. [Commands: build, run, test, lint](#10-commands-build-run-test-lint)
11. [Continuous integration](#11-continuous-integration)
12. [Recipes: how to add things](#12-recipes-how-to-add-things)
13. [Invariants a change must not break](#13-invariants-a-change-must-not-break)
14. [Troubleshooting and gotchas](#14-troubleshooting-and-gotchas)
15. [Notes for AI coding assistants](#15-notes-for-ai-coding-assistants)
16. [References and contact](#16-references-and-contact)

---

## 1. How to use this document

**If you are a human developer:** read §2–§3 to load the mental model, skim §5 as a map, then
use §12 (recipes) whenever you add something. §14 collects the traps that are easy to hit.

**If you are an AI assistant:** before writing any code,

1. read §3 (the dependency graph — it dictates where new code belongs),
2. read §5 (which crate owns which responsibility),
3. pick the matching recipe in §12,
4. obey §13 (invariants) and §9 (feature flags / `cfg`),
5. run the commands in §10 to prove the change.

Do **not** invent crates, modules or APIs. Everything that exists is listed in §4 and §5. If
something is not there, the recipe in §12 tells you where it belongs.

---

## 2. What the template is

Rustinaz is a **Rust-first cross-platform application template**. One workspace produces:

| Target | Artifact | Where the entry point lives |
| --- | --- | --- |
| Linux / Windows / macOS | a native desktop binary | `apps/example/src/main.rs` → `example_app::run()` |
| Android | a `.so` loaded by `android.app.NativeActivity` | `apps/example/src/lib.rs` → `android_main()` |
| iOS | a static library linked into an Xcode app | `apps/example/src/lib.rs` (the `staticlib` crate type) |
| Web | a `.wasm` + `index.html` bundled by Trunk | `apps/example/src/lib.rs` → `run_web()` |
| (any) | the bundled JSON backend | `apps/server/src/main.rs` |

The bet the template makes is simple: **write the UI once in egui and ship the same widget tree
to every target.** Only what genuinely differs per operating system — window creation, theme
defaults, device information, accessibility announcements, JNI — lives in per-platform code.

There is **no Java, Kotlin, Swift, or HTML/CSS application code**. The Android project
(`android-app/`) contains only Gradle build scripts (Kotlin DSL) and XML *resources* (manifest,
theme, icon) — the APK ships zero `.class` files and the Rust `.so` draws every pixel.

---

## 3. Architecture at a glance

### 3.1 The dependency graph (authoritative)

These are the **real** edges, taken from `cargo metadata`. The direction is strictly one-way;
nothing below ever depends on something above it.

```
                       ┌──────────────┐
                       │   shared     │   domain, config, errors, api, services   (no UI, no OS)
                       └──────┬───────┘
              ┌───────────────┼────────────────┬─────────────────┐
              │               │                │                 │
       ┌──────▼──────┐ ┌──────▼──────┐  ┌──────▼───────┐  ┌──────▼────────┐
       │     ui      │ │  resources  │  │ build-config │  │ platform/*    │
       │ (egui,      │ │ (strings,   │  │ (version,    │  │ (linux, …)    │
       │  tokens,    │ │  colours)   │  │  signing)    │  │               │
       │  components)│ └──────┬──────┘  └──────────────┘  └──────┬────────┘
       └──────┬──────┘        │                                 │
              │               │        ui  is also a dependency ─┘
              │               │        of every platform/* crate
              └───────────────┴───────────────┬─────────────────┘
                                              │
                                    ┌─────────▼──────────┐        ┌──────────┐
                                    │  apps/example      │        │apps/server│
                                    │  (the application) │        │ (backend) │
                                    └────────────────────┘        └─────┬────┘
                                                                        │
                                                                  shared only
```

Read aloud:

- **`shared` depends on nothing internal.** Domain types, config, errors, the wire contract
  (`api`) and the service traits + in-memory implementations.
- **`ui` depends on `shared`.** Theme tokens, layout maths, navigation, the component library,
  and the `PlatformUi`/`Window` traits.
- **`platform/*` depend on `shared` and `ui`.** They add the OS glue on top of `ui`'s traits.
- **`resources` and `build-config` depend on `shared`.**
- **`apps/example` depends on `shared`, `ui`, `resources` and all six `platform/*` crates**
  (each gated by `#[cfg(target_os = …)]`, see §7.1).
- **`apps/server` depends on `shared` *only*** — the backend never pulls `ui` or a platform
  crate into its binary, so it carries no GUI code.

> **Two things are called "platform".** `ui` ships its own `PlatformUi` implementations under
> `crates/ui/src/platform/`, while the OS-facing crates live in `crates/platform/*`. Only the
> latter are separate workspace members, and they depend on `ui` — never the reverse.

### 3.2 How a run is wired

Desktop and Android both funnel through the same boot sequence in `apps/example/src/lib.rs`:

```
main() / android_main() / run_web()
        │
        ▼
  bootstrap(&CliOptions)
        │  1. AppConfig::load()  (or --config PATH)
        │  2. apply --platform / --server overrides
        │  3. shared::init_blocking(config)        → tracing + globals
        │  4. platform_*::init()                   → OS setup (per target)
        │  5. ResourceManager::init_global(config) → strings/colours/themes
        │  6. UiBuilder::new(config)
        │        .platform(platform)               → corrects host → device
        │        .framework(UiFramework::Egui)
        │        .build()                          → UiContext
        ▼
  eframe::run_native(...)  (desktop)   |   WebRunner::start(...)  (web)
        │
        ▼
  ExampleApp::update(&egui::Context)   → draws with egui, reads UiContext
```

`UiContext` (in `crates/ui`) is the object the UI reads from: the shared `AppConfig`, the
detected `DeviceInfo`, the resolved `ThemeTokens`, and the active `UiFramework`.

---

## 4. The complete file tree

Everything below exists in the repository (generated output such as `target/`,
`apps/example/dist/` and `android-app/build/` is omitted).

```
code-template/
├── Cargo.toml                      workspace: 12 members, workspace dependency versions, profiles
├── Cargo.lock                      committed on purpose: reproducible first build
├── rust-toolchain.toml             pins stable + rustfmt + clippy (editors and CI agree)
├── .editorconfig                   UTF-8, LF, 4-space (2 for TOML/YAML)
├── .gitignore
├── .cargo/
│   └── config.toml                 net settings + the short cargo aliases (no hard-coded linkers)
├── .github/
│   ├── dependabot.yml
│   └── workflows/ci.yml            the six CI gates
├── .vscode/                        committed run config, tasks, rust-analyzer settings, extensions
│
├── apps/
│   ├── example/                    ← the app you actually edit
│   │   ├── Cargo.toml              crate-type = rlib, cdylib, staticlib; per-target deps
│   │   ├── Trunk.toml              web build config
│   │   ├── index.html              the <canvas> eframe mounts into (id = the_canvas_id)
│   │   └── src/
│   │       ├── lib.rs              run(), run_web(), android_main(), bootstrap(), the UI
│   │       ├── main.rs             thin desktop binary → example_app::run()
│   │       └── backend.rs          the client's Backend panel (native) / placeholder (web)
│   └── server/                     the bundled backend
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs              routing: endpoint constants → handlers; serve()
│           ├── http.rs             hand-written HTTP/1.1 parse + response write
│           └── main.rs             clap binary (--bind, --no-demo), demo seeding
│
├── crates/
│   ├── shared/                     business logic (no UI, no OS)
│   │   ├── Cargo.toml              the network feature lives here
│   │   └── src/
│   │       ├── lib.rs              init()/init_blocking(), re-exports, DefaultNetworkService alias
│   │       ├── domain/mod.rs       User, AppSettings, DeviceInfo, Platform, ThemeTokens, Route, …
│   │       ├── config/mod.rs       AppConfig, ConfigBuilder, ThemeMode, FeatureFlags
│   │       ├── errors/mod.rs       AppError + sub-errors, ErrorContext
│   │       ├── api/mod.rs          the wire contract: endpoints + request/response types
│   │       └── services/
│   │           ├── mod.rs          service traits, in-memory impls, ServiceContainer, factory
│   │           └── http.rs         HttpNetworkService: std-only blocking HTTP/1.1 client
│   ├── ui/                         shared UI (this is where the widget library lives)
│   │   ├── Cargo.toml              the egui feature lives here (default on)
│   │   └── src/
│   │       ├── lib.rs              UiContext, UiBuilder, UiFramework, responsive helpers
│   │       ├── theme/mod.rs        ThemeManager, ThemeBuilder, built-in themes, CSS export
│   │       ├── layout/mod.rs       LayoutCalculator, FlexLayout, GridLayout, builders
│   │       ├── navigation/mod.rs   NavigationController, deep links, bottom-nav/drawer configs
│   │       ├── components/         the component library
│   │       │   ├── mod.rs          Component trait, ComponentProps, colour helpers, re-exports
│   │       │   ├── app_bar.rs      AppBar (+ BACK_ACTION)
│   │       │   ├── button.rs       Button, ButtonStyle, MIN_TOUCH_TARGET
│   │       │   ├── card.rs         Card
│   │       │   ├── dialog.rs       Dialog, DialogState
│   │       │   ├── navigation_bar.rs  NavItem, NavigationBar
│   │       │   ├── scaffold.rs     Scaffold, ScaffoldLayout, Slot
│   │       │   └── text_field.rs   TextField, TextFieldState
│   │       ├── platform/
│   │       │   ├── mod.rs          PlatformUi + Window traits, PlatformTheme, PlatformFactory,
│   │       │   │                   keyboard/pointer/text-input behaviours
│   │       │   ├── linux.rs        LinuxUi / LinuxWindow      (in-crate PlatformUi impls)
│   │       │   ├── windows.rs      WindowsUi / WindowsWindow
│   │       │   ├── macos.rs        MacosUi / MacosWindow
│   │       │   ├── android.rs      AndroidUi / AndroidWindow
│   │       │   ├── ios.rs          IosUi / IosWindow
│   │       │   └── web.rs          WebUi / WebWindow
│   │       ├── hooks/mod.rs        present but NOT declared (not part of the egui stack yet)
│   │       └── utils/mod.rs        present but NOT declared (not part of the egui stack yet)
│   ├── resources/                  strings, colours, themes, assets (runtime loaded)
│   │   ├── Cargo.toml
│   │   └── src/lib.rs              ResourceManager, ResourceBuilder, StringMap, ColorMap
│   ├── build-config/               version / target / signing metadata
│   │   ├── Cargo.toml
│   │   └── src/lib.rs              BuildTimeConfig, VersionInfo, platform_build_config, signing
│   └── platform/                   the OS glue (one crate per target)
│       ├── linux/     src/lib.rs   platform_info(), init()
│       ├── windows/   src/lib.rs   init(), window/theme helpers
│       ├── macos/     src/lib.rs   init(), window/theme helpers
│       ├── android/   src/lib.rs   AndroidApp, install_app/current_app, device_info(), JNI, init()
│       ├── ios/       src/lib.rs   IosInfo, device_info(), init(), haptics
│       └── web/       src/lib.rs   user agent, viewport, touch detection
│
├── android-app/                    Android Studio packaging project (no app source)
│   ├── settings.gradle.kts
│   ├── build.gradle.kts
│   ├── gradle.properties
│   ├── gradlew  /  gradlew.bat
│   ├── gradle/wrapper/…            wrapper jar + properties
│   └── app/
│       ├── build.gradle.kts        runs Cargo per ABI, then packages the .so
│       ├── proguard-rules.pro
│       └── src/main/
│           ├── AndroidManifest.xml points at android.app.NativeActivity
│           └── res/…               strings, theme, launcher icon
│
├── resources/                      shared data files
│   ├── colors/colors.json
│   └── strings/strings_en.json  ·  strings_es.json
│
└── scripts/                        one script per platform, plus the dev loop
    ├── android-env.sh              exports NDK linkers/CC for Cargo (source it)
    ├── build-all.sh                build every target this machine can
    ├── build-linux.sh  build-windows.sh  build-macos.sh  build-ios.sh
    ├── build-android.sh  build-web.sh
    └── dev.sh                      start the backend, wait for the port, run the client
```

---

## 5. The crates, one by one

### 5.1 `shared`

The business-logic crate. **No `egui`, no OS types.** Re-exported from the crate root
(`crates/shared/src/lib.rs`) so downstream code can `use shared::…`.

| Module | Owns | Key public items |
| --- | --- | --- |
| `shared::domain` | Core data model | `EntityId` (= `Uuid`), `Timestamp`, `Entity`/`RepositoryEntity` traits, `User`, `UserPreferences`, `AppSettings`, `AccessibilitySettings`, `WindowState`, `WindowPosition`, `DeviceInfo`, `Platform`, `FormFactor`, `ScreenSize`, `Orientation`, `SafeArea`, `Route`, `ThemePreference`, `ThemeTokens`, `ColorTokens`/`ColorRole`/`ColorValue`, `TypographyTokens`, `SpacingTokens`, `ShapeTokens`, `MotionTokens`, `ElevationTokens`, `Shadow`, `BuildConfig`, `SigningConfig` |
| `shared::config` | Runtime configuration | `AppConfig`, `ConfigBuilder`, `AppInfo`, `PlatformConfig`, `UiConfig`, `ThemeConfig`, `ThemeMode`, `ResponsiveBreakpoints`, `NetworkConfig`, `StorageConfig`, `LoggingConfig`, `LogLevel`/`LogFormat`/`LogOutput`, `FeatureFlags`. `AppConfig::load()`, `::save()`, `::device_info()`, `::theme_tokens()`, `::is_feature_enabled()` |
| `shared::errors` | The single error type | `AppError` (wraps `ConfigError`, `NetworkError`, `StorageError`, `AuthError`, `ValidationError`, `PlatformError`, `UiError`, `SerializationError`, plus `Io`, `Http`, `Json`, `NotFound`, `Timeout`, `Unsupported`, `Multiple`), `Result<T>`, `ErrorContext`/`ErrorContextExt` |
| `shared::api` | **The wire contract** | `endpoints::{HEALTH, USERS, USERS_SEARCH}`, `HealthResponse`, `ErrorResponse`, `RegisterRequest`, `UserList`, … |
| `shared::services` | Service traits + in-memory impls | traits `AuthService`, `UserService`, `SettingsService`, `NetworkService`, `StorageService`, `PlatformService`, `AnalyticsService`, `NotificationService`; `InMemoryRepository<T>`; `ServiceContainer`, `AppServices`, `PlatformServices`; `services::defaults::*`; `services::factory::create_services()`; `HttpNetworkService` |

**Initialisation.** `shared::init_blocking(config) -> Result<Arc<AppConfig>>` loads config,
installs `tracing`, and returns the shared config. `shared::init()` is the same, `async`, for
callers inside a runtime. Use `init_blocking` from `main`, `android_main` or a `extern "C"`
entry point.

**The network aliasing trick.** `shared` re-exports *one* name, `DefaultNetworkService`, which
is either the `reqwest`-backed client (feature `network`) or `StubNetworkService` (default).
Calling code is identical either way and simply gets `AppError::Unsupported` at runtime when the
stub is compiled in. `HttpNetworkService` is separate and always available off-web.

### 5.2 `ui`

The shared UI crate. Two halves, split by the `egui` feature (default on):

- **Pure logic** (always compiles): tokens, layout maths, builders, component state and sizing.
- **Rendering** (`#[cfg(feature = "egui")]`): the `show(&mut egui::Ui, …)` methods.

| Module | Key public items |
| --- | --- |
| `ui` (root) | `UiFramework`, `UiContext`, `UiBuilder`, module `ui::responsive` |
| `ui::theme` | `ThemeManager`, `ThemeBuilder`, `ThemeChangeListener`; built-in themes `material3()`, `ios()`, `material_you()`, `high_contrast()`, `minimal()`; `generate_css()` |
| `ui::layout` | `LayoutConstraints`, `LayoutCalculator`, `FlexLayout`, `GridLayout`, `LayoutBuilder`, `FlexLayoutBuilder`, `GridLayoutBuilder`, `LayoutDirection`, `FlexDirection`, `JustifyContent`, `AlignItems`, `AlignContent`, `FlexWrap`, `GridAutoFlow` |
| `ui::navigation` | `NavigationController`, `NavEntry`, `NavDirection`, `DeepLinkHandler`, `NavigationBuilder`, `NavigationListener`, `BottomNavConfig`/`BottomNavItem`, `NavigationDrawerConfig`/`DrawerHeader`/`DrawerItem`/`DrawerFooter` |
| `ui::components` | see §5.2.1 |
| `ui::platform` | `PlatformUi`, `Window`, `WindowConfig`, `PlatformTheme`, `PlatformColors`, `PlatformFonts`, `PlatformSpacing`, `PlatformShapes`, `PlatformMotion`, `PlatformFactory`, and the `keyboard`/`pointer`/`text_input` behaviour helpers |

#### 5.2.1 The component library (`ui::components`)

Every component follows the same **two-part shape**: pure logic (builders, state, sizing) that
compiles with `egui` off, and a `show(…)` method gated behind `#[cfg(feature = "egui")]`.

| Component | Constructed with | Renders with |
| --- | --- | --- |
| `AppBar` | `AppBar::new(title)` + `.subtitle()`, `.action()`, `.back_enabled()`, `.center_title()`, `.elevated()` | `show(ui, …)` (feature `egui`) |
| `Button` | `Button::new(label)` + `.style(ButtonStyle)`, `.size(ComponentSize)`, `.enabled()`, `.icon(char)`, `.full_width()` | returns a `ButtonResponse` |
| `Card` | `Card::new(title)` + `.subtitle()`, `.elevation(f32)`, `.padding_multiplier()`, `.clickable()` | `show(ui, ctx) -> ButtonResponse` / `show_with(…)` |
| `Dialog` | `Dialog::new(title)` + `.body()`, `.dismissable()`, `.visible()`, `.add_action()` | `show(…)`, driven by a `DialogState` |
| `NavigationBar` | `NavigationBar::new(Vec<NavItem>)` + `.items()`, `.select()`, `.select_next()`, `.show_labels()` | `show(…)` |
| `Scaffold` | `ScaffoldLayout::new(app_bar, navigation_bar, body)` of `Slot`s | layout maths; `ScaffoldResponse` |
| `TextField` | `TextField::new()`; state via `TextFieldState::new(placeholder)` + `.with_max_length()`, `.with_password()`, `.with_disabled()`, `.set_value()`, `.take_changed()` | `show(…)` |

Also public: `Component` trait, `ComponentProps`, `ComponentVariant`, `ComponentSize`,
`ComponentState`, `AccessibilityRole`, `ButtonResponse`, `MIN_TOUCH_TARGET`, `BACK_ACTION`, and
the colour helpers `role_value`, `value`, `role`, `token`, `with_alpha`, `state_fill`.

### 5.3 `platform/*` (the OS glue)

One crate per target. Each depends on `shared` and `ui` and exposes a small, imperative API plus
a `PlatformUi` implementation where the OS has something to say (window, theme, accessibility).

| Crate | Public surface | Used from |
| --- | --- | --- |
| `platform-linux` | `platform_info()`, `PlatformInfo`, `init()` | `apps/example` `init_platform()` |
| `platform-windows` | `init()`, window/theme helpers | `apps/example` |
| `platform-macos` | `init()`, window/theme helpers | `apps/example` |
| `platform-android` | `AndroidApp`, `install_app()`, `current_app()`, `device_info()`, `AndroidInfo`, `init()`, `material3_theme()`, `AndroidPlatformUi`, `AndroidWindow` | `apps/example` `android_main()` |
| `platform-ios` | `IosInfo`, `device_info()`, `init()`, haptics | `apps/example` |
| `platform-web` | user-agent, viewport and touch detection | `apps/example` (via the web target) |

> The `ui` crate *also* contains per-platform `PlatformUi` implementations under
> `crates/ui/src/platform/*.rs`. The distinction is deliberate: **`ui::platform` defines the
> abstractions and default impls**; the **`crates/platform/*` crates do the real OS work**
> (JNI, native windows, `uname`, `Win32` API, `web-sys`). When you need the OS, edit
> `crates/platform/*`; when you need the shape/contract, edit `crates/ui/src/platform/mod.rs`.

### 5.4 `resources`

Runtime resource loading: strings, colours, themes, assets and fonts, loaded from the
`resources/` directory and cached process-wide.

- `ResourceManager::init_global(config) -> Arc<ResourceManager>`, `::global()`, `::global_or_init()`
- `load_all()`, `reload()`, `add_resource_dir(path)`
- Getters: `get_string(key, locale)`, `get_string_formatted(…)`, `get_color(name)`,
  `get_theme(name)`, `current_theme()`, `get_asset(name)`, `get_asset_data(name)`, `get_font(name)`,
  `get_raw(name)`
- `ResourceBuilder` + `StringMap` / `ColorMap` for building resources in code

Data files: `resources/strings/strings_en.json`, `resources/strings/strings_es.json`,
`resources/colors/colors.json`.

### 5.5 `build-config`

Version, target and signing metadata.

- `generate_build_metadata() -> BuildTimeConfig`, with `VersionInfo`, `GitInfo`, `RustcInfo`,
  `TargetInfo`
- `platform_build_config(platform) -> BuildConfig`
- Signing: `load_signing_config`, `save_signing_config`, `android_signing_config`,
  `ios_signing_config`
- `BuildTarget::all_targets()`, `target_for_platform(platform)`
- A small `clap` `Cli`/`Commands` for printing build info

---

## 6. The UI layer in detail

### 6.1 `UiContext` and `UiBuilder`

`UiContext` is what the drawing code reads; `UiBuilder` is what produces it during boot.

```rust
// crates/ui/src/lib.rs
pub struct UiContext {
    pub config: Arc<RwLock<AppConfig>>,  // parking_lot::RwLock
    pub device_info: DeviceInfo,
    pub theme_tokens: ThemeTokens,
    pub framework: UiFramework,
}

impl UiContext {
    pub fn platform(&self) -> Platform;
    pub fn is_mobile(&self) -> bool;
    pub fn is_desktop(&self) -> bool;
    pub fn theme_preference(&self) -> ThemePreference;
    pub fn color_tokens(&self) -> &ColorTokens;
    pub fn spacing(&self) -> &SpacingTokens;
    pub fn typography(&self) -> &TypographyTokens;
}
```

```rust
let ui_context = UiBuilder::new(config)          // Arc<RwLock<AppConfig>>
    .platform(Platform::Android)                 // device, not the build host
    .framework(UiFramework::Egui)                // egui is the only backend today
    .build();
```

`.platform(...)` matters on Android, where the config file records the *build host's* platform.
`.device_info(...)` overrides the detected `DeviceInfo` entirely.

### 6.2 Theme

`ThemeManager` holds a `ThemeTokens` and notifies listeners on change. `ThemeBuilder` produces
tokens; the built-in themes are `material3()`, `ios()`, `material_you()`, `high_contrast()` and
`minimal()`. `generate_css(&tokens)` renders CSS for a web surface.

### 6.3 Layout

`LayoutCalculator::new(device_info)` answers the responsive questions the widgets ask:
`grid_columns(base)`, `spacing(base)`, `font_scale(base)`, `is_compact()`, `is_two_pane()`,
`safe_area()`, `breakpoint()`, `is_landscape()` / `is_portrait()`.

The builders `LayoutBuilder`, `FlexLayoutBuilder`, `GridLayoutBuilder` describe flex and grid
layouts declaratively, and `ui::responsive` offers free helpers (`grid_columns`,
`spacing_multiplier`, `font_scale`, `is_compact`, `is_two_pane`).

### 6.4 Navigation

`NavigationController` keeps a route stack (`navigate`, `push`, `pop`, `replace`, `reset`,
`current`, `previous`, `stack`, `can_go_back`, `depth`, `clear`) and notifies
`NavigationListener`s. `DeepLinkHandler` maps URL patterns to `Route`s.
`NavigationBuilder::new(initial_route).route(name, route).deep_link(pattern, resolver).build()`.

### 6.5 The `PlatformUi` contract

```rust
// crates/ui/src/platform/mod.rs
pub trait PlatformUi: Send + Sync {
    fn platform(&self) -> Platform;
    fn create_window(&self, config: WindowConfig) -> Result<Box<dyn Window>>;
    fn theme(&self) -> PlatformTheme;
    fn announce_for_accessibility(&self, text: &str);
    // …plus the behaviour helpers below
}

pub struct PlatformFactory;
impl PlatformFactory {
    pub fn create(platform: Platform) -> Box<dyn PlatformUi> { /* … */ }
}
```

`PlatformTheme` groups `PlatformColors`, `PlatformFonts`, `PlatformSpacing`, `PlatformShapes`
and `PlatformMotion`. The `keyboard`, `pointer` and `text_input` submodules translate the
platform's conventions (`platform_key_mapping`, `platform_pointer_behavior`,
`platform_text_input_behavior`, and the `KeyCode`/`PointerType`/`TextInputConfig` types).

---

## 7. The apps

### 7.1 `apps/example`

The whole application lives in the **library** (`src/lib.rs`), with a thin binary
(`src/main.rs`) so that the *same* code becomes a desktop binary, an Android `.so`, a browser
`.wasm` and an iOS `.a`. The crate types are declared in `Cargo.toml`:
`crate-type = ["rlib", "cdylib", "staticlib"]`, library name `example_app`.

**Entry points**

| Entry point | Target | Notes |
| --- | --- | --- |
| `run()` | desktop | `#[cfg(not(target_arch = "wasm32"))]`; parses CLI, boots, `run_egui` |
| `run_web()` | web | `#[wasm_bindgen(start)]`, `async`; finds `<canvas id="the_canvas_id">` |
| `android_main(app)` | Android | `#[no_mangle] extern "C"`; `install_app(app)` then boots and runs |

**CLI** (`build_cli()` / `parse_args()` → `CliOptions`): `--platform`, `--framework`, `--debug`
(`-d`), `--config` (`-c PATH`), `--server URL`. `--server` overrides `network.base_url` for a
single run; the built-in default already points at the bundled backend.

**Platform deps are `cfg`-gated in `Cargo.toml`**, so a desktop build never compiles the JNI or
`web-sys` code:

```toml
[target.'cfg(target_os = "android")'.dependencies]
winit = { workspace = true }
android-activity = { workspace = true }
platform-android = { path = "../../crates/platform/android" }

[target.'cfg(target_os = "linux")'.dependencies]
platform-linux = { path = "../../crates/platform/linux" }
# …and windows / macos / ios / wasm32
```

**The browser has no clock.** `wasm32-unknown-unknown` aborts on `Instant::now()` and
`SystemTime::now()`. Anything that needs time uses `web_time`, which is `std::time` on native
targets and `performance.now()`/`Date.now()` in the browser.

**The backend panel** (`src/backend.rs`) calls the bundled server without blocking the UI:
requests run on a worker thread and post results back over an `mpsc` channel. On the web — where
there are no raw sockets — it renders a placeholder instead of pretending to connect.

### 7.2 `apps/server`

A **library + binary**: the routing/HTTP logic is testable without binding a socket
(`server::Server`, `server::serve`), and `main.rs` is the `clap` binary.

```bash
cargo run --package server                          # 127.0.0.1:8080, demo users seeded
cargo run --package server -- --bind 0.0.0.0:8080    # reachable from an emulator / LAN
cargo run --package server -- --no-demo              # empty user store
```

Public surface: `Server::new(services)`, `Server::handle(&Request) -> Response`,
`Server::services()`, `serve(listener, server)`, `DEFAULT_LIMIT = 50`, `MAX_LIMIT = 200`. The
HTTP/1.1 wire code is hand-written (`src/http.rs`); the seam for a framework such as `axum` is
the routing table in `src/lib.rs`. The binary seeds two demo users
(`alice@example.com`, `bob@example.com`, password `password123`) unless `--no-demo` is given.

---

## 8. The wire contract and the bundled backend

**The contract is written once** in `crates/shared/src/api/mod.rs` and imported by *both* ends,
so the client and server cannot drift:

```rust
pub mod endpoints {
    pub const HEALTH: &str = "/api/health";
    pub const USERS: &str = "/api/users";
    pub const USERS_SEARCH: &str = "/api/users/search";
}
```

- **Server side:** `apps/server/src/lib.rs` maps the `endpoints` constants to handlers.
- **Client side:** `shared::services::HttpNetworkService` — a blocking HTTP/1.1 client built on
  `std::net`, `#[cfg(not(target_arch = "wasm32"))]`, speaking plain `http://`. It does **not**
  use `reqwest`, so it needs neither the `network` feature nor a TLS stack, and the Android /
  web builds stay TLS-free.

```rust
let client = HttpNetworkService::new("http://127.0.0.1:8080", Duration::from_millis(5_000));
let health = client.get_json(endpoints::HEALTH)?;
let created = client.post_json(endpoints::USERS, &serde_json::to_value(&request)?)?;
```

Run both halves together with `scripts/dev.sh` (it starts the backend, waits until the port
answers, runs the client pointed at it with `--server`, and stops the backend when the client
exits). On Android, forward the host port first: `adb reverse tcp:8080 tcp:8080`.

---

## 9. Feature flags and the `cfg` matrix

### 9.1 Cargo features

| Feature | Crate | Default | Effect |
| --- | --- | --- | --- |
| `network` | `shared` | **off** | Enables the `reqwest` (TLS) network client. Off ⇒ `StubNetworkService`, which returns `AppError::Unsupported` and cross-compiles everywhere with no C toolchain. |
| `egui` | `ui` | **on** | Turns shared component *logic* into real widgets. Off ⇒ the crate still builds (platform crates need its tokens/traits), but `show(…)` methods disappear. |
| `hot_reload` | `resources` | off | Reserved for hot-reloading resources. |

Verify honestly:

```bash
cargo check -p shared                       # stub network, no TLS
cargo build -p shared --features network    # real reqwest client
cargo check -p ui --no-default-features     # components without a renderer
```

### 9.2 `cfg` matrix (what compiles where)

| `cfg` | Applies to | What it gates |
| --- | --- | --- |
| `target_os = "android"` | Android | `android_main`, JNI, `platform-android`, winit android glue |
| `target_os = "linux" / "windows" / "macos" / "ios"` | that desktop/mobile OS | the matching `platform-*` crate and `init()` arm |
| `target_arch = "wasm32"` | web | `run_web`, `WEB_CANVAS_ID`, `web-sys`, `getrandom/js`, `uuid/js` |
| `not(target_arch = "wasm32")` | everything else | `run()`, `native_options()`, `HttpNetworkService` |
| `feature = "egui"` | `ui` | every component `show(…)` |
| `feature = "network"` | `shared` | the `reqwest` client |

Rule of thumb: **entry-point code that cannot exist on a target must be `#[cfg]`-gated, not
`if`-ed at runtime.** The template already models this for `run`/`run_web`/`android_main`.

---

## 10. Commands: build, run, test, lint

### 10.1 The three commands that matter

```bash
cargo check --workspace --all-targets                # 1. everything, every crate and target
cargo test  --workspace --all-targets                # 2. the tests
cargo clippy --workspace --all-targets --all-features -- -D warnings   # 3. lints = errors
```

Plus `cargo fmt --all`. CI runs the same, and also `cargo doc --workspace --no-deps` with
`RUSTDOCFLAGS=-D warnings`.

### 10.2 Short aliases (`cargo c`, `cargo t`, …)

Defined in `.cargo/config.toml`. None is named after a real Cargo subcommand on purpose (an
alias of the same name resolves to itself and recurses forever).

| Alias | Expands to |
| --- | --- |
| `cargo c` | `check --workspace --all-targets` |
| `cargo t` | `test --workspace --all-targets` |
| `cargo b` / `cargo br` | `build --workspace` / `--release` |
| `cargo f` | `fmt --all` |
| `cargo l` | `clippy --workspace --all-targets` |
| `cargo check-all` | `check --workspace --all-targets --all-features` |
| `cargo server` | `run --package server` |
| `cargo deps` | `tree --workspace --depth 2` |
| `cargo docs` | `doc --workspace --no-deps` |

### 10.3 Run and build scripts

```bash
cargo run --package example-app                      # desktop, default backend URL
cargo run --package example-app -- --debug           # -d for debug logging
cargo run --package example-app -- --server http://192.168.1.20:8080

scripts/build-all.sh --list        # what this machine can build
scripts/build-all.sh               # build all of them
scripts/build-all.sh android web   # just these

scripts/build-linux.sh [--release]
scripts/build-android.sh [--release] [--abi arm64-v8a] [--apk]
scripts/build-windows.sh [--release]     # MinGW-w64, or run on Windows
scripts/build-macos.sh [--release]       # macOS only
scripts/build-ios.sh [--simulator]       # macOS only
scripts/build-web.sh [--serve]           # needs trunk
scripts/dev.sh [--release] [--bind ADDR] [--server-only|--client-only]
```

Android without Gradle (just the `.so`):

```bash
source scripts/android-env.sh
cargo build --package example-app --target aarch64-linux-android --release
# -> target/aarch64-linux-android/release/libexample_app.so
```

---

## 11. Continuous integration

`.github/workflows/ci.yml` defines the gates. A change is "done" only when all of them pass.

| Job | Runner | Proves |
| --- | --- | --- |
| `checks` | ubuntu | `cargo fmt --check`, `clippy … -D warnings`, `check --all-features`, `test --all-features`, `doc -D warnings` |
| `backend` | ubuntu | builds `server`, starts it, and curls `/api/health`, `/api/users`, `/api/users/search?q=…` |
| `cross-compile` | ubuntu (matrix) | Android (4 ABIs) produce `libexample_app.so`; Web builds via Trunk |
| `android-apk` | ubuntu | JDK 17 + NDK + Gradle assemble a real debug `.apk` |
| `desktop` | ubuntu / macos / windows (matrix) | the native binary builds on each OS |
| `ios` | macos | the iOS static library compiles (signing is out of scope for CI) |

**Honest scope:** CI proves compilation, linting and tests on every target; it does **not**
prove a signed store build, and it does not run the app on a physical device. See the README's
"What is verified, and what is not" section for the project's own honesty statement.

---

## 12. Recipes: how to add things

Each recipe tells you **which file to touch and which invariants to respect** (§13).

### Recipe A — add a new screen / page

1. Add a variant to `Route` in `crates/shared/src/domain/mod.rs`.
2. Navigate to it with a `NavigationController` (`ui::navigation`) — `navigate(route, NavDirection::Forward)`.
3. Render it inside `ExampleApp::update` (or your own `eframe::App`) by matching on
   `controller.current()` and calling a `draw_*` helper.
4. Keep the drawing code in the app (`apps/example`) — the *route model* belongs in `shared`.

### Recipe B — add a shared component

1. Create `crates/ui/src/components/<name>.rs`.
2. **Pure logic first:** the struct, a `new(...)`, builder methods, and any sizing/state maths —
   all with **no `egui` types**, so it compiles with the `egui` feature off.
3. Add the rendering method behind `#[cfg(feature = "egui")] pub fn show(&self, ui: &mut egui::Ui, ctx: &UiContext)`.
4. Re-export it from `crates/ui/src/components/mod.rs` (`pub use <name>::…;`).
5. Add a unit test for the logic that runs without `egui`.

```rust
pub struct Badge { text: String, count: u32 }

impl Badge {
    pub fn new(text: impl Into<String>) -> Self { /* … */ }
    pub fn count(mut self, count: u32) -> Self { self.count = count; self }

    #[cfg(feature = "egui")]
    pub fn show(&self, ui: &mut egui::Ui, ctx: &UiContext) {
        let color = components::token(ctx, &ctx.color_tokens().primary);
        ui.colored_label(color, format!("{} ({})", self.text, self.count));
    }
}
```

### Recipe C — add a backend endpoint (contract first)

1. Add the path to `shared::api::endpoints` and the request/response types to
   `crates/shared/src/api/mod.rs` (both `serde::Serialize + Deserialize`).
2. Handle it in `apps/server/src/lib.rs` (match on the path, produce a `Response`).
3. Call it from the client with `HttpNetworkService::get_json`/`post_json`.
4. Both ends compile against the same type — rename a field and the other end stops compiling.
5. Add a smoke check similar to the `backend` CI job if the endpoint is user-facing.

### Recipe D — add a domain type + repository

1. Define it in `crates/shared/src/domain/mod.rs` and implement `Entity` (and
   `RepositoryEntity` if it will live in a repository).
2. Use `InMemoryRepository::<T>::new()` for storage, or add a method to an existing service
   trait in `crates/shared/src/services/mod.rs` and implement it in `services::defaults`.
3. Wire it into `ServiceContainer`/`AppServices` in `defaults::*` and in
   `services::factory::create_services`.
4. Prefer returning `shared::errors::Result<T>` (`AppError`) over `anyhow` inside `shared`.

### Recipe E — add a platform capability

1. If it is a *contract* (a method every platform answers): add it to `PlatformUi` /
   `Window` in `crates/ui/src/platform/mod.rs` and implement the default in
   `crates/ui/src/platform/*.rs`.
2. If it is *real OS work* (JNI, `Win32`, `uname`, `web-sys`, haptics): implement it in the
   matching `crates/platform/<os>/src/lib.rs` behind `#[cfg(target_os = "…")]`.
3. Expose a small imperative function (like `platform_android::device_info()`) and call it from
   the app's `init_platform`/entry point, **gated by `#[cfg]`**.

### Recipe F — change the theme

- Change tokens programmatically with `ThemeBuilder` (`primary_color`, `secondary_color`,
  `font_family`, `base_font_size`, `spacing_scale`, `border_radius_scale`, `motion_speed`).
- Or pick a built-in: `material3()`, `ios()`, `material_you()`, `high_contrast()`, `minimal()`.
- At runtime, `ThemeManager` updates a live theme and `generate_css()` exports it for the web.
- The user-facing theme choice lives in `AppConfig.ui.theme.mode` (`ThemeMode`).

### Recipe G — add a localized string

1. Add the key to `resources/strings/strings_en.json` (and `strings_es.json`).
2. Read it with `ResourceManager::global().get_string("key", Some("en"))`, or
   `get_string_formatted(…)` for placeholders.
3. `ResourceManager::load_all()` / `reload()` refresh the cache.

### Recipe H — point the client at a different backend

- One run: `cargo run --package example-app -- --server http://host:port` (also `--server` for
  `dev.sh`).
- Permanently: change `network.base_url` in the config written by `AppConfig::save()`.
- On Android, `adb reverse tcp:8080 tcp:8080` first (device loopback ≠ host loopback).

### Recipe I — call the backend from the UI without freezing it

- Follow `apps/example/src/backend.rs`: spawn a worker thread, perform the blocking
  `HttpNetworkService` call there, and send the result back over an `mpsc` channel that
  `update()` drains each frame. Never call the blocking client on the egui thread.
- On the web, do not ship a fake client; render a placeholder (there are no raw sockets), or use
  `web_sys::fetch` on a `wasm32` code path.

### Recipe J — rename the template to your own app

1. Rename the `[lib] name` in `apps/example/Cargo.toml` **and** update the two Android
   identifiers that must match it (`AndroidManifest.xml`'s `android.app.lib_name` and Gradle's
   `rustLibName`).
2. Change `WINDOW_TITLE` in `apps/example/src/lib.rs`.
3. Update `[workspace.package]` metadata (`repository`, `homepage`, `authors`, `description`) in
   the root `Cargo.toml`.
4. Update `android-app`'s application id and `resources/` strings.

---

## 13. Invariants a change must not break

These are the rules that keep the workspace cross-compiling and the README's claims true.

1. **One-way dependencies.** `shared` depends on nothing internal; `ui`/`resources`/
   `build-config` depend only on `shared`; `platform/*` depend on `shared` + `ui`;
   `apps/server` depends on `shared` only. **Never** make `shared` or `ui` depend on a platform
   crate — that creates a cycle, and `cargo check --workspace` on a machine with no SDKs must
   keep working.
2. **`shared` and the pure halves of `ui` contain no `egui` and no OS types.**
3. **Every component's pure logic compiles with `egui` off.** `cargo check -p ui
   --no-default-features` must stay green.
4. **The wire contract lives only in `shared::api`.** Both ends import it; never duplicate DTOs.
5. **Entry points that cannot exist on a target are `#[cfg]`-gated**, not runtime-branched.
6. **Time goes through `web_time`**, never `std::time::Instant::now()` / `SystemTime::now()`
   directly, so the wasm target does not abort.
7. **The Android library name is load-bearing:** it must match in `Cargo.toml`, the manifest and
   Gradle.
8. **Clippy is `-D warnings`** with `all`, `pedantic`, `nursery` and `cargo` at warn (workspace
   lints). New code must be lint-clean.
9. **`Cargo.lock` stays committed**, and `panic = "abort"` stays out of the release profile
   (it would make `cargo test --release` unusable).
10. **The `network` feature stays off by default** and `HttpNetworkService` stays `reqwest`-free,
    so Android/web builds keep cross-compiling without a TLS/C toolchain.

---

## 14. Troubleshooting and gotchas

| Symptom | Cause / fix |
| --- | --- |
| `web-sys` "no method named `class_list`/`style`/`head`" | `web-sys` generates a method only when its feature is on. Add the feature to the `web-sys` list in the root `Cargo.toml`. |
| Android build fails to find the linker/`cc` | NDK r23+ removed the bare `<arch>-linux-android-clang` symlinks. `source scripts/android-env.sh` (exports `CARGO_TARGET_*_LINKER`, `CC_*`, `CXX_*`, `AR_*`). |
| A `platform-windows` (or `platform-macos`) error appears only when you target that OS, while `cargo check --workspace` on Linux is green | Desktop code sits behind `#[cfg(target_os = "windows")]`, so a Linux host never type-checks it. Cross-**check** without a linker: `rustup target add x86_64-pc-windows-msvc` then `cargo check --workspace --target x86_64-pc-windows-msvc`. Producing a real `.exe` (linking) still needs MinGW-w64 via `scripts/build-windows.sh`, or a Windows host. |
| `android_main` code is not type-checked in the editor | rust-analyzer analyzes for the host. Set `rust-analyzer.cargo.target = "aarch64-linux-android"` in your **uncommitted** `.vscode/settings.json`. |
| Two incompatible `JNIEnv` types | Two `jni` versions in the graph. `jni = "0.22"` must match what `android-activity` 0.6 resolves to. |
| `uuid`/`getrandom` fails on wasm | Entropy needs the `js` backend: `getrandom { features = ["js"] }` and `uuid { features = ["v4","serde","js"] }` for the wasm graph (already declared in `apps/example/Cargo.toml`). |
| Freeze/panic on time in the browser | Use `web_time`, not `std::time`, on `wasm32`. |
| `cargo <alias>` recurses forever | An alias may not share a name with a real Cargo subcommand. Keep using `f`/`l`, not `fmt`/`clippy`. |
| Backend unreachable from an emulator | `127.0.0.1` is the device. `adb reverse tcp:8080 tcp:8080`, or bind `--bind 0.0.0.0:8080`. |
| `ui` methods missing under `--no-default-features` | Expected: `show(…)` is behind the `egui` feature. |
| `hooks`/`utils` look empty | They exist as files but are **not declared** in `ui/src/lib.rs` — they are intentionally out of the egui stack for now. |

---

## 15. Notes for AI coding assistants

Concrete rules that make generated code correct in this repository:

1. **Locate first, then write.** Use the tree in §4 and the map in §5. If a symbol is not there,
   it does not exist — create it in the crate that §3.1 says owns it.
2. **Respect the layers.** Business logic → `shared`. Reusable widget → `ui::components` with a
   pure-logic half and a `cfg(egui)` `show`. OS call → `crates/platform/*`. App wiring →
   `apps/example`. Endpoint → `shared::api` + `apps/server/src/lib.rs`.
3. **Keep the `egui`-off build green.** No `egui` type may appear outside `#[cfg(feature = "egui")]`
   blocks in `ui`.
4. **Prefer `shared::errors::Result`/`AppError` in `shared`;** `anyhow` is fine at the app edge.
5. **Gate per-target code with `#[cfg]`,** mirroring the existing `run`/`run_web`/`android_main`
   pattern. Never call an Android/JNI/`web-sys` API unguarded.
6. **Do not add a second way to do an existing thing** (a second HTTP client, a second theme
   system, a second router). Extend the existing seam.
7. **Run the gates** in §10 before declaring a task done, and state honestly what you ran and
   what you did not (matching the project's own style).
8. **Comments explain *why*.** The codebase's comments document the reasoning (why a `cfg`, why
   `jlink` matters, why an alias is short) — match that, not "what" comments.

---

## 16. References and contact

- [README.md](README.md) — English quick start and project overview.
- [README.fa.md](README.fa.md) — نسخهٔ فارسی.
- Wire contract: `crates/shared/src/api/mod.rs`
- Component library: `crates/ui/src/components/`
- Platform traits: `crates/ui/src/platform/mod.rs`
- Entry points: `apps/example/src/lib.rs`
- Backend: `apps/server/src/lib.rs`
- CI gates: `.github/workflows/ci.yml`

**Author:** Sina Khanzadeh — <khanzadeh.1377@gmail.com> — [sina-khanzadeh.ir](https://sina-khanzadeh.ir)
**Instagram / Telegram / Discord:** `programmer_1998`
**Licence:** MIT OR Apache-2.0 — see `Cargo.toml` (`[workspace.package] license`).

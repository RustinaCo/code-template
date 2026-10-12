# قالب Rustinaz — راهنمای کامل

**مرجع توسعه‌دهنده و هوش مصنوعی برای ورک‌اسپیس `Rustinaz/code-template`.**

این سند مکمل [README.fa.md](README.fa.md) است. README توضیح می‌دهد قالب *چیست* و چطور اجرا
می‌شود؛ **این راهنما توضیح می‌دهد قالب چطور ساخته شده و چطور باید داخلش کد نوشت** — قرارداد
crate‌به‌crate، قواعدی که یک تغییر نباید بشکند، و دستورهای آمادهٔ کار.

هدف این است که **هم یک آدم و هم یک دستیار هوش مصنوعی بتواند بدون حدس‌زدن از روی آن کار کند**:
هر مسیر، هر crate و هر API عمومی که اینجا آمده، واقعاً در همین مخزن وجود دارد.

- 🇬🇧 **English guide:** [TEMPLATE.md](TEMPLATE.md)
- 🇮🇷 **شروع سریع (فارسی):** [README.fa.md](README.fa.md)
- 🇬🇧 **Getting started (English):** [README.md](README.md)

> نویسنده: **سینا خانزاده** · <khanzadeh.1377@gmail.com> · [sina-khanzadeh.ir](https://sina-khanzadeh.ir)
> اینستاگرام / تلگرام / دیسکورد: **programmer_1998**
> مجوز: **MIT OR Apache-2.0**

---

## فهرست مطالب

1. [چطور از این سند استفاده کنیم](#۱-چطور-از-این-سند-استفاده-کنیم)
2. [قالب چیست](#۲-قالب-چیست)
3. [معماری در یک نگاه](#۳-معماری-در-یک-نگاه)
4. [درخت کامل فایل‌ها](#۴-درخت-کامل-فایلها)
5. [crateها، یکی‌یکی](#۵-crateها-یکییکی)
6. [لایهٔ UI با جزئیات](#۶-لایهٔ-ui-با-جزئیات)
7. [اپلیکیشن‌ها](#۷-اپلیکیشنها)
8. [قرارداد سیم و بک‌اندِ هم‌بسته](#۸-قرارداد-سیم-و-بکاندِ-همبسته)
9. [فیچرها و ماتریس `cfg`](#۹-فیچرها-و-ماتریس-cfg)
10. [دستورها: بیلد، اجرا، تست، لینت](#۱۰-دستورها-بیلد-اجرا-تست-لینت)
11. [یکپارچه‌سازی مداوم (CI)](#۱۱-یکپارچهسازی-مداوم-ci)
12. [دستورهای آماده: چطور چیزی اضافه کنیم](#۱۲-دستورهای-آماده-چطور-چیزی-اضافه-کنیم)
13. [قواعدی که یک تغییر نباید بشکند](#۱۳-قواعدی-که-یک-تغییر-نباید-بشکند)
14. [عیب‌یابی و تله‌ها](#۱۴-عیبیابی-و-تلهها)
15. [نکات برای دستیارهای هوش مصنوعی](#۱۵-نکات-برای-دستیارهای-هوش-مصنوعی)
16. [منابع و راه ارتباطی](#۱۶-منابع-و-راه-ارتباطی)

---

## ۱. چطور از این سند استفاده کنیم

**اگر توسعه‌دهندهٔ انسانی هستی:** §۲ تا §۳ را بخوان تا مدل ذهنی را بگیری، §۵ را به‌عنوان نقشه
مرور کن، و هر وقت چیزی اضافه می‌کنی سراغ §۱۲ (دستورهای آماده) برو. §۱۴ تله‌های رایج را جمع
کرده است.

**اگر دستیار هوش مصنوعی هستی:** پیش از نوشتن هر کدی،

1. §۳ را بخوان (گراف وابستگی — تعیین می‌کند کد جدید کجا باید باشد)،
2. §۵ را بخوان (کدام crate مسئول چیست)،
3. دستور آمادهٔ متناظر را در §۱۲ بردار،
4. §۱۳ (قواعد) و §۹ (فیچرها / `cfg`) را رعایت کن،
5. برای اثبات تغییر، دستورهای §۱۰ را اجرا کن.

Crate، ماژول یا API **از خودت نساز**. هرچه وجود دارد در §۴ و §۵ فهرست شده. اگر چیزی آنجا
نبود، §۱۲ می‌گوید جایش کجاست.

---

## ۲. قالب چیست

Rustinaz یک **قالب اپلیکیشن کراس‌پلتفرم با رویکرد «اول راست»** است. یک ورک‌اسپیس این‌ها را
می‌سازد:

| هدف | خروجی | نقطهٔ ورود |
| --- | --- | --- |
| لینوکس / ویندوز / مک | یک باینری نیتیو دسکتاپ | `apps/example/src/main.rs` → `example_app::run()` |
| اندروید | یک `.so` که `android.app.NativeActivity` لود می‌کند | `apps/example/src/lib.rs` → `android_main()` |
| iOS | یک کتابخانهٔ استاتیک برای لینک‌شدن در اپ Xcode | `apps/example/src/lib.rs` (نوع `staticlib` در crate) |
| وب | یک `.wasm` + `index.html` که Trunk بسته‌بندی می‌کند | `apps/example/src/lib.rs` → `run_web()` |
| (هرکدام) | بک‌اند JSON هم‌بسته | `apps/server/src/main.rs` |

شرط قالب ساده است: **UI را یک‌بار با egui بنویس و همان درخت ویجت‌ها را به هر هدف بفرست.**
فقط چیزهایی که واقعاً بین سیستم‌عامل‌ها تفاوت دارند — ساخت پنجره، تم پیش‌فرض، اطلاعات دستگاه،
اعلام‌های دسترس‌پذیری، JNI — در کد مخصوص هر پلتفرم می‌مانند.

هیچ **کد اپلیکیشنی** به جاوا، کاتلین، Swift یا HTML/CSS وجود ندارد. پروژهٔ اندروید
(`android-app/`) فقط اسکریپت‌های بیلد Gradle (Kotlin DSL) و فایل‌های منبع XML (manifest، تم،
آیکون) دارد — APK صفر فایل `.class` دارد و همهٔ پیکسل‌ها را `.so` راست می‌کشد.

---

## ۳. معماری در یک نگاه

### ۳.۱ گراف وابستگی (مرجع قطعی)

این‌ها **واقعی** هستند، گرفته‌شده از `cargo metadata`. جهت کاملاً یک‌طرفه است؛ هیچ‌چیز پایین‌تر
به چیزی بالاتر وابسته نمی‌شود.

```
                       ┌──────────────┐
                       │   shared     │   دامنه، کانفیگ، خطاها، api، سرویس‌ها (بدون UI و OS)
                       └──────┬───────┘
              ┌───────────────┼────────────────┬─────────────────┐
              │               │                │                 │
       ┌──────▼──────┐ ┌──────▼──────┐  ┌──────▼───────┐  ┌──────▼────────┐
       │     ui      │ │  resources  │  │ build-config │  │ platform/*    │
       │ (egui،      │ │ (رشته‌ها،   │  │ (نسخه،       │  │ (linux، …)    │
       │  توکن‌ها،   │ │  رنگ‌ها)    │  │  امضا)       │  │               │
       │  کامپوننت‌ها)│ └──────┬──────┘  └──────────────┘  └──────┬────────┘
       └──────┬──────┘        │                                 │
              │               │     ui هم وابستگی هر            │
              │               │     crate از platform/* است  ───┘
              └───────────────┴───────────────┬─────────────────┘
                                              │
                                    ┌─────────▼──────────┐        ┌──────────┐
                                    │  apps/example      │        │apps/server│
                                    │  (خود اپلیکیشن)     │        │ (بک‌اند)  │
                                    └────────────────────┘        └─────┬────┘
                                                                        │
                                                                 تنها shared
```

بخوانید:

- **`shared` به هیچ چیزی داخل پروژه وابسته نیست.** تایپ‌های دامنه، کانفیگ، خطاها، قرارداد سیم
  (`api`) و traitهای سرویس + پیاده‌سازی‌های in-memory.
- **`ui` به `shared` وابسته است.** توکن‌های تم، ریاضیات layout، ناوبری، کتابخانهٔ کامپوننت و
  traitهای `PlatformUi`/`Window`.
- **`platform/*` به `shared` و `ui` وابسته‌اند.** روی traitهای `ui` کار سیستم‌عامل را اضافه
  می‌کنند.
- **`resources` و `build-config` به `shared` وابسته‌اند.**
- **`apps/example` به `shared`، `ui`، `resources` و هر شش crate `platform/*` وابسته است**
  (هر کدام با `#[cfg(target_os = …)]` مهار شده‌اند؛ §۷.۱).
- **`apps/server` فقط به `shared` وابسته است** — بک‌اند هیچ‌وقت `ui` یا crate پلتفرم را وارد
  باینری‌اش نمی‌کند، پس صفر کد GUI حمل می‌کند.

> **دو چیز «platform» نام دارند.** `ui` پیاده‌سازی‌های `PlatformUi` خودش را زیر
> `crates/ui/src/platform/` دارد، در حالی که crateهای رو به سیستم‌عامل در `crates/platform/*`
> هستند. فقط این دومی‌ها عضو مستقل ورک‌اسپیس‌اند و به `ui` وابسته‌اند — نه برعکس.

### ۳.۲ یک اجرا چطور سیم‌کشی می‌شود

دسکتاپ و اندروید از همان ترتیب راه‌اندازی در `apps/example/src/lib.rs` عبور می‌کنند:

```
main() / android_main() / run_web()
        │
        ▼
  bootstrap(&CliOptions)
        │  ۱. AppConfig::load()  (یا --config PATH)
        │  ۲. اعمال override‌های --platform / --server
        │  ۳. shared::init_blocking(config)        → tracing + globalها
        │  ۴. platform_*::init()                   → آماده‌سازی OS (هر هدف)
        │  ۵. ResourceManager::init_global(config) → رشته‌ها/رنگ‌ها/تم‌ها
        │  ۶. UiBuilder::new(config)
        │        .platform(platform)               → تصحیح میزبان → دستگاه
        │        .framework(UiFramework::Egui)
        │        .build()                          → UiContext
        ▼
  eframe::run_native(...)  (دسکتاپ)   |   WebRunner::start(...)  (وب)
        │
        ▼
  ExampleApp::update(&egui::Context)   → رسم با egui، خواندن از UiContext
```

`UiContext` (در `crates/ui`) همان شیئی است که UI از آن می‌خواند: `AppConfig` مشترک،
`DeviceInfo` تشخیص‌داده‌شده، `ThemeTokens` حل‌شده و `UiFramework` فعلی.

---

## ۴. درخت کامل فایل‌ها

هر چیزی که پایین آمده در مخزن وجود دارد (خروجی‌های تولیدشده مثل `target/`،
`apps/example/dist/` و `android-app/build/` حذف شده‌اند).

```
code-template/
├── Cargo.toml                      ورک‌اسپیس: ۱۲ عضو، نسخهٔ وابستگی‌ها، پروفایل‌ها
├── Cargo.lock                      عمداً کامیت شده: بیلد اول قابل‌تکرار
├── rust-toolchain.toml             قفل stable + rustfmt + clippy (ادیتور و CI یکی)
├── .editorconfig                   UTF-8، LF، تودرتوی ۴ (۲ برای TOML/YAML)
├── .gitignore
├── .cargo/
│   └── config.toml                 تنظیمات شبکه + aliasهای کوتاه (بدون linker ثابت)
├── .github/
│   ├── dependabot.yml
│   └── workflows/ci.yml            شش دروازهٔ CI
├── .vscode/                        ران‌کانفیگ، taskها، تنظیم rust-analyzer، افزونه‌ها
│
├── apps/
│   ├── example/                    ← اپی که واقعاً ویرایش می‌کنی
│   │   ├── Cargo.toml              crate-type = rlib, cdylib, staticlib؛ وابستگی per-target
│   │   ├── Trunk.toml              کانفیگ بیلد وب
│   │   ├── index.html              همان <canvas> که eframe در آن mount می‌شود (id = the_canvas_id)
│   │   └── src/
│   │       ├── lib.rs              run()، run_web()، android_main()، bootstrap()، خود UI
│   │       ├── main.rs             باینری نازک دسکتاپ → example_app::run()
│   │       └── backend.rs          پنل Backend کلاینت (نیتیو) / جای‌گزین (وب)
│   └── server/                     بک‌اند هم‌بسته
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs              روتینگ: نگاشت مسیرها به هندلرها؛ serve()
│           ├── http.rs             پارس/نوشتن دستی HTTP/1.1
│           └── main.rs             باینری clap (--bind، --no-demo)، seed دمو
│
├── crates/
│   ├── shared/                     منطق کسب‌وکار (بدون UI، بدون OS)
│   │   ├── Cargo.toml              فیچر network اینجاست
│   │   └── src/
│   │       ├── lib.rs              init()/init_blocking()، re-exportها، alias شبکه
│   │       ├── domain/mod.rs       User، AppSettings، DeviceInfo، Platform، ThemeTokens، Route، …
│   │       ├── config/mod.rs       AppConfig، ConfigBuilder، ThemeMode، FeatureFlags
│   │       ├── errors/mod.rs       AppError + زیرخطاها، ErrorContext
│   │       ├── api/mod.rs          قرارداد سیم: endpointها + تایپ‌های درخواست/پاسخ
│   │       └── services/
│   │           ├── mod.rs          traitهای سرویس، پیاده‌سازی‌های in-memory، ServiceContainer، factory
│   │           └── http.rs         HttpNetworkService: کلاینت بلوکینگ HTTP/1.1 فقط با std
│   ├── ui/                         UI مشترک (کتابخانهٔ ویجت اینجاست)
│   │   ├── Cargo.toml              فیچر egui اینجاست (پیش‌فرض روشن)
│   │   └── src/
│   │       ├── lib.rs              UiContext، UiBuilder، UiFramework، هلپرهای responsive
│   │       ├── theme/mod.rs        ThemeManager، ThemeBuilder، تم‌های آماده، خروجی CSS
│   │       ├── layout/mod.rs       LayoutCalculator، FlexLayout، GridLayout، builderها
│   │       ├── navigation/mod.rs   NavigationController، deep link، کانفیگ bottom-nav/drawer
│   │       ├── components/         کتابخانهٔ کامپوننت
│   │       │   ├── mod.rs          trait Component، ComponentProps، هلپرهای رنگ، re-export
│   │       │   ├── app_bar.rs      AppBar (+ BACK_ACTION)
│   │       │   ├── button.rs       Button، ButtonStyle، MIN_TOUCH_TARGET
│   │       │   ├── card.rs         Card
│   │       │   ├── dialog.rs       Dialog، DialogState
│   │       │   ├── navigation_bar.rs  NavItem، NavigationBar
│   │       │   ├── scaffold.rs     Scaffold، ScaffoldLayout، Slot
│   │       │   └── text_field.rs   TextField، TextFieldState
│   │       ├── platform/
│   │       │   ├── mod.rs          traitهای PlatformUi + Window، PlatformTheme، PlatformFactory،
│   │       │   │                   رفتارهای keyboard/pointer/text-input
│   │       │   ├── linux.rs        LinuxUi / LinuxWindow      (پیاده‌سازی‌های درون‌crate)
│   │       │   ├── windows.rs      WindowsUi / WindowsWindow
│   │       │   ├── macos.rs        MacosUi / MacosWindow
│   │       │   ├── android.rs      AndroidUi / AndroidWindow
│   │       │   ├── ios.rs          IosUi / IosWindow
│   │       │   └── web.rs          WebUi / WebWindow
│   │       ├── hooks/mod.rs        هست ولی declare نشده (هنوز بخشی از egui نیست)
│   │       └── utils/mod.rs        هست ولی declare نشده (هنوز بخشی از egui نیست)
│   ├── resources/                  رشته‌ها، رنگ‌ها، تم‌ها، assetها (بارگذاری در زمان اجرا)
│   │   ├── Cargo.toml
│   │   └── src/lib.rs              ResourceManager، ResourceBuilder، StringMap، ColorMap
│   ├── build-config/               متادیتای نسخه / هدف / امضا
│   │   ├── Cargo.toml
│   │   └── src/lib.rs              BuildTimeConfig، VersionInfo، platform_build_config، امضا
│   └── platform/                   چسب سیستم‌عامل (یک crate به‌ازای هر هدف)
│       ├── linux/     src/lib.rs   platform_info()، init()
│       ├── windows/   src/lib.rs   init()، هلپرهای پنجره/تم
│       ├── macos/     src/lib.rs   init()، هلپرهای پنجره/تم
│       ├── android/   src/lib.rs   AndroidApp، install_app/current_app، device_info()، JNI، init()
│       ├── ios/       src/lib.rs   IosInfo، device_info()، init()، هپتیک
│       └── web/       src/lib.rs   user agent، viewport، تشخیص لمس
│
├── android-app/                    پروژهٔ بسته‌بندی Android Studio (بدون سورس اپ)
│   ├── settings.gradle.kts
│   ├── build.gradle.kts
│   ├── gradle.properties
│   ├── gradlew  /  gradlew.bat
│   ├── gradle/wrapper/…            jar + properties ریپر
│   └── app/
│       ├── build.gradle.kts        Cargo را per-ABI اجرا و سپس .so را بسته‌بندی می‌کند
│       ├── proguard-rules.pro
│       └── src/main/
│           ├── AndroidManifest.xml به android.app.NativeActivity اشاره می‌کند
│           └── res/…               رشته‌ها، تم، آیکون لانچر
│
├── resources/                      فایل‌های داده‌ی مشترک
│   ├── colors/colors.json
│   └── strings/strings_en.json  ·  strings_es.json
│
└── scripts/                        یک اسکریپت per-platform، به‌علاوه حلقهٔ dev
    ├── android-env.sh              linker/CC یادNDK را برای Cargo export می‌کند (source کن)
    ├── build-all.sh                هر هدفی که این ماشین می‌تواند را بساز
    ├── build-linux.sh  build-windows.sh  build-macos.sh  build-ios.sh
    ├── build-android.sh  build-web.sh
    └── dev.sh                      بک‌اند را بالا می‌آورد، صبر می‌کند پورت جواب دهد، کلاینت را اجرا می‌کند
```

---

## ۵. crateها، یکی‌یکی

### ۵.۱ `shared`

crate منطق کسب‌وکار. **بدون `egui`، بدون تایپ سیستم‌عامل.** از ریشهٔ crate
(`crates/shared/src/lib.rs`) دوباره export شده تا کد پایین‌دست بتواند `use shared::…` بنویسد.

| ماژول | مسئول | اقلام عمومی کلیدی |
| --- | --- | --- |
| `shared::domain` | مدل داده‌ی اصلی | `EntityId` (= `Uuid`)، `Timestamp`، traitهای `Entity`/`RepositoryEntity`، `User`، `UserPreferences`، `AppSettings`، `AccessibilitySettings`، `WindowState`، `WindowPosition`، `DeviceInfo`، `Platform`، `FormFactor`، `ScreenSize`، `Orientation`، `SafeArea`، `Route`، `ThemePreference`، `ThemeTokens`، `ColorTokens`/`ColorRole`/`ColorValue`، `TypographyTokens`، `SpacingTokens`، `ShapeTokens`، `MotionTokens`، `ElevationTokens`، `Shadow`، `BuildConfig`، `SigningConfig` |
| `shared::config` | کانفیگ زمان اجرا | `AppConfig`، `ConfigBuilder`، `AppInfo`، `PlatformConfig`، `UiConfig`، `ThemeConfig`، `ThemeMode`، `ResponsiveBreakpoints`، `NetworkConfig`، `StorageConfig`، `LoggingConfig`، `LogLevel`/`LogFormat`/`LogOutput`، `FeatureFlags`. `AppConfig::load()`، `::save()`، `::device_info()`، `::theme_tokens()`، `::is_feature_enabled()` |
| `shared::errors` | تنها تایپ خطا | `AppError` (می‌پیچد `ConfigError`، `NetworkError`، `StorageError`، `AuthError`، `ValidationError`، `PlatformError`، `UiError`، `SerializationError` و همچنین `Io`، `Http`، `Json`، `NotFound`، `Timeout`، `Unsupported`، `Multiple`)، `Result<T>`، `ErrorContext`/`ErrorContextExt` |
| `shared::api` | **قرارداد سیم** | `endpoints::{HEALTH, USERS, USERS_SEARCH}`، `HealthResponse`، `ErrorResponse`، `RegisterRequest`، `UserList`، … |
| `shared::services` | traitهای سرویس + پیاده‌سازی in-memory | traitهای `AuthService`، `UserService`، `SettingsService`، `NetworkService`، `StorageService`، `PlatformService`، `AnalyticsService`، `NotificationService`؛ `InMemoryRepository<T>`؛ `ServiceContainer`، `AppServices`، `PlatformServices`؛ `services::defaults::*`؛ `services::factory::create_services()`؛ `HttpNetworkService` |

**راه‌اندازی.** `shared::init_blocking(config) -> Result<Arc<AppConfig>>` کانفیگ را لود می‌کند،
`tracing` را نصب می‌کند و کانفیگ مشترک را برمی‌گرداند. `shared::init()` همان است ولی `async`،
برای فراخوانی از داخل یک runtime. از `init_blocking` در `main`، `android_main` یا هر نقطهٔ
ورود `extern "C"` استفاده کن.

**ترفند alias شبکه.** `shared` یک نام را دوباره export می‌کند، `DefaultNetworkService`، که
یا کلاینت مبتنی بر `reqwest` است (فیچر `network`) یا `StubNetworkService` (پیش‌فرض). کد
فراخوان در هر دو حالت یکسان است و در حالت stub در زمان اجرا فقط `AppError::Unsupported`
می‌گیرد. `HttpNetworkService` جداگانه است و همیشه بیرون وب در دسترس است.

### ۵.۲ `ui`

crate UI مشترک. دو نیمه که با فیچر `egui` (پیش‌فرض روشن) از هم جدا می‌شوند:

- **منطق خالص** (همیشه کامپایل می‌شود): توکن‌ها، ریاضیات layout، builderها، state و اندازه‌گیری کامپوننت.
- **رندر** (`#[cfg(feature = "egui")]`): متدهای `show(&mut egui::Ui, …)`.

| ماژول | اقلام عمومی کلیدی |
| --- | --- |
| `ui` (ریشه) | `UiFramework`، `UiContext`، `UiBuilder`، ماژول `ui::responsive` |
| `ui::theme` | `ThemeManager`، `ThemeBuilder`، `ThemeChangeListener`؛ تم‌های آماده `material3()`، `ios()`، `material_you()`، `high_contrast()`، `minimal()`؛ `generate_css()` |
| `ui::layout` | `LayoutConstraints`، `LayoutCalculator`، `FlexLayout`، `GridLayout`، `LayoutBuilder`، `FlexLayoutBuilder`، `GridLayoutBuilder`، `LayoutDirection`، `FlexDirection`، `JustifyContent`، `AlignItems`، `AlignContent`، `FlexWrap`، `GridAutoFlow` |
| `ui::navigation` | `NavigationController`، `NavEntry`، `NavDirection`، `DeepLinkHandler`، `NavigationBuilder`، `NavigationListener`، `BottomNavConfig`/`BottomNavItem`، `NavigationDrawerConfig`/`DrawerHeader`/`DrawerItem`/`DrawerFooter` |
| `ui::components` | §۵.۲.۱ را ببین |
| `ui::platform` | `PlatformUi`، `Window`، `WindowConfig`، `PlatformTheme`، `PlatformColors`، `PlatformFonts`، `PlatformSpacing`، `PlatformShapes`، `PlatformMotion`، `PlatformFactory`، و هلپرهای `keyboard`/`pointer`/`text_input` |

#### ۵.۲.۱ کتابخانهٔ کامپوننت (`ui::components`)

هر کامپوننت همان **شکل دو‌تکه** را دارد: منطق خالص (builderها، state، اندازه‌گیری) که با خاموشی
`egui` هم کامپایل می‌شود، و متد `show(…)` که پشت `#[cfg(feature = "egui")]` است.

| کامپوننت | ساخت با | رندر با |
| --- | --- | --- |
| `AppBar` | `AppBar::new(title)` + `.subtitle()`، `.action()`، `.back_enabled()`، `.center_title()`، `.elevated()` | `show(ui, …)` (فیچر `egui`) |
| `Button` | `Button::new(label)` + `.style(ButtonStyle)`، `.size(ComponentSize)`، `.enabled()`، `.icon(char)`، `.full_width()` | یک `ButtonResponse` برمی‌گرداند |
| `Card` | `Card::new(title)` + `.subtitle()`، `.elevation(f32)`، `.padding_multiplier()`، `.clickable()` | `show(ui, ctx) -> ButtonResponse` / `show_with(…)` |
| `Dialog` | `Dialog::new(title)` + `.body()`، `.dismissable()`، `.visible()`، `.add_action()` | `show(…)`، با یک `DialogState` |
| `NavigationBar` | `NavigationBar::new(Vec<NavItem>)` + `.items()`، `.select()`، `.select_next()`، `.show_labels()` | `show(…)` |
| `Scaffold` | `ScaffoldLayout::new(app_bar, navigation_bar, body)` از `Slot`ها | ریاضیات layout؛ `ScaffoldResponse` |
| `TextField` | `TextField::new()`؛ state با `TextFieldState::new(placeholder)` + `.with_max_length()`، `.with_password()`، `.with_disabled()`، `.set_value()`، `.take_changed()` | `show(…)` |

همچنین عمومی: trait `Component`، `ComponentProps`، `ComponentVariant`، `ComponentSize`،
`ComponentState`، `AccessibilityRole`، `ButtonResponse`، `MIN_TOUCH_TARGET`، `BACK_ACTION` و
هلپرهای رنگ `role_value`، `value`، `role`، `token`، `with_alpha`، `state_fill`.

### ۵.۳ `platform/*` (چسب سیستم‌عامل)

یک crate برای هر هدف. هرکدام به `shared` و `ui` وابسته‌اند و یک API کوچک و امری به‌علاوه یک
پیاده‌سازی `PlatformUi` (هرجا سیستم‌عامل حرفی داشته باشد: پنجره، تم، دسترس‌پذیری) ارائه می‌دهند.

| crate | سطح عمومی | از کجا استفاده می‌شود |
| --- | --- | --- |
| `platform-linux` | `platform_info()`، `PlatformInfo`، `init()` | `init_platform()` در `apps/example` |
| `platform-windows` | `init()`، هلپرهای پنجره/تم | `apps/example` |
| `platform-macos` | `init()`، هلپرهای پنجره/تم | `apps/example` |
| `platform-android` | `AndroidApp`، `install_app()`، `current_app()`، `device_info()`، `AndroidInfo`، `init()`، `material3_theme()`، `AndroidPlatformUi`، `AndroidWindow` | `android_main()` در `apps/example` |
| `platform-ios` | `IosInfo`، `device_info()`، `init()`، هپتیک | `apps/example` |
| `platform-web` | تشخیص user-agent، viewport و لمس | `apps/example` (از مسیر هدف وب) |

> crate `ui` **همچنین** پیاده‌سازی‌های `PlatformUi` مخصوص هر پلتفرم را زیر
> `crates/ui/src/platform/*.rs` دارد. این تفکیک عمدی است: **`ui::platform` انتزاع‌ها و
> پیاده‌سازی‌های پیش‌فرض را تعریف می‌کند**؛ **crateهای `crates/platform/*` کار واقعی سیستم‌عامل
> را انجام می‌دهند** (JNI، پنجرهٔ نیتیو، `uname`، `Win32`، `web-sys`). وقتی سیستم‌عامل لازم است
> `crates/platform/*` را ویرایش کن؛ وقتی شکل/قرارداد لازم است `crates/ui/src/platform/mod.rs` را.

### ۵.۴ `resources`

بارگذاری منابع در زمان اجرا: رشته‌ها، رنگ‌ها، تم‌ها، assetها و فونت‌ها، از پوشهٔ `resources/`
و کش‌شده در سطح کل پروسه.

- `ResourceManager::init_global(config) -> Arc<ResourceManager>`، `::global()`، `::global_or_init()`
- `load_all()`، `reload()`، `add_resource_dir(path)`
- گترها: `get_string(key, locale)`، `get_string_formatted(…)`، `get_color(name)`،
  `get_theme(name)`، `current_theme()`، `get_asset(name)`، `get_asset_data(name)`،
  `get_font(name)`، `get_raw(name)`
- `ResourceBuilder` + `StringMap` / `ColorMap` برای ساختن منابع در کد

فایل‌های داده: `resources/strings/strings_en.json`، `resources/strings/strings_es.json`،
`resources/colors/colors.json`.

### ۵.۵ `build-config`

متادیتای نسخه، هدف و امضا.

- `generate_build_metadata() -> BuildTimeConfig` با `VersionInfo`، `GitInfo`، `RustcInfo`، `TargetInfo`
- `platform_build_config(platform) -> BuildConfig`
- امضا: `load_signing_config`، `save_signing_config`، `android_signing_config`، `ios_signing_config`
- `BuildTarget::all_targets()`، `target_for_platform(platform)`
- یک `Cli`/`Commands` کوچک `clap` برای چاپ اطلاعات بیلد

---

## ۶. لایهٔ UI با جزئیات

### ۶.۱ `UiContext` و `UiBuilder`

`UiContext` همان چیزی است که کد رسم می‌خواند؛ `UiBuilder` همان چیزی است که آن را در زمان
راه‌اندازی می‌سازد.

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
    .platform(Platform::Android)                 // دستگاه، نه میزبان بیلد
    .framework(UiFramework::Egui)                // egui تنها بک‌اند فعلی است
    .build();
```

`.platform(...)` روی اندروید مهم است، جایی که فایل کانفیگ پلتفرمِ *میزبان بیلد* را ثبت کرده.
`.device_info(...)` کل `DeviceInfo` تشخیص‌داده‌شده را جایگزین می‌کند.

### ۶.۲ تم

`ThemeManager` یک `ThemeTokens` نگه می‌دارد و روی تغییر به listenerها اطلاع می‌دهد.
`ThemeBuilder` توکن‌ها را می‌سازد؛ تم‌های آماده `material3()`، `ios()`، `material_you()`،
`high_contrast()` و `minimal()` هستند. `generate_css(&tokens)` برای سطح وب CSS تولید می‌کند.

### ۶.۳ Layout

`LayoutCalculator::new(device_info)` به سؤال‌های responsive که ویجت‌ها می‌پرسند جواب می‌دهد:
`grid_columns(base)`، `spacing(base)`، `font_scale(base)`، `is_compact()`، `is_two_pane()`،
`safe_area()`، `breakpoint()`، `is_landscape()` / `is_portrait()`.

builderهای `LayoutBuilder`، `FlexLayoutBuilder`، `GridLayoutBuilder` یک layout فلکس یا گرید را
به‌صورت اعلانی توصیف می‌کنند و `ui::responsive` هلپرهای آزاد می‌دهد (`grid_columns`،
`spacing_multiplier`، `font_scale`، `is_compact`، `is_two_pane`).

### ۶.۴ ناوبری

`NavigationController` یک پشتهٔ مسیر نگه می‌دارد (`navigate`، `push`، `pop`، `replace`،
`reset`، `current`، `previous`، `stack`، `can_go_back`، `depth`، `clear`) و به
`NavigationListener`ها اطلاع می‌دهد. `DeepLinkHandler` الگوهای URL را به `Route` نگاشت می‌کند.
`NavigationBuilder::new(initial_route).route(name, route).deep_link(pattern, resolver).build()`.

### ۶.۵ قرارداد `PlatformUi`

```rust
// crates/ui/src/platform/mod.rs
pub trait PlatformUi: Send + Sync {
    fn platform(&self) -> Platform;
    fn create_window(&self, config: WindowConfig) -> Result<Box<dyn Window>>;
    fn theme(&self) -> PlatformTheme;
    fn announce_for_accessibility(&self, text: &str);
    // …به‌علاوهٔ هلپرهای رفتار که پایین می‌آیند
}

pub struct PlatformFactory;
impl PlatformFactory {
    pub fn create(platform: Platform) -> Box<dyn PlatformUi> { /* … */ }
}
```

`PlatformTheme` سه گروه را کنار هم می‌گذارد: `PlatformColors`، `PlatformFonts`،
`PlatformSpacing`، `PlatformShapes` و `PlatformMotion`. زیرماژول‌های `keyboard`، `pointer` و
`text_input` قراردادهای پلتفرم را ترجمه می‌کنند (`platform_key_mapping`،
`platform_pointer_behavior`، `platform_text_input_behavior` و تایپ‌های
`KeyCode`/`PointerType`/`TextInputConfig`).

---

## ۷. اپلیکیشن‌ها

### ۷.۱ `apps/example`

کل اپلیکیشن در **کتابخانه** (`src/lib.rs`) زندگی می‌کند، با یک باینری نازک (`src/main.rs`) تا
همان کد تبدیل شود به باینری دسکتاپ، `.so` اندروید، `.wasm` وب و `.a` آی‌اواس. نوع‌های crate در
`Cargo.toml` اعلام شده‌اند: `crate-type = ["rlib", "cdylib", "staticlib"]` و نام کتابخانه
`example_app`.

**نقاط ورود**

| نقطهٔ ورود | هدف | نکته |
| --- | --- | --- |
| `run()` | دسکتاپ | `#[cfg(not(target_arch = "wasm32"))]`؛ CLI را پارس می‌کند، boot و `run_egui` |
| `run_web()` | وب | `#[wasm_bindgen(start)]`، `async`؛ `<canvas id="the_canvas_id">` را پیدا می‌کند |
| `android_main(app)` | اندروید | `#[no_mangle] extern "C"`؛ `install_app(app)` سپس boot و اجرا |

**CLI** (`build_cli()` / `parse_args()` → `CliOptions`): `--platform`، `--framework`، `--debug`
(`-d`)، `--config` (`-c PATH`)، `--server URL`. `--server` برای یک اجرا `network.base_url` را
override می‌کند؛ پیش‌فرضِ توکار از قبل به بک‌اند هم‌بسته اشاره می‌کند.

**وابستگی‌های پلتفرم در `Cargo.toml` با `cfg` مهار شده‌اند**، پس بیلد دسکتاپ هیچ‌وقت کد JNI یا
`web-sys` را کامپایل نمی‌کند:

```toml
[target.'cfg(target_os = "android")'.dependencies]
winit = { workspace = true }
android-activity = { workspace = true }
platform-android = { path = "../../crates/platform/android" }

[target.'cfg(target_os = "linux")'.dependencies]
platform-linux = { path = "../../crates/platform/linux" }
# …و windows / macos / ios / wasm32
```

**مرورگر ساعت ندارد.** `wasm32-unknown-unknown` روی `Instant::now()` و `SystemTime::now()`
abort می‌کند. هر چیزی که زمان لازم دارد از `web_time` استفاده می‌کند که روی هدف‌های نیتیو همان
`std::time` است و در مرورگر `performance.now()`/`Date.now()`.

**پنل بک‌اند** (`src/backend.rs`) بدون قفل‌کردن UI به سرور هم‌بسته درخواست می‌زند: درخواست‌ها
روی یک worker thread اجرا می‌شوند و نتیجه را از طریق یک کانال `mpsc` برمی‌گردانند. روی وب — که
سوکت خام وجود ندارد — به‌جای تظاهر به اتصال یک جای‌گزین نشان می‌دهد.

### ۷.۲ `apps/server`

یک **کتابخانه + باینری**: منطق روتینگ/HTTP بدون باز کردن سوکت قابل تست است (`server::Server`،
`server::serve`) و `main.rs` باینری `clap` است.

```bash
cargo run --package server                          # ۱۲۷.۰.۰.۱:۸۰۸۰، کاربران دمو seed می‌شوند
cargo run --package server -- --bind 0.0.0.0:8080    # قابل‌دسترس از امولاتور / LAN
cargo run --package server -- --no-demo              # مخزن کاربر خالی
```

سطح عمومی: `Server::new(services)`، `Server::handle(&Request) -> Response`، `Server::services()`،
`serve(listener, server)`، `DEFAULT_LIMIT = 50`، `MAX_LIMIT = 200`. کد سیم HTTP/1.1 دستی
نوشته شده (`src/http.rs`)؛ درزِ جای‌دادن یک فریمورک مثل `axum` همان جدول روتینگ در `src/lib.rs`
است. باینری دو کاربر دمو seed می‌کند (`alice@example.com`، `bob@example.com`، رمز
`password123`) مگر `--no-demo` داده شود.

---

## ۸. قرارداد سیم و بک‌اندِ هم‌بسته

**قرارداد یک‌بار نوشته می‌شود** در `crates/shared/src/api/mod.rs` و هر دو سر آن را import
می‌کنند، پس کلاینت و سرور از هم دور نمی‌افتند:

```rust
pub mod endpoints {
    pub const HEALTH: &str = "/api/health";
    pub const USERS: &str = "/api/users";
    pub const USERS_SEARCH: &str = "/api/users/search";
}
```

- **سمت سرور:** `apps/server/src/lib.rs` ثابت‌های `endpoints` را به هندلرها نگاشت می‌کند.
- **سمت کلاینت:** `shared::services::HttpNetworkService` — یک کلاینت بلوکینگ HTTP/1.1 ساخته‌شده
  روی `std::net`، `#[cfg(not(target_arch = "wasm32"))]`، که فقط `http://` حرف می‌زند. از
  `reqwest` استفاده نمی‌کند، پس نه فیچر `network` لازم دارد نه پشتهٔ TLS، و بیلدهای اندروید/وب
  بدون TLS می‌مانند.

```rust
let client = HttpNetworkService::new("http://127.0.0.1:8080", Duration::from_millis(5_000));
let health = client.get_json(endpoints::HEALTH)?;
let created = client.post_json(endpoints::USERS, &serde_json::to_value(&request)?)?;
```

هر دو نیمه را با `scripts/dev.sh` با هم اجرا کن (بک‌اند را بالا می‌آورد، صبر می‌کند تا پورت
جواب دهد، کلاینت را با `--server` به آن اشاره می‌دهد و با خروج کلاینت بک‌اند را متوقف می‌کند).
روی اندروید اول پورت میزبان را فوروارد کن: `adb reverse tcp:8080 tcp:8080`.

---

## ۹. فیچرها و ماتریس `cfg`

### ۹.۱ فیچرهای Cargo

| فیچر | crate | پیش‌فرض | اثر |
| --- | --- | --- | --- |
| `network` | `shared` | **خاموش** | کلاینت شبکهٔ `reqwest` (TLS) را فعال می‌کند. خاموش ⇒ `StubNetworkService` که `AppError::Unsupported` برمی‌گرداند و بدون هیچ زنجیرهٔ ابزار C همه‌جا کراس‌کامپایل می‌شود. |
| `egui` | `ui` | **روشن** | منطق *مشترک* کامپوننت‌ها را به ویجت واقعی تبدیل می‌کند. خاموش ⇒ crate باز هم بیلد می‌شود (crateهای پلتفرم به توکن‌ها/traitهایش نیاز دارند) اما متدهای `show(…)` حذف می‌شوند. |
| `hot_reload` | `resources` | خاموش | برای بارگذاری مجدد منابع در آینده رزرو شده. |

صادقانه تأیید کن:

```bash
cargo check -p shared                       # شبکهٔ stub، بدون TLS
cargo build -p shared --features network    # کلاینت واقعی reqwest
cargo check -p ui --no-default-features     # کامپوننت‌ها بدون رندرر
```

### ۹.۲ ماتریس `cfg` (چه چیزی کجا کامپایل می‌شود)

| `cfg` | مربوط به | چه چیزی را مهار می‌کند |
| --- | --- | --- |
| `target_os = "android"` | اندروید | `android_main`، JNI، `platform-android`، چسب winit اندروید |
| `target_os = "linux" / "windows" / "macos" / "ios"` | همان سیستم‌عامل | crate `platform-*` متناظر و شاخهٔ `init()` |
| `target_arch = "wasm32"` | وب | `run_web`، `WEB_CANVAS_ID`، `web-sys`، `getrandom/js`، `uuid/js` |
| `not(target_arch = "wasm32")` | بقیه | `run()`، `native_options()`، `HttpNetworkService` |
| `feature = "egui"` | `ui` | همهٔ `show(…)`های کامپوننت‌ها |
| `feature = "network"` | `shared` | کلاینت `reqwest` |

قاعدهٔ سرانگشتی: **کدی که نمی‌تواند روی یک هدف وجود داشته باشد باید `#[cfg]` شود، نه با `if`
در زمان اجرا.** قالب این را برای `run`/`run_web`/`android_main` الگو کرده است.

---

## ۱۰. دستورها: بیلد، اجرا، تست، لینت

### ۱۰.۱ سه دستور مهم

```bash
cargo check --workspace --all-targets                # ۱. همه‌چیز، هر crate و هدف
cargo test  --workspace --all-targets                # ۲. تست‌ها
cargo clippy --workspace --all-targets --all-features -- -D warnings   # ۳. لینت = خطا
```

به‌علاوهٔ `cargo fmt --all`. CI همین‌ها را اجرا می‌کند و همچنین
`cargo doc --workspace --no-deps` با `RUSTDOCFLAGS=-D warnings`.

### ۱۰.۲ aliasهای کوتاه (`cargo c`، `cargo t`، …)

در `.cargo/config.toml` تعریف شده‌اند. هیچ‌کدام عمداً هم‌نام یک زیرفرمان واقعی Cargo نیست (alias
هم‌نام به خودش resolve می‌شود و بی‌نهایت بازگشت می‌کند).

| alias | معادل |
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

### ۱۰.۳ اسکریپت‌های اجرا و بیلد

```bash
cargo run --package example-app                      # دسکتاپ، URL پیش‌فرض بک‌اند
cargo run --package example-app -- --debug           # -d برای لاگ دیباگ
cargo run --package example-app -- --server http://192.168.1.20:8080

scripts/build-all.sh --list        # این ماشین چه چیزی می‌تواند بسازد
scripts/build-all.sh               # همه را بساز
scripts/build-all.sh android web   # فقط این‌ها

scripts/build-linux.sh [--release]
scripts/build-android.sh [--release] [--abi arm64-v8a] [--apk]
scripts/build-windows.sh [--release]     # MinGW-w64، یا روی ویندوز اجرا کن
scripts/build-macos.sh [--release]       # فقط macOS
scripts/build-ios.sh [--simulator]       # فقط macOS
scripts/build-web.sh [--serve]           # نیازمند trunk
scripts/dev.sh [--release] [--bind ADDR] [--server-only|--client-only]
```

اندروید بدون Gradle (فقط `.so`):

```bash
source scripts/android-env.sh
cargo build --package example-app --target aarch64-linux-android --release
# -> target/aarch64-linux-android/release/libexample_app.so
```

---

## ۱۱. یکپارچه‌سازی مداوم (CI)

`.github/workflows/ci.yml` دروازه‌ها را تعریف می‌کند. یک تغییر تنها وقتی «تمام» است که همه‌شان
پاس شوند.

| Job | Runner | چه چیزی را ثابت می‌کند |
| --- | --- | --- |
| `checks` | ubuntu | `cargo fmt --check`، `clippy … -D warnings`، `check --all-features`، `test --all-features`، `doc -D warnings` |
| `backend` | ubuntu | `server` را می‌سازد، اجرا می‌کند و `/api/health`، `/api/users`، `/api/users/search?q=…` را curl می‌کند |
| `cross-compile` | ubuntu (ماتریس) | اندروید (۴ ABI) `libexample_app.so` تولید می‌کند؛ وب با Trunk بیلد می‌شود |
| `android-apk` | ubuntu | JDK 17 + NDK + Gradle یک `.apk` دیباگ واقعی می‌سازند |
| `desktop` | ubuntu / macos / windows (ماتریس) | باینری نیتیو روی هر سیستم‌عامل بیلد می‌شود |
| `ios` | macos | کتابخانهٔ استاتیک iOS کامپایل می‌شود (امضا خارج از حوزهٔ CI است) |

**دامنهٔ صادقانه:** CI کامپایل، لینت و تست روی هر هدف را ثابت می‌کند؛ **نه** یک بیلد امضاشدهٔ
فروشگاهی را، و اپ را روی دستگاه فیزیکی اجرا نمی‌کند. برای بیانیهٔ صراحت خود پروژه بخش «چه چیزی
تست شده و چه چیزی نه» را در README ببین.

---

## ۱۲. دستورهای آماده: چطور چیزی اضافه کنیم

هر دستور می‌گوید **کدام فایل را باید دست بزنی و کدام قواعد (§۱۳) را رعایت کنی**.

### دستور A — افزودن یک صفحهٔ جدید

1. یک variant به `Route` در `crates/shared/src/domain/mod.rs` اضافه کن.
2. با یک `NavigationController` (`ui::navigation`) به آن برو —
   `navigate(route, NavDirection::Forward)`.
3. آن را داخل `ExampleApp::update` (یا `eframe::App` خودت) با match روی `controller.current()`
   رندر کن و یک هلپر `draw_*` صدا بزن.
4. کد رسم را در اپ (`apps/example`) نگه دار — *مدل مسیر* جای‌اش در `shared` است.

### دستور B — افزودن یک کامپوننت مشترک

1. `crates/ui/src/components/<name>.rs` را بساز.
2. **اول منطق خالص:** struct، یک `new(...)`، متدهای builder و هر ریاضیات اندازه/state — همه
   **بدون تایپ‌های `egui`** تا با خاموشی فیچر `egui` هم کامپایل شود.
3. متد رندر را پشت `#[cfg(feature = "egui")] pub fn show(&self, ui: &mut egui::Ui, ctx: &UiContext)` بگذار.
4. از `crates/ui/src/components/mod.rs` دوباره export کن (`pub use <name>::…;`).
5. برای منطقی که بدون `egui` اجرا می‌شود یک تست واحد بگذار.

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

### دستور C — افزودن یک endpoint بک‌اند (اول قرارداد)

1. مسیر را به `shared::api::endpoints` و تایپ‌های درخواست/پاسخ را به
   `crates/shared/src/api/mod.rs` اضافه کن (هر دو `serde::Serialize + Deserialize`).
2. در `apps/server/src/lib.rs` هندلش کن (match روی مسیر، تولید یک `Response`).
3. از کلاینت با `HttpNetworkService::get_json`/`post_json` صدا بزن.
4. هر دو سر روی همان تایپ کامپایل می‌شوند — یک فیلد را تغییر نام بده و سر دیگر از کامپایل
   می‌افتد.
5. اگر endpoint کاربرمحور است یک بررسی دودی شبیه job `backend` اضافه کن.

### دستور D — افزودن یک تایپ دامنه + مخزن

1. آن را در `crates/shared/src/domain/mod.rs` تعریف کن و `Entity` (و اگر در مخزن می‌نشیند
   `RepositoryEntity`) را پیاده کن.
2. برای ذخیره از `InMemoryRepository::<T>::new()` استفاده کن، یا به یک trait سرویس موجود در
   `crates/shared/src/services/mod.rs` متد اضافه کن و در `services::defaults` پیاده‌اش کن.
3. آن را در `ServiceContainer`/`AppServices` در `defaults::*` و در
   `services::factory::create_services` سیم‌کشی کن.
4. داخل `shared` برگرداندن `shared::errors::Result<T>` (`AppError`) را به `anyhow` ترجیح بده.

### دستور E — افزودن یک قابلیت پلتفرم

1. اگر یک *قرارداد* است (متدی که هر پلتفرم پاسخ می‌دهد): به `PlatformUi` / `Window` در
   `crates/ui/src/platform/mod.rs` اضافه کن و پیش‌فرض را در `crates/ui/src/platform/*.rs`
   پیاده کن.
2. اگر *کار واقعی OS* است (JNI، `Win32`، `uname`، `web-sys`، هپتیک): در
   `crates/platform/<os>/src/lib.rs` متناظر و پشت `#[cfg(target_os = "…")]` پیاده کن.
3. یک تابع امری کوچک (مثل `platform_android::device_info()`) بیرون بده و از
   `init_platform`/نقطهٔ ورود اپ صدا بزن، **مهارشده با `#[cfg]`**.

### دستور F — تغییر تم

- توکن‌ها را برنامه‌نویسی‌شده با `ThemeBuilder` تغییر بده (`primary_color`، `secondary_color`،
  `font_family`، `base_font_size`، `spacing_scale`، `border_radius_scale`، `motion_speed`).
- یا یکی از تم‌های آماده را بردار: `material3()`، `ios()`، `material_you()`، `high_contrast()`،
  `minimal()`.
- در زمان اجرا `ThemeManager` یک تم زنده را به‌روز می‌کند و `generate_css()` آن را برای وب
  export می‌کند.
- انتخاب تم کاربر در `AppConfig.ui.theme.mode` (`ThemeMode`) می‌نشیند.

### دستور G — افزودن یک رشتهٔ محلی‌سازی‌شده

1. کلید را به `resources/strings/strings_en.json` (و `strings_es.json`) اضافه کن.
2. با `ResourceManager::global().get_string("key", Some("en"))` بخوان، یا برای placeholderها
   `get_string_formatted(…)`.
3. `ResourceManager::load_all()` / `reload()` کش را تازه می‌کند.

### دستور H — اشارهٔ کلاینت به بک‌اند دیگر

- برای یک اجرا: `cargo run --package example-app -- --server http://host:port` (برای
  `dev.sh` هم `--server` هست).
- به‌صورت دائمی: `network.base_url` را در کانفیگی که `AppConfig::save()` می‌نویسد تغییر بده.
- روی اندروید اول `adb reverse tcp:8080 tcp:8080` (لوکال‌هاست دستگاه ≠ لوکال‌هاست میزبان).

### دستور I — صدا زدن بک‌اند از UI بدون فریز شدن

- از `apps/example/src/backend.rs` الگو بگیر: یک worker thread بساز، فراخوانی بلوکینگ
  `HttpNetworkService` را آنجا انجام بده و نتیجه را از طریق کانال `mpsc` برگردان که `update()`
  هر فریم تخلیه‌اش می‌کند. هیچ‌وقت کلاینت بلوکینگ را روی ترد egui صدا نزن.
- روی وب کلاینت تقلبی نفرست؛ یک جای‌گزین رندر کن (سوکت خام وجود ندارد) یا روی مسیر `wasm32`
  از `web_sys::fetch` استفاده کن.

### دستور J — تغییر نام قالب به اپ خودت

1. `[lib] name` در `apps/example/Cargo.toml` را عوض کن **و** دو شناسهٔ اندروید که باید با آن
   یکی باشند را به‌روز کن (`android.app.lib_name` در `AndroidManifest.xml` و `rustLibName` در
   Gradle).
2. `WINDOW_TITLE` در `apps/example/src/lib.rs` را عوض کن.
3. متادیتای `[workspace.package]` (`repository`، `homepage`، `authors`، `description`) را در
   `Cargo.toml` ریشه به‌روز کن.
4. application id پروژهٔ `android-app` و رشته‌های `resources/` را به‌روز کن.

---

## ۱۳. قواعدی که یک تغییر نباید بشکند

این‌ها قواعدی‌اند که ورک‌اسپیس را کراس‌کامپایل‌پذیر و ادعاهای README را درست نگه می‌دارند.

1. **وابستگی‌ها یک‌طرفه‌اند.** `shared` به هیچ چیز داخلی وابسته نیست؛ `ui`/`resources`/
   `build-config` فقط به `shared`؛ `platform/*` به `shared` + `ui`؛ `apps/server` فقط به
   `shared`. **هرگز** `shared` یا `ui` را به یک crate پلتفرم وابسته نکن — این چرخه می‌سازد و
   `cargo check --workspace` روی ماشینی بدون SDK هم باید کار کند.
2. **`shared` و نیمهٔ خالص `ui` هیچ `egui` و هیچ تایپ OS ندارند.**
3. **منطق خالص هر کامپوننت با خاموشی `egui` هم کامپایل می‌شود.**
   `cargo check -p ui --no-default-features` باید سبز بماند.
4. **قرارداد سیم فقط در `shared::api` زندگی می‌کند.** هر دو سر importش می‌کنند؛ DTO را تکرار نکن.
5. **نقاط ورودی که نمی‌توانند روی یک هدف وجود داشته باشند `#[cfg]` می‌شوند**، نه شاخهٔ زمان اجرا.
6. **زمان از `web_time` می‌گذرد**، هرگز مستقیم `std::time::Instant::now()` /
   `SystemTime::now()`، تا هدف wasm abort نکند.
7. **نام کتابخانهٔ اندروید بارِ معنایی دارد:** باید در `Cargo.toml`، manifest و Gradle یکی باشد.
8. **Clippy با `-D warnings`** و `all`، `pedantic`، `nursery` و `cargo` روی warn است (لینت
   ورک‌اسپیس). کد جدید باید بدون خطای لینت باشد.
9. **`Cargo.lock` کامیت‌شده می‌ماند** و `panic = "abort"` از پروفایل release بیرون می‌ماند
   (وگرنه `cargo test --release` بی‌استفاده می‌شود).
10. **فیچر `network` پیش‌فرض خاموش می‌ماند** و `HttpNetworkService` بدون `reqwest` می‌ماند، تا
    بیلدهای اندروید/وب بدون زنجیرهٔ ابزار TLS/C کراس‌کامپایل شوند.

---

## ۱۴. عیب‌یابی و تله‌ها

| نشانه | علت / راه‌حل |
| --- | --- |
| `web-sys` «no method named `class_list`/`style`/`head`» | `web-sys` یک متد را فقط وقتی تولید می‌کند که فیچرش روشن باشد. فیچر را به لیست `web-sys` در `Cargo.toml` ریشه اضافه کن. |
| بیلد اندروید linker/`cc` پیدا نمی‌کند | NDK r23+ سیم‌لینک‌های خام `<arch>-linux-android-clang` را حذف کرد. `source scripts/android-env.sh` (بیان‌کنندهٔ `CARGO_TARGET_*_LINKER`، `CC_*`، `CXX_*`، `AR_*`). |
| لایهٔ ویندوز فقط هنگام هدف‌گیری ویندوز خطای `cargo check` می‌دهد، ولی `cargo check --workspace` روی لینوکس سالم است | کد دسکتاپ پشت `#[cfg(target_os = "windows")]` است، پس میزبان لینوکسی آن را تایپ‌چک نمی‌کند. بدون لینکر کراس‌چک کن: `rustup target add x86_64-pc-windows-msvc` و سپس `cargo check --workspace --target x86_64-pc-windows-msvc`. ساخت خروجی واقعی (`.exe`) همچنان به MinGW-w64 (`scripts/build-windows.sh`) یا میزبان ویندوز نیاز دارد. |
| کد `android_main` در ادیتور type-check نمی‌شود | rust-analyzer برای میزبان تحلیل می‌کند. در `.vscode/settings.json` **کامیت‌نشدهٔ** خودت `rust-analyzer.cargo.target = "aarch64-linux-android"` بگذار. |
| دو تایپ ناسازگار `JNIEnv` | دو نسخهٔ `jni` در گراف. `jni = "0.22"` باید با آنچه `android-activity` 0.6 resolve می‌کند یکی باشد. |
| خطای `uuid`/`getrandom` روی wasm | آنتروپی به بک‌اند `js` نیاز دارد: `getrandom { features = ["js"] }` و `uuid { features = ["v4","serde","js"] }` برای گراف wasm (از قبل در `apps/example/Cargo.toml` اعلام شده). |
| فریز/پنیک روی زمان در مرورگر | روی `wasm32` از `web_time` استفاده کن، نه `std::time`. |
| `cargo <alias>` بی‌نهایت بازگشت می‌کند | alias نباید هم‌نام زیرفرمان واقعی Cargo باشد. از `f`/`l` استفاده کن نه `fmt`/`clippy`. |
| بک‌اند از امولاتور در دسترس نیست | `127.0.0.1` خودِ دستگاه است. `adb reverse tcp:8080 tcp:8080` یا `--bind 0.0.0.0:8080`. |
| متدهای `ui` با `--no-default-features` نیستند | انتظار می‌رود: `show(…)` پشت فیچر `egui` است. |
| `hooks`/`utils` خالی به‌نظر می‌رسند | به‌عنوان فایل هستند ولی در `ui/src/lib.rs` **declare نشده‌اند** — عمداً فعلاً خارج از پشتهٔ egui‌اند. |

---

## ۱۵. نکات برای دستیارهای هوش مصنوعی

قواعد عینی که کد تولیدشده را در این مخزن درست می‌کند:

1. **اول پیدا کن، بعد بنویس.** از درخت §۴ و نقشهٔ §۵ استفاده کن. اگر نمادی آنجا نیست، وجود
   ندارد — آن را در crateی که §۳.۱ می‌گوید بساز.
2. **لایه‌ها را رعایت کن.** منطق کسب‌وکار → `shared`. ویجت قابل‌استفادهٔ مجدد →
   `ui::components` با یک نیمهٔ منطق خالص و یک `show` پشت `cfg(egui)`. فراخوانی OS →
   `crates/platform/*`. سیم‌کشی اپ → `apps/example`. endpoint → `shared::api` +
   `apps/server/src/lib.rs`.
3. **بیلد بدون `egui` را سبز نگه دار.** در `ui` هیچ تایپ `egui` نباید بیرون از بلوک‌های
   `#[cfg(feature = "egui")]` ظاهر شود.
4. **در `shared` `shared::errors::Result`/`AppError` را ترجیح بده؛** `anyhow` در لبهٔ اپ خوب است.
5. **کد per-target را با `#[cfg]` مهار کن،** هم‌شکل الگوی موجود `run`/`run_web`/`android_main`.
   هرگز یک API اندروید/JNI/`web-sys` را بدون مهار صدا نزن.
6. **راه دومی برای کار موجود نساز** (کلاینت HTTP دوم، سیستم تم دوم، روتر دوم). درزِ موجود را
   گسترش بده.
7. **پیش از اعلام «تمام‌شدن»** دروازه‌های §۱۰ را اجرا کن و صادقانه بگو چه چیزی را اجرا کردی و
   چه چیزی را نه (هم‌سبک خود پروژه).
8. **کامنت‌ها *چرا* را توضیح می‌دهند.** کامنت‌های این کدبیس استدلال را ثبت می‌کنند (چرا یک
   `cfg`، چرا `jni` مهم است، چرا یک alias کوتاه است) — همان سبک را رعایت کن، نه کامنت «چه».

---

## ۱۶. منابع و راه ارتباطی

- [README.fa.md](README.fa.md) — شروع سریع و مرور پروژه (فارسی).
- [README.md](README.md) — نسخهٔ انگلیسی.
- [TEMPLATE.md](TEMPLATE.md) — نسخهٔ انگلیسی این راهنما.
- قرارداد سیم: `crates/shared/src/api/mod.rs`
- کتابخانهٔ کامپوننت: `crates/ui/src/components/`
- traitهای پلتفرم: `crates/ui/src/platform/mod.rs`
- نقاط ورود: `apps/example/src/lib.rs`
- بک‌اند: `apps/server/src/lib.rs`
- دروازه‌های CI: `.github/workflows/ci.yml`

**نویسنده:** سینا خانزاده — <khanzadeh.1377@gmail.com> — [sina-khanzadeh.ir](https://sina-khanzadeh.ir)
**اینستاگرام / تلگرام / دیسکورد:** `programmer_1998`
**مجوز:** MIT OR Apache-2.0 — بخش `[workspace.package] license` در `Cargo.toml`.

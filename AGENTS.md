# AGENTS.md — Android UI Rules (Jetpack Compose + Material 3)

> **This file is the rulebook for every AI coding agent working on this app's UI.**
> If a rule here conflicts with your own preference, a blog post, a tutorial, or "what looks nice" — **this file wins.**
> If a rule here conflicts with the actual codebase, **stop and report the conflict** instead of silently picking one.

---

## 0. Non-negotiables (read this first)

1. **No hardcoded colors.** Every color comes from `MaterialTheme.colorScheme` or a token in `ui/theme/Color.kt`. `Color(0xFF...)` inside a screen or component file is a **build-blocking defect**.
2. **No off-grid spacing.** Every `padding`, `spacer`, `size`, `width`, `height` uses a value from the spacing scale (§4). `12.dp`, `18.dp`, `22.dp` are defects.
3. **No hardcoded typography.** Every text uses `MaterialTheme.typography.*`. `fontSize = 15.sp` in a screen is a defect.
4. **No hardcoded shapes.** Every corner radius comes from `MaterialTheme.shapes.*` or the shape token file. `RoundedCornerShape(13.dp)` is a defect.
5. **M3 component first.** Before writing a custom composable, prove the M3 component cannot do the job (§6).
6. **No hardcoded user-facing strings.** All copy lives in `res/values/strings.xml` (§9).
7. **Every interactive element is reachable, labeled, and ≥48dp** (§8).
8. **State is hoisted.** Screens are stateless; state lives in a `ViewModel` exposed as one `UiState` (§7).

A PR that violates any of 1–8 is **not reviewable** — fix it before asking for review.

---

## 1. Project layout (where things go)

```
app/src/main/java/<pkg>/
├── ui/
│   ├── theme/
│   │   ├── Color.kt          # raw palette + M3 ColorScheme light/dark
│   │   ├── Type.kt           # M3 Typography scale
│   │   ├── Shape.kt          # M3 Shapes scale
│   │   ├── Dimens.kt         # spacing / radius / elevation / size tokens
│   │   └── Theme.kt          # <App>Theme { } — the ONLY theme entry point
│   ├── components/           # shared, reusable composables
│   │   ├── AppTopBar.kt
│   │   ├── AppScaffold.kt
│   │   ├── PrimaryButton.kt
│   │   └── ...
│   ├── screens/
│   │   └── <feature>/
│   │       ├── <Feature>Screen.kt        # stateless UI
│   │       ├── <Feature>ViewModel.kt     # state + events
│   │       └── <Feature>UiState.kt       # state model
│   └── navigation/
│       └── AppNavHost.kt
└── data/ ...
```

**Rules**
- A screen file contains **only** composables. No `ViewModel` construction, no repository calls, no `Context` business logic.
- Anything used by **two or more** screens moves to `ui/components/`. Do not copy-paste composables.
- One public composable per screen file, named `<Feature>Screen`. Private helpers stay `private`.
- No file over **400 lines**. If it grows past that, split by section into `components/`.

---

## 2. Theme entry point

There is exactly **one** theme wrapper, and it is applied once, at the Activity root.

```kotlin
// ui/theme/Theme.kt
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = false,   // see rule below
    content: @Composable () -> Unit,
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val ctx = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(ctx) else dynamicLightColorScheme(ctx)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }
    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography,
        shapes = AppShapes,
        content = content,
    )
}
```

```kotlin
// MainActivity.kt
setContent {
    AppTheme {           // ← no arguments invented at the call site
        AppNavHost()
    }
}
```

**Rules**
- `MaterialTheme { }` is **never** called directly in a screen. Always `AppTheme { }`.
- **Dynamic color is OFF by default.** It is a product decision, not a default. If enabled, it must be a user setting that actually reaches `AppTheme(dynamicColor = ...)`.
- **A theme setting must actually change the theme.** If a "Dark / Light / System" preference exists in Settings, it must be read at the Activity root and passed into `AppTheme(darkTheme = ...)`. A preference that only changes a label in a list is a **bug**, not a feature.
- Never wrap a subtree in a second `MaterialTheme` to "fix" a color. Fix the token instead.

---

## 3. Color — M3 color roles only

### 3.1 The role table

Use the **role**, never the hex. Pick by intent:

| Intent | Role |
|---|---|
| Screen background | `colorScheme.background` |
| Card / sheet / elevated surface | `colorScheme.surface` / `surfaceContainer*` |
| Primary action (filled button, FAB) | `colorScheme.primary` on `onPrimary` |
| Secondary action (tonal button, chip selected) | `secondaryContainer` on `onSecondaryContainer` |
| Text — primary | `colorScheme.onSurface` |
| Text — secondary / supporting | `colorScheme.onSurfaceVariant` |
| Text — on colored surface | `colorScheme.onPrimary` / `onSecondaryContainer` / `onError` |
| Dividers, outlines, borders | `colorScheme.outlineVariant` (subtle) / `outline` (strong) |
| Error text / error container | `colorScheme.error` / `errorContainer` on `onErrorContainer` |
| Disabled content | `colorScheme.onSurface.copy(alpha = 0.38f)` |
| Scrim behind modal | `colorScheme.scrim` |
| Inverse (snackbar) | `inverseSurface` on `inverseOnSurface` |

### 3.2 Rules

- **Never** write `Color(0xFF...)`, `Color.Black`, `Color.White`, `Color.Gray` in a screen or component. Only `ui/theme/Color.kt` may contain raw hex.
- **Never** use `Color.Red` for errors. Use `colorScheme.error`.
- **Never** invent a color by alpha-blending a role to "make it softer" unless the token file already defines that token. Add the token, then use it.
- **Contrast is a hard requirement** (§8.3). If a role pair fails contrast, the palette is wrong — fix the palette, do not patch the screen.
- **Dark theme is not "invert the light theme."** Both schemes are authored explicitly in `Color.kt` and both must pass contrast checks.
- **No gradients as decoration.** See §10.

### 3.3 Token file shape

```kotlin
// ui/theme/Color.kt
internal val Green40 = Color(0xFF1B6B3A)
internal val Green80 = Color(0xFF7FD8A0)
// ... full tonal palette, named by tone, not by usage

internal val LightColorScheme = lightColorScheme(
    primary = Green40,
    onPrimary = Color.White,
    primaryContainer = Green90,
    onPrimaryContainer = Green10,
    // ... every role filled explicitly — no reliance on library defaults
)

internal val DarkColorScheme = darkColorScheme(
    primary = Green80,
    onPrimary = Green20,
    // ...
)
```

**Rule:** every role in both schemes is set **explicitly**. Do not leave a role to the library default — defaults drift between Compose versions and silently change your UI.

---

## 4. Spacing, radius, elevation — the token scales

### 4.1 Spacing grid — 4dp base

```kotlin
// ui/theme/Dimens.kt
object Spacing {
    val xxs = 2.dp
    val xs  = 4.dp
    val sm  = 8.dp
    val md  = 12.dp
    val lg  = 16.dp
    val xl  = 24.dp
    val xxl = 32.dp
    val xxxl = 48.dp
}
```

**Allowed values only:** `2, 4, 8, 12, 16, 24, 32, 48, 64`.
**Forbidden:** `6, 10, 14, 18, 20, 22, 28, 36, 40` and any other non-listed value.

**Screen padding:** horizontal `Spacing.lg` (16dp) on phones, `Spacing.xl` (24dp) on tablets/expanded width.
**Vertical rhythm between sections:** `Spacing.xl` (24dp). Between related items: `Spacing.sm` (8dp) or `Spacing.md` (12dp).

### 4.2 Shape scale — M3

```kotlin
// ui/theme/Shape.kt
internal val AppShapes = Shapes(
    extraSmall = RoundedCornerShape(4.dp),
    small      = RoundedCornerShape(8.dp),
    medium     = RoundedCornerShape(12.dp),
    large      = RoundedCornerShape(16.dp),
    extraLarge = RoundedCornerShape(28.dp),
)
```

| Use | Shape |
|---|---|
| Chip, badge, small tag | `shapes.small` |
| Button, text field, card | `shapes.medium` |
| Dialog, bottom sheet top corners, large card | `shapes.large` |
| FAB, full-width sheet | `shapes.extraLarge` |
| Avatar, icon button, circular | `CircleShape` |

**Rule:** `RoundedCornerShape(<number>.dp)` outside `Shape.kt` is a defect. Use `MaterialTheme.shapes.*`.

### 4.3 Elevation

Use M3 tonal elevation, not arbitrary shadows.

| Surface | Elevation |
|---|---|
| Flat content on background | `0.dp` |
| Card (resting) | `1.dp` |
| Card (raised / hovered) | `3.dp` |
| Bottom bar / nav bar | `3.dp` |
| FAB | `6.dp` |
| Modal sheet / dialog | `6.dp` |

**Rule:** prefer `Surface(tonalElevation = ...)` over `Modifier.shadow(...)`. Custom `shadow(elevation = 12.dp, shape = ...)` is a defect unless the design system explicitly defines it.

---

## 5. Typography — M3 scale only

```kotlin
// ui/theme/Type.kt
internal val AppTypography = Typography(
    displayLarge  = TextStyle(fontFamily = AppFont, fontWeight = FontWeight.Normal, fontSize = 57.sp, lineHeight = 64.sp, letterSpacing = (-0.25).sp),
    displayMedium = TextStyle(...),
    displaySmall  = TextStyle(...),
    headlineLarge = TextStyle(...),
    headlineMedium= TextStyle(...),
    headlineSmall = TextStyle(...),
    titleLarge    = TextStyle(...),
    titleMedium   = TextStyle(...),
    titleSmall    = TextStyle(...),
    bodyLarge     = TextStyle(...),
    bodyMedium    = TextStyle(...),
    bodySmall     = TextStyle(...),
    labelLarge    = TextStyle(...),
    labelMedium   = TextStyle(...),
    labelSmall    = TextStyle(...),
)
```

### 5.1 Which style for what

| Content | Style |
|---|---|
| Screen title in top app bar | `titleLarge` |
| Section header | `titleMedium` |
| Card title | `titleMedium` |
| Body copy, message text | `bodyLarge` |
| Supporting / secondary text | `bodyMedium` |
| Caption, timestamp, helper | `bodySmall` |
| Button label | `labelLarge` |
| Chip label, tab label | `labelLarge` / `labelMedium` |
| Badge, overline | `labelSmall` |

### 5.2 Rules

- **Never** set `fontSize`, `lineHeight`, or `letterSpacing` in a screen. If a size is missing, add a role to `Type.kt` — do not inline it.
- **Never** set `fontFamily` in a screen. The family is set once in `Type.kt`.
- **Never** use `fontWeight = FontWeight.Bold` to fake emphasis. Use the correct role (`titleMedium` vs `bodyMedium`).
- **Custom fonts must be wired.** If a font file ships in `res/font/`, it must be referenced by `Type.kt`. A shipped-but-unused font is dead weight — either wire it or delete it.
- **Line height is part of the token.** Do not override it for "tightness."

---

## 6. Components — M3 first, custom last

### 6.1 Decision order

Before writing a composable, walk this list and stop at the first match:

1. **Is there an M3 component for this?** → use it (`Button`, `Card`, `TextField`, `NavigationBar`, `ModalBottomSheet`, `FilterChip`, `ListItem`, `Snackbar`, `AlertDialog`, `SegmentedButton`, `TopAppBar`, `FloatingActionButton`, `Badge`, `Divider`/`HorizontalDivider`, `Switch`, `Checkbox`, `RadioButton`, `Slider`, `LinearProgressIndicator`, `CircularProgressIndicator`, `PullToRefreshBox`).
2. **Can it be composed from M3 components?** → compose them (`Surface` + `Row` + `Text`).
3. **Does the design system already have a shared component?** → use `ui/components/*`.
4. **Only then** write a new composable — and put it in `ui/components/`, not in the screen.

### 6.2 Component rules

- **Never re-implement an M3 component.** A hand-rolled button with `Box` + `clickable` + `background` is a defect. Use `Button`.
- **Never fork an M3 component to change one color.** Use the component's own parameters (`colors = ButtonDefaults.buttonColors(...)`) with **theme roles**.
- **Every shared component takes a `Modifier` as its first optional parameter** and applies it to its root:
  ```kotlin
  @Composable
  fun PrimaryButton(
      text: String,
      onClick: () -> Unit,
      modifier: Modifier = Modifier,
      enabled: Boolean = true,
  ) { ... }
  ```
- **No `Modifier` default of anything but `Modifier`.** Never `modifier: Modifier = Modifier.padding(16.dp)`.
- **Components are stateless.** No `remember { mutableStateOf }` for anything the caller might need to read. Hoist it (§7).
- **No `Context` in a component** unless it is genuinely required (e.g. `stringResource` handles most cases without it).
- **Previews are required** for every shared component, with `@Preview` for light and dark:
  ```kotlin
  @Preview(name = "Light", showBackground = true)
  @Preview(name = "Dark", showBackground = true, uiMode = UI_MODE_NIGHT_YES)
  @Composable
  private fun PrimaryButtonPreview() { AppTheme { PrimaryButton("Continue", {}) } }
  ```

### 6.3 Lists

- `LazyColumn` / `LazyRow` **must** supply a stable `key`:
  ```kotlin
  items(items = messages, key = { it.id }) { message -> ... }
  ```
  An index-based key is a defect — it breaks scroll position and item state on insert/remove.
- `contentPadding` on the lazy list, not `padding` on the list itself (otherwise the last item is clipped).
- Never nest a `LazyColumn` inside a `LazyColumn` with the same scroll axis. Use a single list with item types.

---

## 7. State — hoisting and `UiState`

### 7.1 The pattern

```kotlin
// <Feature>UiState.kt
data class ChatUiState(
    val isLoading: Boolean = false,
    val messages: List<Message> = emptyList(),
    val input: String = "",
    val error: UiError? = null,
)

sealed interface ChatEvent {
    data class InputChanged(val value: String) : ChatEvent
    data object Send : ChatEvent
    data object Retry : ChatEvent
}
```

```kotlin
// <Feature>ViewModel.kt
class ChatViewModel(private val repo: ChatRepository) : ViewModel() {
    private val _state = MutableStateFlow(ChatUiState())
    val state: StateFlow<ChatUiState> = _state.asStateFlow()

    fun onEvent(event: ChatEvent) { /* reduce into _state */ }
}
```

```kotlin
// <Feature>Screen.kt — STATELESS
@Composable
fun ChatScreen(
    state: ChatUiState,
    onEvent: (ChatEvent) -> Unit,
    modifier: Modifier = Modifier,
) { ... }
```

```kotlin
// route — the only place that knows the ViewModel
@Composable
fun ChatRoute(vm: ChatViewModel = viewModel()) {
    val state by vm.state.collectAsStateWithLifecycle()
    ChatScreen(state = state, onEvent = vm::onEvent)
}
```

### 7.2 Rules

- **One `UiState` per screen.** Not five separate `StateFlow`s that the UI has to combine.
- **Screens are stateless.** A `@Composable` named `*Screen` must not call `viewModel()`, must not touch a repository, and must be previewable with a literal state object.
- **State goes down, events go up.** No callbacks that mutate state directly; they emit an event.
- **`collectAsStateWithLifecycle()`**, never `collectAsState()`. The latter keeps collecting while the app is backgrounded.
- **`rememberSaveable`** for UI-only state that must survive process death (scroll position, expanded/collapsed, selected tab). `remember` for everything else.
- **`remember` keys must be correct.** `remember { }` capturing a changing value without a key is a stale-state bug.
- **No `runBlocking` on the main thread.** Ever. Not for a disk write, not for a "quick" call. Use `viewModelScope.launch` + `Dispatchers.IO`.
- **`LaunchedEffect` keys must be the real dependencies.** `LaunchedEffect(Unit)` that reads a changing value is a bug.
- **Derived values use `derivedStateOf`**, not a `remember` that recomputes on every recomposition.

---

## 8. Accessibility — WCAG 2.1 AA, mandatory

### 8.1 Touch targets

- **Minimum 48×48dp** for every interactive element, including icon buttons, chips, and list rows.
- If the visual is smaller (e.g. a 24dp icon), expand the hit area:
  ```kotlin
  Modifier
      .size(48.dp)
      .clickable(onClickLabel = "Open settings") { ... }
  ```
- **Minimum 8dp** between adjacent touch targets.
- Use the shared `Modifier.minTouchTarget()` helper if the project has one — do not re-implement it per screen.

### 8.2 Content descriptions

- **Every `Icon` that conveys meaning needs a `contentDescription`.** Decorative icons use `contentDescription = null`.
- **Every `Image` needs either `contentDescription` or an explicit `null`** with a comment saying why it is decorative.
- **Icon-only buttons** need a description that names the action: `"Send message"`, not `"Icon"`.
- **Never** put the description on a container that also has text — the text is already read. Avoid double-announcing.
- **Custom composables** that are interactive must set `Modifier.semantics { role = Role.Button; onClick(label = "...") { ...; true } }` or use `clickable(onClickLabel = ...)`.

### 8.3 Contrast

- **Body text:** ≥ **4.5:1** against its background.
- **Large text** (≥18sp regular / ≥14sp bold) and **icons/graphics:** ≥ **3:1**.
- **UI component boundaries** (input borders, focus rings): ≥ **3:1**.
- **Never** rely on color alone to convey state. Pair with an icon, label, or shape change.
- Check **both** light and dark schemes. A pair that passes in light and fails in dark is a defect.

### 8.4 Semantics and structure

- **Headings:** use `Modifier.semantics { heading() }` on section titles so TalkBack can jump between them.
- **State:** use `Modifier.semantics { stateDescription = "Selected" }` for toggles, and `selected` / `disabled` semantics where applicable.
- **Live regions:** for a typing indicator or a new message, use `Modifier.semantics { liveRegion = LiveRegionMode.Polite }`.
- **Grouping:** `Modifier.semantics(mergeDescendants = true) { }` on a card so it is read as one unit, not five fragments.
- **Focus order** must follow visual order. Do not use `focusOrder` to paper over a bad layout.

### 8.5 Motion and text scaling

- **Respect `LocalMotionDurationScale`** — do not hardcode animation durations that ignore the system "remove animations" setting.
- **Respect font scale.** Never use fixed-height containers for text. Test at **200% font scale**; text must not clip or overlap.
- **No auto-playing motion** that cannot be paused.

---

## 9. Strings and localization

- **Every user-facing string** lives in `res/values/strings.xml` and is read with `stringResource(R.string.x)`.
- **No string literals in composables.** `Text("Send")` is a defect; `Text(stringResource(R.string.send))` is correct.
- **Plurals** use `<plurals>` + `pluralStringResource`.
- **Formatted strings** use positional args: `<string name="greeting">Hello, %1$s</string>`.
- **No string concatenation** to build a sentence — word order differs across languages.
- **Content descriptions are strings too** — they go in `strings.xml`.

---

## 10. Anti-patterns — do not do these

### 10.1 The "AI aesthetic" (banned)

- ❌ **Purple/blue gradients** as a background or button fill. This is the single most common "generated by an AI" tell. Use a flat M3 role.
- ❌ **Glassmorphism / frosted blur** panels.
- ❌ **Neon glow** shadows, `shadow(ambientColor = Color.Magenta)`.
- ❌ **Gradient text** (`Brush.linearGradient` on a `TextStyle`).
- ❌ **Emoji as UI icons.** Use `Icons.*` or a vector drawable.
- ❌ **Random decorative blobs / orbs / sparkles** behind content.
- ❌ **Every card with a different radius.** One shape scale, applied consistently.
- ❌ **"Hero" gradients on every screen** — a screen is not a landing page.

### 10.2 Compose anti-patterns

- ❌ `Color(0xFF...)` in a screen or component.
- ❌ `12.dp` / `18.dp` / `22.dp` spacing.
- ❌ `fontSize = 15.sp` inline.
- ❌ `RoundedCornerShape(13.dp)` inline.
- ❌ `runBlocking` on the main thread.
- ❌ `collectAsState()` instead of `collectAsStateWithLifecycle()`.
- ❌ `LazyColumn` without `key`.
- ❌ `Modifier.fillMaxSize()` on a child inside a scrollable parent.
- ❌ Nested scrollables on the same axis.
- ❌ `remember { }` capturing a changing value with no key.
- ❌ `LaunchedEffect(Unit)` that reads changing state.
- ❌ Passing `Context` into a composable to read a string — use `stringResource`.
- ❌ `!!` on a nullable that can legitimately be null (callbacks, optional fields). Use `?.let` or a default.
- ❌ `e.message` shown to the user. Map to a friendly, localized message; log the technical detail.
- ❌ A `Box` with `clickable` pretending to be a button.
- ❌ `Modifier.offset` to fake layout — use real padding/arrangement.
- ❌ Hardcoded `width`/`height` on text containers.
- ❌ `@Composable` functions over ~80 lines — extract.

### 10.3 Architecture anti-patterns

- ❌ Business logic in a composable.
- ❌ A `ViewModel` referenced from a `*Screen` composable.
- ❌ Multiple `StateFlow`s the UI must combine.
- ❌ A settings toggle that persists but never reaches the code that would apply it.
- ❌ Duplicated component code across screens.
- ❌ A "temporary" hardcoded color with a `// TODO` — it will ship.

---

## 11. Screen structure and navigation

### 11.1 Screen skeleton

Every screen follows this shape:

```kotlin
@Composable
fun <Feature>Screen(
    state: <Feature>UiState,
    onEvent: (<Feature>Event) -> Unit,
    modifier: Modifier = Modifier,
) {
    Scaffold(
        modifier = modifier,
        topBar = { AppTopBar(title = stringResource(R.string.<feature>_title)) },
        bottomBar = { /* only if the screen owns one */ },
        snackbarHost = { SnackbarHost(hostState) },
    ) { innerPadding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(innerPadding)
                .padding(horizontal = Spacing.lg),
        ) {
            // content
        }
    }
}
```

**Rules**
- **`Scaffold` is the root of every screen.** It owns insets, top bar, bottom bar, snackbar.
- **Consume `innerPadding`.** Ignoring it means content hides behind the status/navigation bar.
- **Never** apply `WindowInsets` manually when `Scaffold` already handles it.
- **Edge-to-edge** is enabled at the Activity (`enableEdgeToEdge()`), and `Scaffold` handles the insets.
- **One scroll container per screen.** `Column(verticalScroll)` or `LazyColumn`, not both.
- **Loading / empty / error / content** are four explicit states of the same screen — render all four, do not leave empty/error to chance.

### 11.2 Navigation

- **Navigation Compose** with a single `NavHost` in `AppNavHost.kt`.
- **Routes are typed** (`@Serializable` data objects/classes) — no string route concatenation.
- **Arguments are typed** and passed as route parameters, not via a global singleton.
- **The `NavHost` never contains UI.** It maps route → `*Route` composable.
- **Back behaviour is explicit.** Every screen that can be left must handle the system back correctly; a screen that traps back is a defect.
- **Deep links** are declared on the destination, not parsed by hand in the screen.
- **No navigation from a `*Screen` composable.** The screen emits an event; the route/ViewModel decides to navigate.
- **Bottom navigation** uses `NavigationBar` + `NavigationBarItem`, with `selected` and `onClick` wired to the current destination. Never a custom `Row` of icons.

### 11.3 Top app bar

- Use `TopAppBar` / `CenterAlignedTopAppBar` from M3.
- **Title** is `titleLarge`, single line, ellipsized.
- **Navigation icon** (back / menu) is an `IconButton` with a `contentDescription`.
- **Actions** are `IconButton`s, max 3 visible; overflow goes into a `DropdownMenu`.
- **Never** build a top bar from a `Row` + `Text`.

---

## 12. Build and verification

### 12.1 Commands

```bash
# Compile + assemble debug
./gradlew :app:assembleDebug

# Unit tests
./gradlew :app:testDebugUnitTest

# Instrumented tests (needs a device/emulator)
./gradlew :app:connectedDebugAndroidTest

# Lint — must be clean, warnings are not "fine"
./gradlew :app:lintDebug

# Static analysis (if configured)
./gradlew detekt
./gradlew ktlintCheck

# Release build — must be exercised, not just debug
./gradlew :app:assembleRelease

# Dependency / API surface check
./gradlew :app:dependencies
```

### 12.2 Definition of done for any UI change

A UI change is **not done** until all of these pass:

- [ ] `./gradlew :app:assembleDebug` succeeds
- [ ] `./gradlew :app:lintDebug` reports **zero** new issues
- [ ] `./gradlew :app:testDebugUnitTest` passes
- [ ] No `Color(0x`, no off-grid `.dp`, no inline `fontSize`, no inline `RoundedCornerShape` in the diff
- [ ] No string literals in composables
- [ ] Every new interactive element has a `contentDescription` or `onClickLabel`
- [ ] Every new interactive element is ≥48dp
- [ ] Light **and** dark previews exist for every new shared component
- [ ] Checked at **200% font scale** — no clipping
- [ ] Checked with **TalkBack** — every control is reachable and announced
- [ ] `./gradlew :app:assembleRelease` succeeds (release path must not rot)

### 12.3 Release build hygiene

- `isMinifyEnabled = true` and `isShrinkResources = true` for release.
- `proguard-rules.pro` must be **exercised** — a release build that has never run is a release build that is broken.
- No `debugImplementation` leaking into release.
- No logging of tokens, headers, or PII in release.

---

## 13. Review checklist (paste into every PR)

```
[ ] Colors: only MaterialTheme.colorScheme / theme tokens — zero Color(0x
[ ] Spacing: only Spacing.* tokens — zero off-grid .dp
[ ] Type: only MaterialTheme.typography.* — zero inline fontSize/lineHeight
[ ] Shape: only MaterialTheme.shapes.* — zero inline RoundedCornerShape
[ ] Components: M3 component used where one exists; no re-implementations
[ ] State: screen is stateless; one UiState; events up, state down
[ ] Lists: stable keys on every Lazy* item
[ ] A11y: 48dp targets, contentDescriptions, contrast checked in light + dark
[ ] Strings: all in strings.xml, no literals in composables
[ ] Navigation: typed routes, no nav from *Screen
[ ] Anti-patterns: no gradients, no glassmorphism, no emoji icons, no runBlocking
[ ] Build: assembleDebug + lintDebug + testDebugUnitTest + assembleRelease all pass
[ ] Previews: light + dark for every new shared component
```

---

## 14. When you are unsure

1. **Look at an existing screen that already does it right** and copy that pattern.
2. **Look for the token.** If the value you want is not in `Dimens.kt` / `Color.kt` / `Type.kt` / `Shape.kt`, the answer is usually "add the token", not "inline the value".
3. **Prefer the boring M3 default.** A standard `Button` beats a clever custom one every time.
4. **If a rule blocks a genuine product requirement**, do not silently break it — add the token or the component to the design system, document why, and note it in the PR.
5. **Never invent a color, a spacing value, or a component to make a screenshot look good.** The design system is the source of truth, not the mockup.

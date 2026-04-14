# Frontend Constitution — Mizuho React Native 0.83 Standard

> Stack-specific constitution for the **mobile app** (`frontend/`). Aligned with **Mizuho React Native** company standard (`guideline/project-structure.md`, `guideline/techstack.md`, `guideline/coding-conventions.md`).
> Load this together with [constitution.md](constitution.md) (shared § I SDD + Governance) when working on frontend code.
> Every Agent that touches `frontend/` MUST read this before producing output. Deviations require an explicit amendment (see Governance in root `constitution.md`).

---

## Core Principles

> § I — Spec-Driven Development is **shared** and lives in the root [constitution.md](constitution.md). All frontend changes must follow it.

### II. React Native + Hermes + TypeScript Strict (NON-NEGOTIABLE)

- **React Native 0.83.x** on **Hermes** JS engine.
- **React 19.x** — function components + hooks only; no class components for new code.
- **TypeScript 5.8+** — explicit types for public APIs (props, store shapes, navigation params, API payloads).
- `any` is **disallowed** by ESLint (`@typescript-eslint/no-explicit-any`). Use `unknown` + narrow, or define a proper type.
- **Node.js ≥ 20** (pinned in `package.json` engines).
- **Package manager: yarn** (not npm). Use `yarn` for install/scripts.
- Target platforms: **Android API 24+** (Android 7.0), **iOS 15.1+**.

### III. Navigation — React Navigation 7 (NON-NEGOTIABLE)

- **React Navigation 7**: native stack + bottom tabs + nested stacks.
- Root stack: `src/navigators/index.tsx` registers almost all stack screens + wraps `NavigationContainer`.
- Bottom tabs: `src/navigators/tabs/MainTabs.tsx` — 5 tabs (Home, Asset, Analysis, Journal, Content), each a nested stack.
- Per-tab stacks: `src/navigators/stacks/<Name>Stack.tsx`.
- **Param lists MUST be typed** in `src/types/react-navigation.ts` (`ScreenName`, `NativeStackNavigationProp`, param lists).
- Adding a new screen: register in the appropriate stack or root navigator AND update `react-navigation.ts`.

### IV. State Management — Zustand (NON-NEGOTIABLE)

- Global client state uses **Zustand** only. No Redux, no MobX, no Context-as-store.
- Stores in `src/store/use<Name>Store.ts` (camelCase with `use` prefix).
- Split by **domain** — `useAuthStore`, `useLanguageStore`, `useTutorialStore`. Avoid one oversized store.
- **Immutable updates** — use Zustand's `set` with spread or immer middleware.
- Persisted slices via AsyncStorage where configured.
- If server/cache layer needed later: adopt **TanStack Query** and document the pattern in this file.

### V. Forms — React Hook Form + yup (NON-NEGOTIABLE for any form)

- **React Hook Form** for all forms.
- **yup** via `@hookform/resolvers` for validation schemas.
- `schema.ts` sibling to screen's `index.tsx` (e.g. `src/pages/LoginScreen/schema.ts`).
- Surface validation errors in UI — **no silent failures**.

### VI. i18n — i18next (NON-NEGOTIABLE for user-facing strings)

- **i18next + react-i18next** for all user-facing strings.
- Locales: `src/i18n/locales/en.json` + `src/i18n/locales/ja.json` — **both locales required for every new key**.
- Bootstrap: side-effect `import 'src/i18n'` in `App.tsx` before first paint.
- Keys **stable + namespaced** as project grows.
- Do NOT duplicate English literals across screens — always reference via `t("key.path")`.

### VII. HTTP — axios Instance (differs from web SPA conventions)

- **Single axios instance** configured in `src/config/axios.ts` and **exported as default**.
- Initialized via `initAxios()` called in `App.tsx`'s `useEffect`.
- Screens/hooks **import the instance directly** — `import api from 'src/config/axios'`.
- **There is NO separate `apiClient` wrapper abstraction**. This differs from web-SPA conventions.
- No `fetch`, no other HTTP clients (WebSocket/native fetch OK for streaming when documented).
- Network-layer concerns (baseURL, interceptors, auth header injection, 401 refresh) live inside `src/config/axios.ts`.

### VIII. Styling — StyleSheet.create ONLY (NON-NEGOTIABLE)

- **`StyleSheet.create`** for static styles.
- **NO Tailwind, NO styled-components, NO emotion, NO CSS-in-JS of any kind.**
- Avoid large inline style objects on **hot paths** (FlatList rows, Animated components).
- Reuse spacing/colors via `src/theme.ts` design tokens (`colors`, `fontSize`, `getSizePx`, ...).

### IX. Lists Performance

- Prefer **`@shopify/flash-list`** for long or scroll-heavy lists when UX fits.
- For `FlatList` / `SectionList`: always provide a stable **`keyExtractor`**; use **`getItemLayout`** when item size is fixed.
- **Memoize row components** (`React.memo` with stable props) when profiling shows unnecessary re-renders.

### X. Folder Structure (AUTHORITATIVE)

Layout matches Mizuho React Native exactly. **Two component roots** by design (primitives at repo root, composites under `src/`):

```
frontend/                          # mobile app root (inside monorepo)
├── App.tsx                        # i18n + initAxios + initAnalytics → root navigator
├── assets/                        # fonts, icons, images (require())
├── components/                    # PRIMITIVES — design-system layer at repo root
│   ├── elements/                  # Button, Text, Input, SearchBar, Dropdown, …
│   ├── templates/                 # e.g. OpenAccountTemplate
│   └── LayoutPublic/              # shared layouts
└── src/
    ├── components/                # COMPOSITE shared widgets (CardAsset, CustomTabBar, BottomSheet, domain folders)
    ├── config/
    │   ├── axios.ts               # HTTP client — SINGLE instance (§ VII)
    │   └── analytics/             # GA4 + Karte, initAnalytics()
    ├── data/                      # small shared datasets
    ├── hooks/                     # shared hooks (useAnalytics, …)
    ├── i18n/
    │   ├── index.ts               # i18next bootstrap
    │   └── locales/{en,ja}.json   # BOTH locales mandatory
    ├── navigators/
    │   ├── index.tsx              # root stack + NavigationContainer
    │   ├── NavigationBar.tsx
    │   ├── tabs/MainTabs.tsx
    │   └── stacks/{Home,Asset,Analysis,Journal,Content}Stack.tsx
    ├── pages/                     # SCREENS — feature-first (§ see below)
    ├── store/                     # Zustand stores (use*Store.ts, domain-split)
    ├── theme.ts                   # design tokens
    ├── types/
    │   ├── react-navigation.ts    # ScreenName + param lists (§ III)
    │   └── *.d.ts                 # ambient module declarations
    └── utils/                     # shared helpers
```

**`src/pages/<Name>Screen/` feature-first pattern:**

```
pages/ExampleScreen/
├── index.tsx              # Screen component (route target)
├── schema.ts              # yup + RHF validation (§ V) — optional
├── hook.ts                # screen-specific hooks — optional
├── components/            # UI only used on this screen
├── modules/               # larger composed sections — optional
├── data/                  # mocks / static slice — optional
└── SubFeatureScreen/      # nested sub-screens — optional
```

**Rules:**

- **New primitive** (Button, Input, Text variant) → `components/elements/<Name>.tsx` (repo root).
- **New reusable composite** (CardAsset, BottomSheet) → `src/components/<Name>.tsx` (or domain subfolder).
- **New screen** → `src/pages/<Name>Screen/index.tsx` + register in stack + update `react-navigation.ts`.
- **UI only for that screen** → `src/pages/<Name>Screen/components/<Part>.tsx`.

Path aliases via `babel-plugin-module-resolver` (`root: ["."]`): `components/elements/Button` resolves to repo-root `components/`, `src/...` resolves under `src/`.

### XI. Components — Function + Hooks Only (NON-NEGOTIABLE)

- **Function components only** — NO class components for new code.
- Follow **Rules of Hooks** (ESLint: `react-hooks/rules-of-hooks` enforced).
- Prefer **custom hooks** (`useX`) for non-trivial logic; keep presentational components thin.
- **`react-hooks/exhaustive-deps`** warnings must be **fixed** or documented with a one-line reason comment.
- No side effects during render (React 19 strictness).

### XII. Testing

- **Jest** + **react-test-renderer** for unit + component tests.
- Test **behavior and user-visible outcomes** — avoid testing implementation details only.
- Test files sibling or under `__tests__/` folder.
- Run `yarn test` before PR.

### XIII. Linting & Formatting (NON-NEGOTIABLE)

- **ESLint** + `@react-native/eslint-config` — zero warnings in CI.
- **Prettier** integrated via `eslint-plugin-prettier` — never fight formatting.
- `yarn lint` + `yarn check-types` **must pass** before PR.
- Auto-fix: `yarn lint:fix`.
- No `// eslint-disable-next-line` without a one-line justification comment.

### XIV. Imports (ESLint-enforced)

- `import/order` — grouped + consistent ordering.
- `import/no-duplicates` — no duplicate imports of the same module.
- `import/newline-after-import` — newline after import block.
- Prefer **shallow imports**; use path aliases from `babel-plugin-module-resolver`.

### XV. Images, Safe Area, Splash

- Remote images: **`@d11/react-native-fast-image`**.
- Safe areas: **`react-native-safe-area-context`** (`SafeAreaProvider`, `useSafeAreaInsets`).
- Boot splash: **`react-native-bootsplash`** — integrate per existing app pattern.
- Hide splash after navigation + auth hydration ready.

### XVI. Performance & Native Stack

- **Hermes** is the default JS engine — profile on real devices, not simulator.
- **New Architecture** (Fabric + TurboModules) — follow official RN upgrade guides when bumping version.
- Avoid patterns that **block the UI thread** (synchronous heavy work in render, huge JSON.parse).
- **No `console.log`** in production branches — use project logger (or strip via babel plugin in release).

### XVII. Security (Mobile)

- Do **not commit secrets** (API keys, private URLs). Use env files + `react-native-config` + native config.
- Prefer **secure storage** for tokens: `react-native-keychain` or equivalent. Adopt explicitly when introducing auth.
- **Validate + sanitize** navigation params and **deep link** payloads before use (deep links are an attack surface).
- `.env*` files must be gitignored.

### XVIII. Build Variants & Platforms

- **Android flavors**: `devDebug`, `stagingDebug`, `productDebug` + corresponding release variants (in `android/app/build.gradle`).
- **iOS schemes**: `MizuhoDev`, `MizuhoStaging`, `MizuhoProd` (in `ios/*.xcscheme`).
- **Fastlane** lanes: `yarn build:*`, `yarn distribute:*` (Ruby / Bundler in `ios/`).
- **Env per flavor** via `react-native-config`.

### XIX. Git Commits — Conventional Commits

- Format: `type(scope): short description`.
- Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `ci`, `perf`.
- Scope optional but encouraged: `feat(journal): add category filter`.

### XX. Naming

| Kind | Convention | Example |
|---|---|---|
| Screen folders | PascalCase + `Screen` suffix | `OpeningNisaAccountScreen/` |
| Component files | PascalCase | `AssetFilterBar.tsx` |
| Hooks / stores | camelCase with `use` prefix | `useLanguageStore.ts`, `useAuthStore.ts` |
| Schemas / utilities | camelCase or lowercase | `schema.ts`, helpers |
| Variables, functions | camelCase | `handleSubmit`, `isLoading` |
| Components | PascalCase | `MenuListItem` |

**Do not mix kebab-case and PascalCase** in the same folder. This repo uses PascalCase for UI files.

---

## Technology Stack (Locked)

| Category | Technology | Version |
|---|---|---|
| Framework | React Native | 0.83.x |
| UI library | React | 19.x |
| Language | TypeScript | 5.8+ |
| JS engine | Hermes | (RN default) |
| Node | Node.js | ≥ 20 |
| Package manager | yarn | (not npm) |
| Navigation | @react-navigation/* | 7.x |
| State | Zustand | latest |
| Forms | react-hook-form + yup + @hookform/resolvers | — |
| i18n | i18next + react-i18next | — |
| HTTP | axios | — |
| Lists | @shopify/flash-list | — |
| Images | @d11/react-native-fast-image | — |
| SVG | react-native-svg + react-native-svg-transformer | — |
| Env / flavors | react-native-config | — |
| Analytics | @react-native-firebase/* + @react-native-karte/core | — |
| Safe area | react-native-safe-area-context | — |
| Splash | react-native-bootsplash | — |
| Storage | @react-native-async-storage/async-storage | — |
| Testing | Jest + react-test-renderer | — |
| Linting | ESLint + @react-native/eslint-config + eslint-plugin-prettier | — |
| Build | Metro (dev), Fastlane (ship) | — |

---

## Mandatory Imports / Patterns

- Every user-facing string → `useTranslation()` from `react-i18next` + key in `en.json` AND `ja.json`.
- Every external HTTP → the axios instance from `src/config/axios.ts`.
- Every navigation → typed via `NativeStackNavigationProp<RootStackParamList, 'ScreenName'>`.
- Every global state → Zustand store under `src/store/`.
- Every form → React Hook Form + yup schema.
- Every list > 20 items → FlashList (or FlatList with keyExtractor + getItemLayout).

---

## Auto-Reject Patterns in Review

Block PR merge if ANY of these is found:

- `any` in new TypeScript code (use `unknown` + narrow).
- A class component.
- A new screen that isn't registered in a stack + typed in `react-navigation.ts`.
- Any CSS file, styled-components, Tailwind import, emotion.
- Inline style on a FlatList row (perf anti-pattern).
- Bypass of `@shopify/flash-list` for a 100+ item list without documented reason.
- Secrets committed (`.env*` not in `.gitignore`).
- `console.log` in production-bound branch.
- New Zustand store that is not domain-split (one mega-store anti-pattern).
- Translation key missing in `ja.json` (must ship en + ja together).
- Navigation without typed param list.
- `fetch(...)` or a second HTTP client (everything goes through the axios instance).

---

## Commands Reference

```bash
cd frontend
yarn install

# Metro + platform
yarn start                         # Metro bundler
yarn android:dev                   # Android devDebug
yarn android:staging               # Android stagingDebug
yarn android:prod                  # Android productDebug
yarn ios:dev                       # iOS MizuhoDev scheme
yarn ios:staging
yarn ios:prod

# Quality gates
yarn lint
yarn lint:fix
yarn check-types                   # tsc --noEmit
yarn test                          # Jest

# Ship
yarn build:android:staging         # Fastlane
yarn distribute:android:staging
```

---

## Governance

Governance rules are **shared** — see [constitution.md § Governance](constitution.md). Amendments to this file follow the same process.

**Version**: 1.0.0 (frontend) | **Ratified**: 2026-04-14 | **Aligned with** Mizuho React Native guideline (RN 0.83 / React 19 / TS 5.8+)

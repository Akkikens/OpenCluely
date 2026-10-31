# Backstage Phase 1 (Foundation) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn Pluely 0.1.9 into a renamed, phone-home-free, keychain-backed, Developer-ID-signed macOS app that launches with zero outbound connections at idle.

**Architecture:** A new git repo seeded from Pluely's history. Rust side loses `activate.rs`, the hosted paths in `api.rs`, PostHog, machine-uid, the updater, and the license gate. Frontend loses the license/updater/analytics components and reads API keys from the macOS keychain instead of localStorage. Build config gains an entitlements file and a signing identity.

**Tech Stack:** Tauri 2, Rust 1.94, React 19 + TypeScript 5.8, Vite 7, Tailwind 4, tauri-plugin-keychain 2, tauri-plugin-sql (sqlite), vitest (added here), Xcode 27 codesign.

**Spec:** `/Users/akshay/Documents/GitHub/OpenCluely/docs/superpowers/specs/2026-10-27-merged-copilot-design.md`

Later phases (Ears, Brain, Stealth and coding, Memory) get their own plans once this one ships; each is a separate subsystem in the spec.

## Global Constraints

- New repo path: `/Users/akshay/Documents/GitHub/backstage`. Source of seed: `/Users/akshay/Documents/GitHub/pluely` at commit `dc35e93`.
- Product name `Backstage`, bundle identifier `co.climbtogether.backstage`, version `0.1.0`. Rename is Akshay's to change later; nothing else may hardcode the name.
- License stays GPL-3.0. `LICENSE` unchanged; `NOTICE.md` credits Pluely (Srikanth Nani) and OpenCluely (TechyCSR).
- macOS only is tested. `minimumSystemVersion` = `14.2`. Windows/Linux `cfg` code stays and must keep compiling (`cargo check` with no target flag is enough).
- No network call the user did not configure. Forbidden identifiers anywhere under `src/` and `src-tauri/src/`: `posthog`, `PAYMENT_ENDPOINT`, `APP_ENDPOINT`, `API_ACCESS_KEY`, `POSTHOG_API_KEY`, `machine_uid`, `tauri_plugin_updater`, `pluely.com`, `validate_license`, `activate_license`.
- API keys never touch localStorage, SQLite, or logs. They live in the keychain under service `co.climbtogether.backstage`.
- Signing identity: `Developer ID Application: AKSHAY KALAPGAR (JFNPZ2JF6V)`. Hardened runtime on. No app sandbox. `macOSPrivateApi` stays `true`.
- Commands: `npm test` runs vitest; `cargo test` runs in `src-tauri`; `npm run tauri build -- --bundles app,dmg` produces the signed app.

## Review Focus

1. A machine that already ran Pluely has provider configs in localStorage with `variables.API_KEY` filled in. First launch of Backstage must move those keys into the keychain and blank them in localStorage, not lose them. (Test in Task 5.)
2. The keychain can refuse (user clicks Deny on the prompt, or the item is missing). Settings must still load with an empty key and the app must not throw at startup. (Test in Task 5.)
3. Ollama and other local providers use plain `http://localhost` or `http://127.0.0.1`. Narrowing the HTTP capability must keep those working while dropping `http://**`. (Test in Task 6.)
4. With `LicenseState` gone, `move_window` and custom shortcut actions must register unconditionally. Previously they silently skipped registration when the license was inactive. (Test in Task 3.)
5. Removing the updater and `createUpdaterArtifacts` must not leave `tauri.conf.json` referencing the `updater` plugin, or `tauri build` fails late, after a full compile. (Test in Task 7.)

---

### Task 1: Seed the repo and rename

**Files:**
- Create: `/Users/akshay/Documents/GitHub/backstage` (clone of pluely with history)
- Modify: `package.json` (name, version, description, homepage, repository, bugs)
- Modify: `src-tauri/Cargo.toml` (`name = "backstage"`, `lib.name = "backstage_lib"`, version, authors, repository, homepage)
- Modify: `src-tauri/src/main.rs` (`backstage_lib::run()`)
- Modify: `src-tauri/tauri.conf.json` (`productName`, `version`, `identifier`, window `title`)
- Modify: `index.html` `<title>`
- Create: `NOTICE.md`
- Modify: `README.md` (replace with a 20-line description: what it is, upstream credits, build commands)

**Interfaces:**
- Produces: a repo where `cargo check` and `npm run build` pass under the new names. Every later task works in this repo.

- [ ] **Step 1: Clone with history**

```bash
git clone /Users/akshay/Documents/GitHub/pluely /Users/akshay/Documents/GitHub/backstage
cd /Users/akshay/Documents/GitHub/backstage && git remote remove origin && git checkout -b main
```

- [ ] **Step 2: Rename in the files listed above**

`identifier` is `co.climbtogether.backstage`. The SQLite preload string in `tauri.conf.json` and `lib.rs` becomes `sqlite:backstage.db`. `grep -n -i pluely package.json src-tauri/Cargo.toml src-tauri/tauri.conf.json src-tauri/src/main.rs src-tauri/src/lib.rs index.html` must return only the attribution comment you add at the top of `src-tauri/src/lib.rs` (one line: `// Derived from Pluely 0.1.9 (GPL-3.0) by Srikanth Nani. See NOTICE.md.`). Pluely-branded UI copy under `src/` is swept in Task 4, not here.

- [ ] **Step 3: Write `NOTICE.md`**

Three paragraphs: derived from Pluely 0.1.9 at `dc35e93` under GPL-3.0 with a link to the upstream repo; algorithms adapted from OpenCluely (Apache 2.0, TechyCSR) with a link; the combined work is GPL-3.0.

- [ ] **Step 4: Verify both builds**

Run: `cd src-tauri && cargo check 2>&1 | tail -1`
Expected: `Finished` line, no errors.
Run: `npm install && npm run build 2>&1 | tail -2`
Expected: `✓ built in …` from Vite, `tsc` silent.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "chore: seed Backstage from Pluely 0.1.9, rename identifiers, add NOTICE"
```

---

### Task 2: Test harness and the phone-home guard test

**Files:**
- Modify: `package.json` (add `vitest` devDependency, `"test": "vitest run"`)
- Create: `vitest.config.ts` (environment `node`, include `tests/**/*.test.ts`)
- Create: `tests/no-phone-home.test.ts`
- Create: `src-tauri/src/lib.rs` test module stub is not needed; `cargo test` already runs with zero tests.

**Interfaces:**
- Produces: `npm test` and `cargo test`. `tests/no-phone-home.test.ts` exports nothing; it is the gate every later removal task runs against.

- [ ] **Step 1: Write the failing test**

```ts
// tests/no-phone-home.test.ts
import { execSync } from "node:child_process";
import { describe, it, expect } from "vitest";

const FORBIDDEN = [
  "posthog", "PAYMENT_ENDPOINT", "APP_ENDPOINT", "API_ACCESS_KEY",
  "POSTHOG_API_KEY", "machine_uid", "tauri_plugin_updater", "pluely.com",
  "validate_license", "activate_license",
];

describe("no phone-home code", () => {
  for (const word of FORBIDDEN) {
    it(`source tree does not mention ${word}`, () => {
      let out = "";
      try {
        out = execSync(
          `grep -rn --include=*.ts --include=*.tsx --include=*.rs --include=*.toml --include=*.json -i "${word}" src src-tauri/src src-tauri/Cargo.toml src-tauri/tauri.conf.json src-tauri/capabilities package.json`,
          { encoding: "utf8" }
        );
      } catch (e: any) { out = e.stdout ?? ""; }
      expect(out.trim()).toBe("");
    });
  }
});
```

- [ ] **Step 2: Run it to verify it fails**

Run: `npm test`
Expected: FAIL on every forbidden word (they all exist in the tree now).

- [ ] **Step 3: Commit the harness**

```bash
git add package.json package-lock.json vitest.config.ts tests/no-phone-home.test.ts
git commit -m "test: add vitest and phone-home guard test (currently failing by design)"
```

---

### Task 3: Strip phone-home code from the Rust side

**Files:**
- Delete: `src-tauri/src/activate.rs`
- Modify: `src-tauri/src/api.rs` — delete `get_app_endpoint`, `get_api_access_key`, `get_secure_storage_path`, `get_stored_credentials`, `transcribe_audio`, `fetch_api_response_config`, `perform_user_audio_transcription`, `chat_stream_response`, `fetch_models`, `fetch_prompts`, `create_system_prompt`, `check_license_status`, `get_activity`, `user_activity`, `report_api_error`, and the `MachineUidExt` import. Keep `decode_audio_base64` and `map_api_error_message` only if something still calls them; otherwise delete and let the file be removed entirely.
- Modify: `src-tauri/src/lib.rs:37-117` — remove the `posthog_init`, `tauri_plugin_updater`, `tauri_plugin_machine_uid` plugin lines, the `LicenseState` manage line, and every `activate::*` and `api::*` entry from `generate_handler!`; remove `mod activate;` (and `mod api;` if the file is gone).
- Modify: `src-tauri/src/shortcuts.rs:33-51,128-135,333-345,504-515` — delete `LicenseState`, `set_license_status`, and both `is_active()` gates so `move_window` and custom actions always register.
- Modify: `src-tauri/build.rs` — body becomes only `tauri_build::build()`; drop the `dotenv` build-dependency.
- Modify: `src-tauri/Cargo.toml` — remove `tauri-plugin-updater`, `tauri-plugin-posthog`, `tauri-plugin-machine-uid`, `dotenv` (both sections).
- Modify: `src-tauri/tauri.conf.json` — remove `plugins.updater`, set `bundle.createUpdaterArtifacts` to `false`.
- Modify: `src-tauri/capabilities/default.json` and `cross-platform.json` — remove `updater:default`, `posthog:default`, `posthog:allow-capture`.
- Test: `src-tauri/src/shortcuts.rs` (new `#[cfg(test)] mod tests`)

**Interfaces:**
- Consumes: nothing.
- Produces: `shortcuts::update_shortcuts` registers `move_window` unconditionally. Frontend `invoke("set_license_status")`, `invoke("validate_license_api")` and the other removed commands no longer exist; Task 4 removes their callers.

- [ ] **Step 1: Write the failing Rust test for the gate removal**

Extract the decision "should this action register" into a pure function so it is testable:

```rust
// in shortcuts.rs
pub fn should_register_action(action: &str, enabled: bool) -> bool
```
Semantics: returns `enabled`; `action` is accepted for logging only. The point is that no license input exists in the signature.

```rust
#[cfg(test)]
mod tests {
    use super::should_register_action;
    #[test]
    fn move_window_registers_when_enabled() {
        assert!(should_register_action("move_window", true));
    }
    #[test]
    fn disabled_action_does_not_register() {
        assert!(!should_register_action("move_window", false));
    }
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `cd src-tauri && cargo test should_register 2>&1 | tail -3`
Expected: compile error, function not found.

- [ ] **Step 3: Make the deletions listed in Files, and route both former gates through `should_register_action`**

- [ ] **Step 4: Verify**

Run: `cd src-tauri && cargo test 2>&1 | tail -3`
Expected: `test result: ok. 2 passed`.
Run: `cd src-tauri && cargo check 2>&1 | grep -c warning`
Expected: no `unused import` warnings for removed crates (any count is fine as long as the build is clean of errors).

- [ ] **Step 5: Commit**

```bash
git add -A src-tauri && git commit -m "feat(rust): remove license, hosted API, PostHog, machine-uid and updater"
```

---

### Task 4: Strip phone-home code from the frontend

**Files:**
- Delete: `src/lib/analytics.ts`, `src/lib/functions/pluely.api.ts`, `src/components/GetLicense.tsx`, `src/components/Promote.tsx`, `src/components/updater/index.tsx`, `src/pages/dashboard/components/PluelyApiSetup.tsx`, `src/pages/system-prompts/PluelyPrompts.tsx`
- Modify: `src/components/index.ts` (drop the exports), `src/components/DragButton.tsx`, `src/pages/dashboard/index.tsx`, `src/pages/system-prompts/Generate.tsx`, `src/pages/app/components/completion/Audio.tsx`, `src/pages/app/components/speech/index.tsx`, `src/pages/chats/components/ChatAudio.tsx`, `src/pages/chats/components/View.tsx`, `src/pages/responses/index.tsx` and its `components/AutoScrollToggle.tsx`, `LanguageSelector.tsx`, `ResponseLength.tsx`, `src/pages/screenshot/components/ScreenshotConfigs.tsx`, `src/pages/settings/components/Theme.tsx`, `src/pages/shortcuts/components/shortcuts/ShortcutManager.tsx`, `src/hooks/useChatCompletion.ts`, `src/hooks/useMenuItems.tsx`, `src/hooks/useSettings.ts`, `src/contexts/app.context.tsx`, `src/types/context.type.ts`, `src/types/settings.hook.ts`, `src/config/constants.ts`
- Modify: `src-tauri/capabilities/*.json` — nothing further (done in Task 3).

**Interfaces:**
- Consumes: Task 3's removal of the Rust commands.
- Produces: `AppContext` no longer has `hasActiveLicense`, `pluelyApiEnabled`, `setPluelyApiEnabled`, or `getActiveLicenseStatus`. Every place that gated on `hasActiveLicense` now behaves as if it were `true`. `STORAGE_KEYS.PLUELY_API_ENABLED` is gone.

- [ ] **Step 1: Confirm the guard test still fails on the frontend words only**

Run: `npm test 2>&1 | grep -E "✓|×" | head -12`
Expected: Rust-only words pass; `posthog`, `pluely.com`, `validate_license`, `activate_license` still fail.

- [ ] **Step 2: Make the deletions and edits**

Also replace remaining user-visible "Pluely" strings under `src/` with "Backstage"; after this task `grep -rn -i pluely src` returns nothing.

Rule for each gated branch: keep the licensed branch, delete the unlicensed one and the `GetLicense` call-to-action. Where a component was rendered only to upsell (`Promote`, `PluelyApiSetup`, `PluelyPrompts`, updater), remove the render site entirely. `useChatCompletion` and `useSettings` keep only the BYOK code path.

- [ ] **Step 3: Verify**

Run: `npm test`
Expected: all forbidden-word tests PASS.
Run: `npx tsc --noEmit && npm run build 2>&1 | tail -1`
Expected: no type errors; Vite `✓ built`.

- [ ] **Step 4: Commit**

```bash
git add -A src && git commit -m "feat(ui): remove license, hosted API, analytics and updater UI"
```

---

### Task 5: API keys in the keychain

**Files:**
- Create: `src/lib/storage/secrets.ts`
- Create: `src/lib/storage/secrets.migration.ts`
- Modify: `src/lib/storage/ai-providers.ts` and `src/lib/storage/stt-providers.ts` (or wherever `CUSTOM_SPEECH_PROVIDERS` / `SELECTED_STT_PROVIDER` are read) — keys resolved through `secrets.ts`
- Modify: `src/contexts/app.context.tsx` — call `migrateKeysToKeychain()` once at startup, before providers load
- Test: `tests/secrets.test.ts`, `tests/secrets.migration.test.ts`

**Interfaces:**
- Consumes: `tauri-plugin-keychain` JS API: `getItem(key): Promise<string | null>`, `saveItem(key, value): Promise<void>`, `removeItem(key): Promise<void>`.
- Produces:
  ```ts
  // src/lib/storage/secrets.ts
  export const SECRET_PREFIX = "co.climbtogether.backstage/";
  export function secretKeyFor(providerId: string, variable: string): string; // `${SECRET_PREFIX}${providerId}/${variable}`
  export async function getSecret(providerId: string, variable: string): Promise<string>; // "" when missing or on keychain error
  export async function setSecret(providerId: string, variable: string, value: string): Promise<void>;
  export async function removeSecret(providerId: string, variable: string): Promise<void>;
  export const SECRET_VARIABLES: readonly string[]; // ["API_KEY", "AUTH_TOKEN", "SECRET"]
  export function isSecretVariable(name: string): boolean; // case-insensitive match on SECRET_VARIABLES or suffix "_KEY"

  // src/lib/storage/secrets.migration.ts
  export async function migrateKeysToKeychain(storage: Pick<Storage, "getItem" | "setItem">, secrets: { setSecret: typeof setSecret }): Promise<number>; // returns count of values moved
  ```
  Provider objects in localStorage keep their `variables` map, but every secret variable's value is the literal string `"<keychain>"` after migration.

- [ ] **Step 1: Write the failing tests**

```ts
// tests/secrets.test.ts
import { vi, describe, it, expect, beforeEach } from "vitest";
const store = new Map<string, string>();
vi.mock("tauri-plugin-keychain", () => ({
  getItem: vi.fn(async (k: string) => store.get(k) ?? null),
  saveItem: vi.fn(async (k: string, v: string) => { store.set(k, v); }),
  removeItem: vi.fn(async (k: string) => { store.delete(k); }),
}));
import { getSecret, setSecret, removeSecret, secretKeyFor, isSecretVariable } from "../src/lib/storage/secrets";

beforeEach(() => store.clear());

describe("secrets", () => {
  it("round-trips a key under the namespaced name", async () => {
    await setSecret("openai", "API_KEY", "sk-test");
    expect(store.get(secretKeyFor("openai", "API_KEY"))).toBe("sk-test");
    expect(await getSecret("openai", "API_KEY")).toBe("sk-test");
  });
  it("returns empty string when missing", async () => {
    expect(await getSecret("openai", "API_KEY")).toBe("");
  });
  it("returns empty string when the keychain throws", async () => {
    const kc = await import("tauri-plugin-keychain");
    (kc.getItem as any).mockRejectedValueOnce(new Error("denied"));
    expect(await getSecret("openai", "API_KEY")).toBe("");
  });
  it("removes", async () => {
    await setSecret("groq", "API_KEY", "x"); await removeSecret("groq", "API_KEY");
    expect(await getSecret("groq", "API_KEY")).toBe("");
  });
  it("classifies secret variables", () => {
    expect(isSecretVariable("API_KEY")).toBe(true);
    expect(isSecretVariable("api_key")).toBe(true);
    expect(isSecretVariable("MY_KEY")).toBe(true);
    expect(isSecretVariable("MODEL")).toBe(false);
  });
});
```

```ts
// tests/secrets.migration.test.ts
import { describe, it, expect, vi } from "vitest";
import { migrateKeysToKeychain } from "../src/lib/storage/secrets.migration";

function fakeStorage(init: Record<string, string>) {
  const m = new Map(Object.entries(init));
  return { getItem: (k: string) => m.get(k) ?? null, setItem: (k: string, v: string) => { m.set(k, v); }, dump: () => Object.fromEntries(m) };
}

describe("migrateKeysToKeychain", () => {
  it("moves API_KEY values out of provider configs and blanks them", async () => {
    const s = fakeStorage({
      curl_custom_ai_providers: JSON.stringify([{ id: "openai", variables: { API_KEY: "sk-1", MODEL: "gpt" } }]),
      curl_selected_ai_provider: JSON.stringify({ provider: "openai", variables: { API_KEY: "sk-1", MODEL: "gpt" } }),
    });
    const setSecret = vi.fn(async () => {});
    const moved = await migrateKeysToKeychain(s, { setSecret });
    expect(moved).toBe(2);
    expect(setSecret).toHaveBeenCalledWith("openai", "API_KEY", "sk-1");
    const dump = s.dump();
    expect(JSON.parse(dump.curl_custom_ai_providers)[0].variables).toEqual({ API_KEY: "<keychain>", MODEL: "gpt" });
    expect(JSON.parse(dump.curl_selected_ai_provider).variables.API_KEY).toBe("<keychain>");
  });
  it("is idempotent", async () => {
    const s = fakeStorage({ curl_selected_ai_provider: JSON.stringify({ provider: "openai", variables: { API_KEY: "<keychain>" } }) });
    const setSecret = vi.fn(async () => {});
    expect(await migrateKeysToKeychain(s, { setSecret })).toBe(0);
    expect(setSecret).not.toHaveBeenCalled();
  });
  it("tolerates missing or malformed entries", async () => {
    const s = fakeStorage({ curl_custom_ai_providers: "not json" });
    expect(await migrateKeysToKeychain(s, { setSecret: vi.fn(async () => {}) })).toBe(0);
  });
});
```

- [ ] **Step 2: Run to verify they fail**

Run: `npm test -- secrets`
Expected: FAIL, modules not found.

- [ ] **Step 3: Implement `secrets.ts` and `secrets.migration.ts` with the signatures above**

Migration scans these localStorage keys: `curl_custom_ai_providers`, `curl_selected_ai_provider`, `curl_custom_speech_providers`, `curl_selected_stt_provider`. Each is either an array of `{id, variables}` or a single `{provider, variables}`.

- [ ] **Step 4: Wire the readers**

Wherever a provider's `variables` are expanded into a curl template (`src/lib/functions/common.function.ts` or the storage helpers that feed it), replace any `"<keychain>"` value with `await getSecret(providerId, variable)` at request time. The settings UI's key inputs call `setSecret` on change and display a masked placeholder when the stored value is `"<keychain>"`.

- [ ] **Step 5: Verify**

Run: `npm test && npx tsc --noEmit`
Expected: all PASS, no type errors.
Manual: `npm run tauri dev`, enter a key in settings, quit, run `security find-generic-password -s co.climbtogether.backstage/openai/API_KEY -w`.
Expected: the key prints; `localStorage` (Safari Web Inspector on the webview, or `grep -r "sk-" ~/Library/WebKit/co.climbtogether.backstage`) does not contain it.

- [ ] **Step 6: Commit**

```bash
git add src/lib/storage/secrets.ts src/lib/storage/secrets.migration.ts tests/secrets*.test.ts src && git commit -m "feat: store provider API keys in the macOS keychain with one-time migration"
```

---

### Task 6: Narrow capabilities and add a CSP

**Files:**
- Modify: `src-tauri/capabilities/default.json`, `src-tauri/capabilities/cross-platform.json`
- Modify: `src-tauri/tauri.conf.json` (`app.security.csp`)
- Test: `tests/capabilities.test.ts`

**Interfaces:**
- Produces: HTTP capability allows `https://**`, `http://localhost:*`, `http://127.0.0.1:*` only. CSP: `default-src 'self'; connect-src 'self' https: http://localhost:* http://127.0.0.1:* ipc: http://ipc.localhost; img-src 'self' data: blob: https:; style-src 'self' 'unsafe-inline'; script-src 'self'; font-src 'self' data:`.

- [ ] **Step 1: Write the failing test**

```ts
// tests/capabilities.test.ts
import { readFileSync } from "node:fs";
import { describe, it, expect } from "vitest";
const files = ["src-tauri/capabilities/default.json", "src-tauri/capabilities/cross-platform.json"];
describe("capabilities", () => {
  for (const f of files) {
    it(`${f} allows https and localhost http only`, () => {
      const cap = JSON.parse(readFileSync(f, "utf8"));
      const http = cap.permissions.find((p: any) => p?.identifier === "http:default");
      const urls = http.allow.map((a: any) => a.url).sort();
      expect(urls).toEqual(["http://127.0.0.1:*", "http://localhost:*", "https://**"]);
      expect(cap.permissions).not.toContain("updater:default");
      expect(cap.permissions).not.toContain("posthog:default");
    });
  }
  it("tauri.conf.json sets a CSP", () => {
    const conf = JSON.parse(readFileSync("src-tauri/tauri.conf.json", "utf8"));
    expect(typeof conf.app.security.csp).toBe("string");
    expect(conf.app.security.csp).toContain("connect-src");
  });
});
```

- [ ] **Step 2: Run to verify it fails** — `npm test -- capabilities` → FAIL.

- [ ] **Step 3: Edit the three files per Interfaces.**

- [ ] **Step 4: Verify** — `npm test` PASS; `npm run tauri dev`, configure Ollama at `http://localhost:11434` and a cloud provider, send one message to each.
Expected: both answer; the webview console shows no CSP violations.

- [ ] **Step 5: Commit** — `git commit -am "chore(security): narrow http capability to https + localhost, add CSP"`

---

### Task 7: Signed build

**Files:**
- Create: `src-tauri/Entitlements.plist`
- Modify: `src-tauri/info.plist` — remove the two `com.apple.security.*` keys (they belong in entitlements); keep usage strings; add `NSScreenCaptureUsageDescription` and `NSAudioCaptureUsageDescription` text for Backstage.
- Modify: `src-tauri/tauri.conf.json` — `bundle.macOS`: `minimumSystemVersion: "14.2"`, `entitlements: "Entitlements.plist"`, `signingIdentity: "Developer ID Application: AKSHAY KALAPGAR (JFNPZ2JF6V)"`, `hardenedRuntime: true`; `bundle.targets: ["app","dmg"]`; replace Pluely icons in `src-tauri/icons/` with a placeholder set generated by `npm run tauri icon <png>` from a plain 1024px PNG (any solid colour with a "B"; Akshay replaces later).
- Create: `scripts/verify-signature.sh`
- Test: `tests/build-config.test.ts`

**Interfaces:**
- Produces: `npm run tauri build -- --bundles app,dmg` emits `src-tauri/target/release/bundle/macos/Backstage.app` and a DMG, both signed. `scripts/verify-signature.sh <path.app>` exits 0 only if `codesign --verify --deep --strict` passes, the entitlements include `com.apple.security.cs.allow-jit`, `com.apple.security.device.audio-input`, and the hardened runtime flag is set.

- [ ] **Step 1: Write the failing config test**

```ts
// tests/build-config.test.ts
import { readFileSync, existsSync } from "node:fs";
import { describe, it, expect } from "vitest";
describe("build config", () => {
  const conf = JSON.parse(readFileSync("src-tauri/tauri.conf.json", "utf8"));
  it("targets macOS 14.2 with hardened runtime and entitlements", () => {
    expect(conf.bundle.macOS.minimumSystemVersion).toBe("14.2");
    expect(conf.bundle.macOS.hardenedRuntime).toBe(true);
    expect(conf.bundle.macOS.entitlements).toBe("Entitlements.plist");
    expect(conf.bundle.macOS.signingIdentity).toMatch(/^Developer ID Application: AKSHAY KALAPGAR/);
    expect(existsSync("src-tauri/Entitlements.plist")).toBe(true);
  });
  it("has no updater plugin or artifacts", () => {
    expect(conf.plugins?.updater).toBeUndefined();
    expect(conf.bundle.createUpdaterArtifacts).toBe(false);
  });
  it("entitlements do not include the app sandbox", () => {
    const ent = readFileSync("src-tauri/Entitlements.plist", "utf8");
    expect(ent).not.toContain("com.apple.security.app-sandbox");
    expect(ent).toContain("com.apple.security.cs.allow-jit");
    expect(ent).toContain("com.apple.security.device.audio-input");
  });
});
```

- [ ] **Step 2: Run to verify it fails** — `npm test -- build-config` → FAIL.

- [ ] **Step 3: Write `Entitlements.plist`**

Keys, all `true`: `com.apple.security.cs.allow-jit`, `com.apple.security.cs.allow-unsigned-executable-memory`, `com.apple.security.device.audio-input`, `com.apple.security.device.microphone`, `com.apple.security.cs.disable-library-validation`. No sandbox key.

- [ ] **Step 4: Edit `tauri.conf.json` and `info.plist`, generate icons, write `scripts/verify-signature.sh`**

The script: `codesign --verify --deep --strict --verbose=2 "$1"` then `codesign -d --entitlements :- "$1" | grep -q allow-jit` then `codesign -dvv "$1" 2>&1 | grep -q 'flags=.*runtime'`; `set -e` so any failure exits non-zero.

- [ ] **Step 5: Build and verify**

Run: `npm test` → PASS.
Run: `npm run tauri build -- --bundles app,dmg 2>&1 | tail -5`
Expected: `Finished 2 bundles at:` with `.app` and `.dmg` paths.
Run: `scripts/verify-signature.sh src-tauri/target/release/bundle/macos/Backstage.app && echo SIGNED`
Expected: `SIGNED`.
Run: `open src-tauri/target/release/bundle/macos/Backstage.app`
Expected: the overlay appears at top-centre of the main display with no Gatekeeper dialog.

- [ ] **Step 6: Commit**

```bash
git add src-tauri/Entitlements.plist src-tauri/info.plist src-tauri/tauri.conf.json src-tauri/icons scripts/verify-signature.sh tests/build-config.test.ts
git commit -m "build: Developer ID signing, hardened runtime entitlements, macOS 14.2 floor"
```

---

### Task 8: Idle-network proof (Phase 1 done criterion)

**Files:**
- Create: `scripts/verify-idle-network.sh`

**Interfaces:**
- Produces: `scripts/verify-idle-network.sh <path.app> [seconds=60]` launches the app, samples `lsof -i -a -p <pid>` every 2 s, kills the app, and exits 0 only if no TCP/UDP connection was ever observed. The spec's "zero outbound connections at idle" is this script passing.

- [ ] **Step 1: Write the script**

Launch with `open -n "$1"`, resolve the PID via `pgrep -n -f "Backstage.app/Contents/MacOS/"`, loop for `$2` seconds collecting `lsof -nP -i -a -p "$pid"` lines beyond the header into a temp file, `kill "$pid"` at the end, print the collected lines, exit 1 if the file is non-empty.

- [ ] **Step 2: Run it against the Task 7 build with no providers configured**

Run: `scripts/verify-idle-network.sh src-tauri/target/release/bundle/macos/Backstage.app 60 && echo QUIET`
Expected: `QUIET`. If any line appears, it names the host; find the caller and remove it (that is a Task 3 or 4 miss, fix there and re-run).

- [ ] **Step 3: Record the result in the README under "Verification" with the date, and commit**

```bash
git add scripts/verify-idle-network.sh README.md && git commit -m "test: idle-network verification script; Phase 1 done"
```

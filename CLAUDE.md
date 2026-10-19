# CLAUDE.md — cordova-plugin-inappbrowser (Paketti fork)

Fork of [apache/cordova-plugin-inappbrowser](https://github.com/apache/cordova-plugin-inappbrowser) adding
TLS **client-certificate** authentication to the Android in-app browser. It existed for the mPollux
(certificate-based SSO) login of the Paketti orderer app. System overview: `../pake-scripts/CLAUDE.md`.
**Currently unused by any released app** — keep it working, don't extend it.

## Branches and fork diff

Forked at upstream `master` `a162bd9` (2019-04-09, `3.1.0-dev`). Our branches are a linear chain, all
pushed to `origin`; the upstream release branches (`1.x`, `2.0.x`, `3.0.x`, `CB-*`) are also on `origin`.

| branch | head | what |
|---|---|---|
| `master` | `a162bd9` 2019-04 | = upstream |
| `sslerr` | `876077c` 2021-06 | client-cert prompt + SSL-error dialog, EMM-configured alias |
| `certprompt` | `d60c7dd` 2023-01 | EMM alias (`certAlias` app restriction) / in-memory alias, prompt fixes |
| **`simplified`** (checked out) | `89f6e91` 2023-04 | alias kept in SharedPreferences (`SP_KEY_ALIAS`) else `KeyChain.choosePrivateKeyAlias`; strips `certificatInstallerConfig_` alias prefix (Fujitsu bug) |

All changes are in `src/android/InAppBrowser.java` (`InAppBrowserClient`, "Certificate management" block):
`onReceivedClientCertRequest` → `AliasCallback` loads key + chain from `KeyChain`, checks validity, stores the
alias and `proceed`s. Any failure, and every `onReceivedSslError`, clears the stored alias, shows a Finnish
"Varmenneongelma" dialog and calls `System.exit(0)` — the SSL error is never proceeded.
Known bug: the SSL-error log line has an inverted null check (`error.getCertificate() != null ? "null" : …`),
which NPEs when the certificate is null. iOS/JS are untouched upstream code.

## Consumers

| app | spec | status |
|---|---|---|
| pake-specialist | `cordova/package.json` `#certprompt` (lock `d60c7dd`) | unreleased app, cordova-android 11 |
| pake-front | `#simplified`, removed in `e9c1e112` (2026-02-05, "Removal of mPollux authentication") | — |
| pake-transporter | removed in `a76dcf3` (2023-08-15) | — |

So nothing consumes the checked-out `simplified` branch any more.

## Toolchain

This repo is not built on its own — it is compiled inside the consuming app's Cordova Android project.

| tool | version | where pinned / why |
|---|---|---|
| cordova CLI | 13.0.0 | `npm install -g cordova@13.0.0` in the app's `do_init_*_mobile` (`pake-tools.sh`) |
| cordova-android | 14.0.1 (pake-front), 15.1.0 (pake-transporter), 11.0.0 (pake-specialist) | app `cordova/package.json` devDependencies |
| Gradle / AGP | 8.13 / 8.7.3 (cdv-android 14), 8.14.2 / 8.10.1 (cdv-android 15) | bundled by cordova-android; plugins must use `implementation`, not `compile` (Gradle 7+) |
| JDK | 21 (porter-front) / 18 (paketti-front packaging) | `export JAVA_HOME=$(/usr/libexec/java_home -v …)` in app `pake-tools.sh`; JDK 25 is too new for this Gradle |
| Android SDK | compileSdk 35 (front) / 36 (transporter), minSdk 24 | app `config.xml` + cordova-android defaults; `ANDROID_HOME` in `~/.zshenv` |
| node | nvm default (25.8.2) or 24.13.0 | no `.nvmrc`; `nvm use` lines in app `pake-tools.sh` are commented out |

## How a consuming app picks up a change

Consumers reference this repo as an npm git dependency in their `cordova/package.json`; their
`cordova/package-lock.json` **pins the exact commit SHA**, and `cordova/plugins/` + `cordova/platforms/` are
git-ignored copies. So pushing to the branch alone changes nothing in an app build.

1. Commit and push here, on the branch the consumer references (see table above).
2. In the consumer's `cordova/` dir (node from nvm — cordova CLI 13.0.0 is installed into node 24.13.0 / 25.8.2):
   `npm install <spec-from-package.json>` (e.g. `npm install github:KaufmannAgency/<repo>#<branch>`) to move
   the lock to the new commit, then `cordova plugin rm <plugin-id> && cordova plugin add <plugin-id>` to
   refresh `cordova/plugins/` and `platforms/android`. `pake init <module> --mobile --clean` also works but
   wipes `platforms/` (re-prompts for `pakeapp.keystore`).
   - Quick iteration without pushing: `cordova plugin add /Users/joni/Github/<repo>` (local path) —
     this rewrites the spec in `package.json`/`config.xml`, so restore the github spec before committing the app.
3. Build + install a debug APK on a USB device: `pake build,package,run <module> --mobile --named-env remote`
   (`-u <user>` sets `TEST_USER`, which **bypasses Imprivata**), then `pake log <module> --mobile` (adb logcat).
4. Commit the consumer's updated `cordova/package-lock.json` (and `package.json` if the spec changed).

`<module>` is `paketti-front` (repo `pake-front`) or `porter-front` (repo `pake-transporter`); the
`do_*_mobile` functions live in each app repo's `pake-tools.sh`. There are no tests in this repo.

The only remaining consumer is `pake-specialist` (no `pake-tools.sh`; plain `cordova` in
`pake-specialist/cordova`, cordova-android 11 → Gradle 7.4.2 / AGP 7.2.1, which need JDK 11–17 —
not installed on the current Mac). Testing needs an Android device with a client certificate in the
system KeyChain (normally pushed by the EMM).

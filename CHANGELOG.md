# Changelog

All notable changes to this project will be documented in this file.

## [0.4.1](https://github.com/inference-gateway/.github/compare/v0.4.0...v0.4.1) (2026-06-01)

### ♻️ Improvements

* commit message for infer.yml centralization ([e8a8791](https://github.com/inference-gateway/.github/commit/e8a879176ed106bcecb29630c752fa380f6b267b))

## [0.4.0](https://github.com/inference-gateway/.github/compare/v0.3.2...v0.4.0) (2026-06-01)

### ✨ Features

* **migrate-infer:** regenerate .infer config in the same PR ([#36](https://github.com/inference-gateway/.github/issues/36)) ([6ee8d2f](https://github.com/inference-gateway/.github/commit/6ee8d2f72ba942db027a65a4cfe627fd770a5fdf))

### ♻️ Improvements

* **migrate-infer:** provide infer CLI via Flox, not curl ([#37](https://github.com/inference-gateway/.github/issues/37)) ([485aba7](https://github.com/inference-gateway/.github/commit/485aba71d7094de6ef73708290e17ff62bdfa55d))

### 🐛 Bug Fixes

* **cleanup-runs:** fail loud on rate-limit + prune orphaned-workflow runs ([#34](https://github.com/inference-gateway/.github/issues/34)) ([56525f5](https://github.com/inference-gateway/.github/commit/56525f52c9445084b568cf4997a1716b596927e7))
* **cleanup-runs:** make keep_last a repo-wide floor, not per-workflow ([#35](https://github.com/inference-gateway/.github/issues/35)) ([17eaec9](https://github.com/inference-gateway/.github/commit/17eaec95faf3ea138b8acffafc171d9b4e0a46c6))
* **infer:** pin CLI version off infer-action's broken v0.115.2 default ([#39](https://github.com/inference-gateway/.github/issues/39)) ([32ca05d](https://github.com/inference-gateway/.github/commit/32ca05d0c2ea22ccc3c8328d266d77d6e05013a8))
* **migrate-infer:** bump each target's OWN Flox infer pin (bump-adl style) ([#38](https://github.com/inference-gateway/.github/issues/38)) ([7c074e0](https://github.com/inference-gateway/.github/commit/7c074e0ba27212711fef16176e0a6c930dc24b0f)), closes [36/#37](https://github.com/36/.github/issues/37)

### 👷 CI

* sweep all registered repos in cleanup-runs (incl. kind: none) ([#33](https://github.com/inference-gateway/.github/issues/33)) ([cda35d2](https://github.com/inference-gateway/.github/commit/cda35d24811af546810f5163c3b875bfc3bf5d86))

### 🔧 Miscellaneous

* remove scheduled job ([4d785de](https://github.com/inference-gateway/.github/commit/4d785de4ac874d0adbc5169c4504fd70a1908164))
* update migrate-claude.yml ([ed09050](https://github.com/inference-gateway/.github/commit/ed0905010331db3288069fbd5599f0b492b9ff86))

## [0.3.2](https://github.com/inference-gateway/.github/compare/v0.3.1...v0.3.2) (2026-05-31)

### ♻️ Improvements

* **repos:** collapse per-repo language to one lowercase top-level field ([#32](https://github.com/inference-gateway/.github/issues/32)) ([5a022cb](https://github.com/inference-gateway/.github/commit/5a022cb3fb1a9ef6dafe7840d7468af77c4cac10))

## [0.3.1](https://github.com/inference-gateway/.github/compare/v0.3.0...v0.3.1) (2026-05-31)

### 👷 CI

* make workflow-run cleanup flexible and rename to cleanup-runs ([#31](https://github.com/inference-gateway/.github/issues/31)) ([1b62b46](https://github.com/inference-gateway/.github/commit/1b62b46768df6e9813dcdb8349aee022c63edf10))
* reconcile cross-repo orchestrators and add the [@infer](https://github.com/infer) reusable workflow ([#30](https://github.com/inference-gateway/.github/issues/30)) ([91ff793](https://github.com/inference-gateway/.github/commit/91ff7936414de0978f9db74b895b03a53dcbcd19))

### 🔧 Miscellaneous

* **assets:** Refresh org star history ([#29](https://github.com/inference-gateway/.github/issues/29)) ([f957f3f](https://github.com/inference-gateway/.github/commit/f957f3f57209901dd29c63f91e37a625c839227a))

## [0.3.0](https://github.com/inference-gateway/.github/compare/v0.2.2...v0.3.0) (2026-05-31)

### ✨ Features

* **claude:** widen default safe Bash toolset, drop operator preset ([#28](https://github.com/inference-gateway/.github/issues/28)) ([8d95176](https://github.com/inference-gateway/.github/commit/8d95176506b3561cded97251965c9df674ce9d74))

### 👷 CI

* add claude code migration ([#27](https://github.com/inference-gateway/.github/issues/27)) ([8614a3b](https://github.com/inference-gateway/.github/commit/8614a3b5704486df2db7a9fda6bf49bb9ae14cf4))

### 🔧 Miscellaneous

* delete .github/workflows/audit-docs-coverage.yml ([02ff1e5](https://github.com/inference-gateway/.github/commit/02ff1e56f083787f62322fe3557e398c163789f4))
* remove weird operator language ([e938855](https://github.com/inference-gateway/.github/commit/e9388558f2e6f9706be43dff18b62d3e81c8dbf2))
* update repos.yaml ([b68c577](https://github.com/inference-gateway/.github/commit/b68c57714eb4ac6219b902624c73dc8208f03d57))

## [0.2.2](https://github.com/inference-gateway/.github/compare/v0.2.1...v0.2.2) (2026-05-30)

### 👷 CI

* add claude.yml migration orchestrator + registry ([#26](https://github.com/inference-gateway/.github/issues/26)) ([f968693](https://github.com/inference-gateway/.github/commit/f96869327f60b31521bddf897c056a35f8756308))

## [0.2.1](https://github.com/inference-gateway/.github/compare/v0.2.0...v0.2.1) (2026-05-30)

### 🐛 Bug Fixes

* **claude:** grant edit/npm tools on workflow_dispatch runs ([#25](https://github.com/inference-gateway/.github/issues/25)) ([afd3d34](https://github.com/inference-gateway/.github/commit/afd3d34612525f57df1d8fa7bd793697095ac108))

## [0.2.0](https://github.com/inference-gateway/.github/compare/v0.1.0...v0.2.0) (2026-05-30)

### ✨ Features

* **claude:** add manual prompt (workflow_dispatch) mode ([#22](https://github.com/inference-gateway/.github/issues/22)) ([e1f6b34](https://github.com/inference-gateway/.github/commit/e1f6b341f946f70235b5f9394b754d76f80749da))
* **claude:** context-aware prompt (issue / PR / manual task) ([#24](https://github.com/inference-gateway/.github/issues/24)) ([c2aeb14](https://github.com/inference-gateway/.github/commit/c2aeb144f6178b055dfc4fa8772bc405c8667eef))

### 🐛 Bug Fixes

* **claude:** disable track_progress on workflow_dispatch ([#23](https://github.com/inference-gateway/.github/issues/23)) ([72b0d7e](https://github.com/inference-gateway/.github/commit/72b0d7e4a082ad71167608de0f196d4037942282))

# Entropy audit — marcelocantos.com

Date: 2026-08-22  
Mode: full (entropy + explicit hygiene validation)  
Auditor: Grok Build subagent (entropy-audit owner)

## Executive summary

- **Snapshot:** `/Users/marcelo/work/github.com/marcelocantos/marcelocantos.com`
  - Branch: `master` (tracking `origin/master`, **ahead 2**)
  - HEAD: `72c96122d4f3563008c89e662e655059a8339e86` (2026-08-03, “Add Open Source products page at /open-source/”)
  - `origin/master`: `ccd7cc32b61d695ba44854f8326311144643786d`
  - Initial dirty state (`git status --porcelain=v1 -b` before any write): clean working tree, `## master...origin/master [ahead 2]`
- **Scope:** site-owned source (15 git paths). Theme submodule inspected as a dependency, not as a product to rewrite.
- **Exclusions:** `public/` and `resources/_gen/` (generated, gitignored); `themes/vanilla-bootstrap-hugo-theme/exampleSite/` and theme `static/` vendored Bootstrap/jQuery/Feather binaries except where they affect the shipped artifact; no Python/Go/C++/Rust/SQL/Bash product code exists to judge.
- **Headline mechanism:** a 2019 Hugo theme is kept alive by **whole-file project partial overrides**, and the **only standing oracle is Netlify after push**. That combination already produced two production outages in 2026-07 (uninstallable Hugo 0.53 pin; literal `$DEPLOY_PRIME_URL` as `baseURL`). Local `hugo --gc --minify` is green on HEAD; GitHub Actions, tests, and `hygiene.yaml` do not exist.
- **Highest-consequence findings:** ENT-001 (deploy-only verification), ENT-002 (frozen theme + override coupling), ENT-003 (`posts` vs theme `post` layout — article dates dropped), ENT-004 (active-nav heuristic fails on Home and post singles).
- **Unverified residue:** live Netlify deploy of unpushed HEAD (owner Ship-plane lag, not a code defect); whether Netlify fetches the git submodule on build; no visual/layout oracle was run (audit did not change CSS).

## Scope and exclusions

**In scope**

- Site root: `config.yaml`, `netlify.toml`, `layouts/partials/*`, `content/**`, `archetypes/default.md`, `.gitignore`, `.gitmodules`, `README.md`, `bullseye.yaml`
- Theme contract used at runtime: `themes/vanilla-bootstrap-hugo-theme/layouts/**` (submodule `dc8c6f2893dede98c600bdf930d2078204dba217`, 2019-01-12)

**Named exclusions (not silent omissions)**

| Tree | Role |
|---|---|
| `public/` | Hugo output; gitignored; regenerated during this audit |
| `resources/_gen/` | Hugo resource cache; gitignored |
| `themes/.../exampleSite/` | Theme demo, including **both** `config.toml` and `config.yaml` |
| `themes/.../static/js/*.min.js`, `static/css/bootstrap.min.css` | Vendored third-party; inspected only for versions and whether HTML references them |
| `themes/.../images/`, `gh-md-toc` | Theme marketing / helper, not on the shipped path |

**Languages actually present:** Hugo templates, YAML, TOML (`netlify.toml`), Markdown, vendored minified JS/CSS. Companions read: `web-development.md`, `journeys.md`. `python.md` / `go.md` / `cpp.md` / `rust.md` / `sql.md` / `bash.md` were not applicable (no product source in those languages). No `AGENTS.md` / `CLAUDE.md` in this repo.

## Commands run

| Command | Version | Exit | Shipped vs auxiliary | Notes / limitations |
|---|---|---|---|---|
| `git rev-parse --abbrev-ref HEAD`; `git rev-parse HEAD`; `git status --porcelain=v1 -b` | git 2.55.0 | 0 | provenance | Initial: `master`, `72c9612…`, clean, ahead 2 |
| `git log --oneline`; `git show c80cc83 18e9a61 ccd7cc3 72c9612` | git 2.55.0 | 0 | history | Two production-fix commits in 2026-07-06 |
| `git ls-files`; `git submodule status` | git 2.55.0 | 0 | inventory | 15 paths; theme gitlink `dc8c6f2` |
| `hugo version`; `hugo env` | hugo v0.163.3+extended+withdeploy darwin/arm64 (Homebrew, 2026-06-18) | 0 | local toolchain | Matches `netlify.toml` pin |
| `hugo --gc --minify` | same | 0 | **shipped path** (same argv as Netlify `command`) | 12 pages, 4 static files, 21 ms. WARN: `languageCode` deprecated since v0.158.0 |
| `gh repo view`; `gh api …/actions/workflows`; `gh api …/branches/master/protection` | gh 2.97.0 | 0 / 0 / 404 | GitHub metadata | Public repo; `workflows.total_count=0`; branch not protected; `licenseInfo=null` |
| HTTP GET `https://marcelocantos.com/`, `/posts/`, `/css/bootstrap.min.css`, `/open-source/` | live Netlify | 200 / 200 / 200 / **404** | production | Production tracks `origin/master`, not local HEAD. `/open-source/` is Netlify’s generic 404 |
| `/Users/marcelo/.claude/skills/hygiene/hygiene_check.py` | skill script (uv-run) | 1 | hygiene validator | `FileNotFoundError: …/hygiene.yaml` |
| `jscpd` / ArchUnit / coverage / secret-scan | n/a | not run | — | Not declared; not installed (audit must not add analyzers) |

Generated `public/` after the shipped build (gitignored): `index.html`, `open-source/index.html`, `posts/{index,goos-goarch-survey,lossy-minds-and-oracles}/index.html`, RSS/sitemaps, `css/bootstrap.min.css`, `js/{feather.min,jquery-3.3.1.slim.min,bootstrap.bundle.min}.js`. **No `public/404.html`.**

## Observed architecture

```
content/*.md  ──►  Hugo 0.163.3  ──►  public/  ──►  Netlify (publish=public)
config.yaml  ─┘         ▲
netlify.toml (HUGO_VERSION, build cmd)
layouts/partials/{head,google-analytics-async}.html  ── overlays ──►
themes/vanilla-bootstrap-hugo-theme @ dc8c6f2 (2019)
  layouts/_default/{baseof,list,single,terms}.html
  layouts/{index,post/single,404}.html
  layouts/partials/{nav,footer,script,style,date-and-tags,…}
  static/{css,js}
```

**Entry points**

- Runtime/build: `hugo --gc --minify` (`netlify.toml:3`), theme selected by `config.yaml:4`.
- Public pages: `/` (theme `layouts/index.html` jumbotron from `params.homeText`), `/open-source/` (`content/open-source.md` → `_default/single.html`), `/posts/` (`content/posts/_index.md` → `_default/list.html`), two posts, RSS `index.xml`, tags/categories (empty taxonomies).
- Deployable unit: one static site. No app server, no DB, no auth.

**Declared vs observed**

| Rule | Status |
|---|---|
| Theme is Vanilla Bootstrap via submodule | Declared (`config.yaml:4`, `.gitmodules`) and observed |
| Project layouts override theme partials | Declared in comments (`layouts/partials/google-analytics-async.html:1-4`, `netlify.toml:8`) and observed (Hugo lookup order) |
| Google Analytics unused | Declared (no-op override) and observed (no GA script in `public/*.html`) |
| Bootstrap JS off unless `includeBootstrapJs` | Observed in theme `script.html:6-8`; site does not set the param → HTML does not reference jQuery/bootstrap.bundle. **Static files are still copied to `public/js/`** |
| `showActiveNav: true` | Declared (`config.yaml:36`) but only works when page `Title` equals menu `Name` (ENT-004) |
| `baseURL` is `https://marcelocantos.com/` | Declared (`config.yaml:1`); observed in generated `absURL` stylesheet/script hrefs |
| Goldmark `unsafe: true` for one table | Declared (`config.yaml:9-14`); observed: GOOS table `<span style=font-size:smaller>` survived minify |
| Hygiene / CI / tests | **Undeclared and absent** |
| Theme `layouts/post/` applies to blog posts | **Contradicted:** content lives in `content/posts/` (Hugo section `posts`), so `layouts/post/single.html` never runs |

**Dependency direction:** site → theme (one way). No cycles. High fan-in hub is theme `baseof.html` (every page). High fan-out is `config.yaml` (menu, params, markup, theme name).

**Cross-cutting:** URLs via `absURL` (CSS/JS) vs root-relative menu hrefs; Feather icons injected in `script.html`; dates via `date-and-tags.html` (list only, see ENT-003).

## Dimension vector

| Dimension | State | Evidence summary | Change from baseline |
|---|---|---|---|
| Architecture topology | concern | Tiny, directional Hugo overlay on a frozen 2019 theme. Overlay is whole-file, so the theme’s `post/` layout and nav heuristic silently miss this site’s `posts/` section and Home title. | n/a (first audit) |
| Redundancy / sources of truth | healthy | Single site `config.yaml`. Theme exampleSite dual TOML/YAML is excluded. Open-source catalog is explicitly curated, not a repo dump. Partial overrides are the SoT for `head` / GA. | n/a |
| Change amplification | concern | Next Hugo API removal requires another full-file override (already done twice). Menu+title+section-name must stay aligned by convention. | n/a |
| Local code quality | concern | Site-owned templates are small and commented. Theme contracts the site does not meet drop dates and active-nav on real pages. | n/a |
| Correctness / verification | critical | `hugo --gc --minify` is green locally and is the Netlify command, but nothing runs it before push. Two 2026-07 production outages. No tests, no HTML proof, no journeys. | n/a |
| Security / dependencies | concern | No runtime backend. Goldmark unsafe HTML (single author). 2018 Bootstrap CSS is served; 2018 jQuery is published but not executed. No secret scan, no Dependabot, submodule unmaintained. | n/a |
| Build / release / operations | concern | Hugo version now pinned and matches local. `HUGO_ENABLEGITINFO` set but unused. No GitHub Actions; Netlify is CD. Local master not on origin (by Ship-plane policy). Empty theme 404 → no `public/404.html`. | n/a |
| Documentation / governance | concern | README is a heading plus “Hugo source repo”. No LICENSE (`gh` `licenseInfo=null`). No `AGENTS.md`. No branch protection. `bullseye.yaml` has T2 achieved locally. | n/a |

Do not aggregate these into a scalar.

## Findings

### ENT-001: The only standing oracle is Netlify after push

- **Priority:** P1
- **Dimensions:** Correctness / verification; Build / release / operations
- **Status:** observed fact
- **Evidence:**
  - `gh api repos/marcelocantos/marcelocantos.com/actions/workflows` → `{"total_count":0,"workflows":[]}`
  - No `Makefile`, no test tree, no `package.json`
  - Shipped command is only `netlify.toml:3` `hugo --gc --minify`
  - `c80cc83` (2026-07-06): Netlify could not install pinned Hugo **0.53** (checksum mismatch); site failed to deploy
  - `18e9a61` (2026-07-06): `HUGO_BASEURL = "$DEPLOY_PRIME_URL"` in `netlify.toml` was **not interpolated**; every absolute URL became the literal `$DEPLOY_PRIME_URL/…` (CSS/JS/post links 404). Comments remain at `netlify.toml:13-16`
  - This audit’s local `hugo --gc --minify` exit 0 is an auxiliary/local run of the shipped argv, **not** a wired gate
- **Mechanism:** Hugo version, `baseURL`, theme APIs, and generated URLs are only validated when Netlify builds a pushed commit. A config typo or deprecation ships as a blank/broken public site.
- **Blast radius:** the whole public site (stylesheet, scripts, every permalink). Already happened twice.
- **Counterevidence checked:** `HUGO_VERSION = "0.163.3"` now matches local Hugo; `baseURL` is a real URL in `config.yaml:1`. Those fix the last incidents; they do not install a pre-push oracle.
- **Smallest coherent remediation:** a GitHub Actions (or `make check`) job that runs `hugo --gc --minify` and fails on WARN/ERROR, plus a link checker on `public/`.
- **Verification:** push a branch that sets `baseURL: "https://broken.example"` or `languageCode` still present after Hugo removes it — CI must go red before Netlify.
- **Ratchet candidate:** CI job `hugo.yml#build` and, once hygiene is declared, `evidence: {ci_job: hugo.yml#build}` / `command: hugo --gc --minify`.

### ENT-002: Frozen 2019 theme kept alive by whole-file partial overrides

- **Priority:** P2
- **Dimensions:** Architecture topology; Change amplification
- **Status:** observed fact
- **Evidence:**
  - Submodule last commit `dc8c6f2` **2019-01-12** (“Add HUGO_BASEURL env var”); theme `netlify.toml` still pins `HUGO_VERSION = "0.53"`
  - Theme `layouts/partials/head.html:4` still uses removed `.Hugo.Generator`
  - Theme `layouts/partials/google-analytics-async.html:2` still uses removed `.Site.GoogleAnalytics`
  - Site copies the files in full: `layouts/partials/head.html` (`hugo.Generator` at line 4 + RSS loop at 7–8) and the GA no-op (`layouts/partials/google-analytics-async.html:1-5`)
  - `c80cc83` message: overrides exist specifically because those APIs were removed since 0.53
  - Current shipped build still WARN-deprecates `config.yaml:2` `languageCode` (removed in a future Hugo; replacement is `locale`)
- **Mechanism:** Hugo partial lookup is all-or-nothing per filename. The site now owns `head.html` forever (RSS was added on the copy in `ccd7cc3`). The next theme- or Hugo-level change does not merge; it either misses the override or requires another hand-port. The live `languageCode` warning is the same class of landmine as 2026-07-06.
- **Blast radius:** any Hugo minor bump on Netlify (`HUGO_VERSION`) or any `git submodule update --remote`.
- **Counterevidence checked:** overrides are commented and small; build is green on 0.163.3; vendoring the theme into the repo would not remove the API problem.
- **Smallest coherent remediation:** (1) replace `languageCode` with `locale` now; (2) keep overrides, but add a CI job that treats Hugo WARN as failure; (3) longer term, drop the submodule for the two templates actually needed, or replace the theme.
- **Verification:** `hugo --gc --minify` with `HUGO_ENABLEGITINFO` on a version that rejects `languageCode` must fail in CI, not only on Netlify.
- **Ratchet candidate:** `hugo --gc --minify -logLevel warning` (or grep the deprecation string) as a blocking command.

### ENT-003: Theme `layouts/post/` never applies to `content/posts/`

- **Priority:** P2
- **Dimensions:** Local code quality; Architecture topology
- **Status:** observed fact
- **Evidence:**
  - Theme `layouts/post/single.html:1-8` renders `date-and-tags.html` under the title
  - Theme `layouts/_default/single.html:1-6` is title + content only
  - Site content is `content/posts/*.md` (section `posts`), with `_index.md` title `Blog`
  - Theme exampleSite uses `content/post/` (section `post`) matching `layouts/post/`
  - Generated `public/posts/lossy-minds-and-oracles/index.html`: `<h1>The gut you can't code</h1>` then body; **no** `<time`, `datetime`, or `data-feather=calendar`
  - Generated `public/posts/index.html` **does** include dates, because `_default/list.html` calls `date-and-tags.html` for every page in the section
- **Mechanism:** Hugo binds `layouts/<section>/` to the content section name. `post` ≠ `posts`, so the dated single template is dead code on this site. List vs single now disagree about whether a post has a visible date.
- **Blast radius:** both current posts and every future `content/posts/*.md` page. Not a correctness crash; owner-visible metadata is missing on the article URL.
- **Counterevidence checked:** `layouts/post/single.html` is not overridden by the site; no `type: post` in front matter; list page dates prove `date-and-tags.html` itself works.
- **Smallest coherent remediation:** add `layouts/posts/single.html` (or `_default/single.html`) that includes `date-and-tags`, **or** move content to `content/post/`. Do not rename the public `/posts/` URL without redirects.
- **Verification:** `public/posts/*/index.html` must contain `<time datetime="…">` after `hugo --gc --minify`.
- **Ratchet candidate:** test or `rg` over generated HTML for `<time datetime=` on each post permalink.

### ENT-004: Active-nav heuristic fails on Home and post singles

- **Priority:** P2
- **Dimensions:** Local code quality
- **Status:** observed fact
- **Evidence:**
  - `config.yaml:36` `showActiveNav: true`
  - Theme `layouts/partials/nav.html:7-10`: active iff `IsMenuCurrent`/`HasMenuCurrent` **or** `eq $currentPage.Title .Name`
  - Home page title is site title `Marcelo Cantos` (`config.yaml:3` + `head.html:11-14`), menu name is `Home` (`config.yaml:18-20`)
  - Generated `public/index.html` nav: four `class=nav-link` links, **none** `nav-link-active`
  - `public/open-source/index.html`: Open Source is active (title matches menu name)
  - `public/posts/index.html`: Blog is active (title `Blog` from `content/posts/_index.md`)
  - `public/posts/lossy-minds-and-oracles/index.html`: Blog is **not** active (`HasMenuCurrent` did not fire for the child URL)
- **Mechanism:** the theme’s fallback compares titles, not URLs. Home can never match. Section children do not inherit the section menu item.
- **Blast radius:** Home (every visit) and every post permalink. Open Source and Blog index happen to work.
- **Counterevidence checked:** `showActiveNav` is not false; CSS for `#nav a.nav-link-active` is present in inlined `style.html`. This is not a CSS-geometry claim (class is absent in HTML).
- **Smallest coherent remediation:** site overlay of `nav.html` that compares `.URL` to `relURL` / `IsHome`, and treats `/posts/` as active for section `posts`.
- **Verification:** after build, Home HTML contains `nav-link-active` on `href=/`; post singles contain it on `href=/posts/`.
- **Ratchet candidate:** same generated-HTML assertions as ENT-003.

### ENT-005: Unused 2018 JS is still published; 404 template is empty

- **Priority:** P3
- **Dimensions:** Security / dependencies; Build / release / operations
- **Status:** observed fact
- **Evidence:**
  - Theme `script.html:6-8` loads jQuery/Bootstrap JS only if `includeBootstrapJs == true` (unset on this site)
  - `rg` over `public/*.html`: no `jquery` or `bootstrap.bundle` references
  - `public/js/jquery-3.3.1.slim.min.js` and `public/js/bootstrap.bundle.min.js` **are** emitted (Hugo copies `static/`)
  - jQuery header: `jQuery v3.3.1`; CSS header: `Bootstrap v4.2.1` (2018)
  - Theme `layouts/404.html` is **0 bytes**; `public/404.html` is absent after `hugo --gc --minify`
  - Live `https://marcelocantos.com/open-source/` (path missing on origin) returns Netlify’s generic “Page not found” page, not a site-branded 404
  - `netlify.toml:11` `HUGO_ENABLEGITINFO = "true"` but no template references `.GitInfo`
- **Mechanism:** static mounts are unconditional; an empty 404 layout produces no output page. Published JS is an extra attack surface even when unreferenced (direct URL). Empty 404 is a missing owner-visible path, not a crash.
- **Blast radius:** `/js/jquery-3.3.1.slim.min.js` on the production origin; unknown URLs get Netlify chrome instead of the site.
- **Counterevidence checked:** JS is not executed on normal pages; Bootstrap **CSS** is required and live (`https://marcelocantos.com/css/bootstrap.min.css` 200). Do not treat unused-JS CVEs as exploited.
- **Smallest coherent remediation:** a real `layouts/404.html`; optionally `module.mounts` (or deleting the unused JS from a forked static dir) so jQuery/bootstrap.bundle are not published.
- **Verification:** `public/404.html` exists and contains site nav; `public/js/jquery-3.3.1.slim.min.js` absent if mounts are tightened.
- **Ratchet candidate:** file existence checks on the generated tree.

### ENT-006: Goldmark `unsafe: true` is a site-wide HTML escape hatch for one table

- **Priority:** P3
- **Dimensions:** Security / dependencies
- **Status:** observed fact (risk); inference (exploitability)
- **Evidence:** `config.yaml:9-14` documents the GOOS/GOARCH `<span>` markup; generated table still contains `<span style=font-size:smaller>`. Single author, no CMS, no untrusted PRs required to build.
- **Mechanism:** every Markdown page can emit raw HTML. A malicious PR would ship script tags without a review bot.
- **Blast radius:** any future Markdown file, not only `goos-goarch-survey.md`.
- **Counterevidence checked:** no comment/user-generated path; Goldmark default is safe; the comment names the exact consumer.
- **Smallest coherent remediation:** keep `unsafe` only if still required; otherwise rewrite the table without raw HTML and set `unsafe: false`. Branch protection + required review if untrusted contributors appear.
- **Verification:** with `unsafe: false`, the GOOS table headers still render (or the build/test asserts they do not regress).
- **Ratchet candidate:** once rewritten, `file` evidence that `unsafe: true` is **absent**.

### ENT-007: Public repo has no LICENSE, no hygiene declaration, no contributor docs

- **Priority:** P3
- **Dimensions:** Documentation / governance
- **Status:** observed fact
- **Evidence:** `README.md:1-3` is a heading plus “Hugo source repo”; `gh repo view` `licenseInfo: null`; no root `LICENSE`; no `AGENTS.md`/`CLAUDE.md`; no `.github/`; `hygiene.yaml` missing; `master` unprotected.
- **Mechanism:** GitHub visitors get no reuse terms (default all-rights-reserved). Agents and humans get no local delivery/test instructions. Hygiene cannot drift-detect what was never declared.
- **Blast radius:** legal ambiguity for a public site repo; onboarding; fleet hygiene aggregation treats this repo as a hole.
- **Counterevidence checked:** theme submodule carries MIT (`theme.toml:2-3`, `LICENSE`); that does not license **this** repo’s content. Personal homepage may intentionally stay unlicensed — owner call.
- **Smallest coherent remediation:** a LICENSE if reuse is intended; a two-paragraph README (Hugo version, `hugo server`, Netlify); `hygiene.yaml` only when the owner wants a declared floor (do not invent one in this audit).
- **Verification:** `gh api repos/…/license`; `hygiene_check.py` exit 0 after a real declaration.
- **Ratchet candidate:** hygiene items `docs.readme`, `governance.license` with `file:` evidence — **only after owner adoption**.

## Redundancy and competing-source-of-truth inventory

| Concept | Authorities | Drift risk |
|---|---|---|
| Site config | `config.yaml` only | Low (healthy) |
| Hugo version | `netlify.toml:10` `0.163.3` vs local Homebrew `v0.163.3` | Currently matched; no CI to keep them matched |
| `baseURL` | `config.yaml:1` (Netlify env no longer overrides) | Low after `18e9a61` |
| `<head>` | Site `layouts/partials/head.html` shadows theme file | Copy will not pick up upstream `head.html` edits (theme is frozen) |
| GA partial | Site no-op vs theme UA snippet | Intentional; site wins |
| Blog dates on singles | Theme `layouts/post/single.html` vs actual section `posts` | **Already drifted** (ENT-003) |
| Active nav | `params.showActiveNav` vs title-equality heuristic | **Already drifted** on Home / post singles (ENT-004) |
| Open-source catalog | `content/open-source.md` vs GitHub org | Deliberate curation (`content/open-source.md` says “not a complete repository index”). `writ` is absent as attested in `bullseye.yaml` T2 |
| Theme example config | `exampleSite/config.toml` **and** `config.yaml` | Theme-only; excluded |
| `netlify.toml` | Site file vs theme demo file (Hugo 0.53, `HUGO_BASEURL`) | Theme file is not used for this site; it **was** the pattern that caused ENT-001’s second outage |
| Archetype | site `archetypes/default.md` (`draft: true`) vs theme (`tags: []`) | Harmless; `hugo new` uses the site file |
| T2 vs production | `bullseye.yaml` T2 achieved + local `/open-source/`; live URL 404 | Ship-plane lag (HEAD not on origin), not two code authorities |

Syntactic clones: site `head.html` is a modified copy of the theme file (intentional overlay). No other meaningful clones in site-owned source. `jscpd` not configured.

## Healthy structure worth retaining

- **One deployable, one config file, one theme pointer.** `config.yaml` is the site SoT; there is no parallel `config.toml` at the site root.
- **Generated output is gitignored** (`.gitignore`: `/public`, `/resources/_gen`, `/.hugo_build.lock`). Netlify builds from source.
- **Hugo version pin matches the developer binary** (`netlify.toml:10` and `hugo version` both 0.163.3). The 0.53 pin is gone.
- **`baseURL` is a real absolute URL** (`config.yaml:1`); the `$DEPLOY_PRIME_URL` foot-gun is documented in `netlify.toml:13-16` so it is less likely to be reintroduced.
- **Overrides are tiny and motivated in comments** (GA no-op; `hugo.Generator`; RSS autodiscovery; Goldmark unsafe for a named table).
- **Bootstrap JS is not executed** (`includeBootstrapJs` unset; `script.html:6-8`).
- **Open-source page is an honest curated list**, not a live GitHub scrape — no second runtime SoT.
- **Local shipped-path build is fast and currently green** (12 pages, 21 ms, exit 0 aside from the `languageCode` WARN).
- **bullseye T2** records the open-source page acceptance criteria and that `writ` was excluded.

## Hygiene posture

**`hygiene.yaml` is absent. Hygiene posture not declared.**

Validator run from repo root:

```
/Users/marcelo/.claude/skills/hygiene/hygiene_check.py
```

Exit 1. Output: `FileNotFoundError: [Errno 2] No such file or directory: '/Users/marcelo/work/github.com/marcelocantos/marcelocantos.com/hygiene.yaml'`.

This audit did **not** initialize `hygiene.yaml`.

Implied reality (not a declared vector — do not treat as floors):

| Dimension | Observed |
|---|---|
| correctness | local `hugo` build only; no tests; no CI |
| security | no scanners |
| quality | no lint/format |
| docs | README exists (heading + one sentence) |
| release | Netlify CD, no GitHub Release |
| governance | public GitHub, no LICENSE, no branch protection |
| build | Netlify `hugo --gc --minify` with pinned Hugo |
| vcs | git + submodule |
| agent | `bullseye.yaml` present |

Overlap with entropy: ENT-001/ENT-007 are the hygiene-shaped gaps. Entropy explains the mechanism (deploy-only oracle, undeclared posture); hygiene would only later decide whether a declared floor still holds.

Entropy findings suitable for future hygiene (after owner adoption, not now): ENT-001 CI hugo job; ENT-002 fail-on-WARN; ENT-003/004 generated-HTML assertions; ENT-005 `public/404.html` exists; ENT-007 LICENSE/README if desired.

## Oracle coverage and residue

| Load-bearing property | How it is decided today |
|---|---|
| Site builds on Hugo 0.163.3 | Local + Netlify shipped path (`hugo --gc --minify`). **Not** gated on GitHub. This audit ran it: exit 0 + `languageCode` WARN |
| CSS/JS URLs are the production host, not a literal env string | Source inspection + generated `href=https://marcelocantos.com/css/bootstrap.min.css`; live CSS 200 on origin |
| Goldmark still emits the GOOS table HTML | Generated HTML after shipped build (auxiliary inspection of shipped argv) |
| Open-source page lists only public products | Manual curation; T2 attestation. No machine check vs GitHub visibility |
| Nav “active” state | **Nothing.** HTML inspection in this audit (ENT-004) |
| Post permalinks show dates | **Nothing.** (ENT-003) |
| 404 is on-brand | **Nothing.** Empty theme template; Netlify default in production |
| `languageCode` still accepted | Hugo WARN only; will become a hard failure later (ENT-002) |
| Layout geometry / visual | **Nothing.** No Playwright/journey. Not claimed. |
| Submodule present on Netlify | **Unknown.** `.gitmodules` exists; Netlify default is usually to clone submodules — not verified against the Netlify project |
| Live `/open-source/` | 404 on production because HEAD is 2 commits ahead of origin. Owner Ship-plane; T2 already says “not pushed to Netlify yet” |

**Failed / skipped checks:** no CI; no tests; no clone detector; no vulnerability scan; `hygiene_check.py` failed on missing yaml (expected); no visual render oracle.

**Owner residue (intent, not mechanical leftover):**

1. Keep the 2019 theme + overlays, or replace the theme?
2. License the public repo, or leave all-rights-reserved?
3. Declare `hygiene.yaml` with a hugo-build floor, or stay undeclared?
4. When to push the 2 unpushed commits (open-source page is live in HEAD, 404 in production)?

## Remediation sequence

1. **Oracle seam (ENT-001, ENT-002):** add a cheap `hugo --gc --minify` check that fails on WARN. Replace `languageCode` with `locale` so the current deprecation cannot become the next Netlify outage. Do this before any Hugo bump.
2. **Boundary ownership (ENT-003, ENT-004):** site-level `layouts/posts/single.html` (dates) and `layouts/partials/nav.html` (URL-based active state). These are local overlays, not a theme rewrite.
3. **Residue after consumers exist (ENT-005):** real `404.html`; stop publishing unused jQuery/bootstrap.bundle if easy.
4. **Ratchet:** CI job + generated-HTML greps for `<time datetime=` on posts and `nav-link-active` on Home. Adopt into `hygiene.yaml` only if the owner asks.
5. **Governance (ENT-007):** README how-to-build; LICENSE iff reuse is intended.
6. **Re-run this audit** against the same finding IDs and the same `hugo --gc --minify` denominator.

Do not start with a theme replacement. The 12-page site does not need a new architecture; it needs a pre-push build gate and two template contracts aligned with `content/posts/`.

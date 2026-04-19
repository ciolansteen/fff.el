# fff.el — Roadmap

> Fork de lucru: `git.ciolan.net/github-mirrors/fff.el` branch `dev`
> Upstream original: https://github.com/JonasThowsen/fff.el
> Upstream fff.nvim: https://github.com/dmtrKovalenko/fff.nvim

---

## Stare curentă (baseline Jonas)

`emacs/fff.el` funcționează cu:
- `fff-find-file` — file picker via consult async
- `fff-grep` / `fff-grep-fuzzy` — grep plain și fuzzy
- FFI direct la `libfff_c.so` via pachetul `emacs-ffi`
- Distribuit ca Nix flake

**Problemă critică**: struct offsets hardcodate în Elisp:
```
offset 8   → FffResult.error
offset 16  → FffResult.handle / SearchResult.count
offset 32  → GrepMatch.line_content
offset 104 → GrepMatch.line (uint64)
offset 120 → GrepMatch.col  (uint32)
```
Dacă upstream `fff-c` schimbă layout-ul structurilor (câmp nou, reordonare,
aliniere diferită), offseturile devin silențios greșite → crash sau date corupte.

---

## TODO

### P0 — Fix struct offsets (trimitem PR la Jonas)

**Opțiunea A — Accessor functions în `fff-c` (recomandat)**

Adaugă în `crates/fff-c/src/lib.rs` funcții getter pentru fiecare câmp expus:
```c
// generat de cbindgen, apelat din Elisp fără offset arithmetic
const char* fff_file_item_get_path(const FffFileItem* item);
const char* fff_file_item_get_relative_path(const FffFileItem* item);
const char* fff_grep_match_get_path(const FffGrepMatch* m);
const char* fff_grep_match_get_relative_path(const FffGrepMatch* m);
const char* fff_grep_match_get_line_content(const FffGrepMatch* m);
uint64_t    fff_grep_match_get_line(const FffGrepMatch* m);
uint32_t    fff_grep_match_get_col(const FffGrepMatch* m);
uint32_t    fff_search_result_get_count(const FffSearchResult* r);
uint32_t    fff_grep_result_get_count(const FffGrepResult* r);
```
Elisp apelează funcții în loc să facă pointer arithmetic → zero dependență de layout.
Sunt **două PR-uri în ordine**:
- **PR 1 → `dmtrKovalenko/fff.nvim`** — getter functions în `crates/fff-c` (C API upstream)
- **PR 2 → `JonasThowsen/fff.el`** — `emacs/fff.el` folosește getterii (depinde de PR 1 mersat)

**Opțiunea B — Generare automată offseturi via cbindgen**

`cbindgen` generează `fff.h` → script Python parsează header-ul →
generează `fff-offsets.el` cu constante. Build step adăugat în `Makefile`/`flake.nix`.
Mai fragil decât A, dar nu necesită modificări upstream.

**Decizie: Opțiunea A** — mai curată, mai sigură, merit să fie în upstream.

**Workflow convenit:**
1. Implementăm în `dev` pe fork-ul nostru (`git.ciolan.net/github-mirrors/fff.nvim`)
2. Updatăm `fff.el` în `dev` pe `git.ciolan.net/github-mirrors/fff.el`
3. Testăm end-to-end în Emacs (file search + grep — verificăm line_content/line_number/col)
4. Push în `main` pe GitHub (`ciolansteen/fff.nvim` + `ciolansteen/fff.el`)
5. PR 1 → Kovalenko (fff-c getters, framing: "fff beyond Neovim")
6. PR 2 → Jonas cu referință la PR 1 + nota că fork-ul nostru e fallback dacă PR 1 nu e mersat

**PR descriptions draft**: salvate în sesiunea Copilot CLI 083b4aaa

**Offset-uri deja greșite în codul lui Jonas vs struct actual:**

| Ce citește Jonas | Offset hardcodat | Ce e de fapt acolo |
|---|---|---|
| `line_content` | 32 | `FffMatchRange*` (pointer la struct!) |
| `line_number` | 104 | `byte_offset` |
| `col` | 120 | `context_before_count` |

Offseturi corecte acum: `line_content`@24, `line_number`@96, `col`@112 — **deja broken, nu doar risc viitor**.

- [ ] Implementat getter functions în `crates/fff-c/src/lib.rs` (branch `dev`, fork fff.nvim)
- [ ] Updatat `emacs/fff.el` să folosească getteri în loc de offset arithmetic
- [ ] Testat end-to-end în Emacs
- [ ] Push main pe GitHub + **PR 1** → `dmtrKovalenko/fff.nvim`
- [ ] **PR 2** → `JonasThowsen/fff.el` via `ciolansteen/fff.el` (cu fallback note)

---

### P1 — Packaging non-Nix

Jonas distribuie doar ca Nix flake. Adăugăm suport pentru:
- [ ] `straight.el` / `use-package` (`:build` step care compilează `.so`)
- [ ] PKGBUILD în `archRepo` (pentru Arch Linux)
- [ ] Script `build.sh` simplu: `cargo build -p fff-c --release && cp target/release/libfff_c.so emacs/`

---

### P2 — Integrare ecosistem Emacs complet

- [ ] `marginalia` annotations — afișează frecency score + git status în coloana dreaptă
- [ ] `embark` actions — `open`, `copy-path`, `git-diff`, `grep-in-dir`
- [ ] `recentf` integration — `fff--track-selection` să adauge și în `recentf`
- [ ] Fix `recentf` pentru `emacsclient` (`server-visit-hook`)
- [ ] `which-key` descriptions pentru toate comenzile

---

### P3 — Features noi

- [ ] `fff-find-file-in-dir` — picker într-un director specificat
- [ ] `fff-switch-project` — schimbă proiectul activ (integrare cu `project.el`)
- [ ] Suport `git:modified` și alte constraints din query parser
- [ ] Preview live în minibuffer (via `consult--preview`)

---

### Referințe utile

- `fff-c` API header: `crates/fff-c/include/fff.h` din upstream
- PR #333 (anmonteiro) — custom picker API, rejectat fără explicații, cherry-pick candidat
- PR #341 (magnusmalm) — BigramQuery + FileRecord repr(C), open fără răspuns
- Research complet fork-uri: `git.ciolan.net/github-mirrors/fff.nvim` `dev/docs/` (fff.nvim fork)

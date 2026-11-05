# Local APT repo: placement + permissions (PLAN, transient)

Status: DESIGN. Delete this file once the local-apt-repo work lands.

## Problem (one sentence)

Where should the derivative-maker LOCAL `file:` apt repo (reprepro codename
`local`) live and how should it be owned/permissioned, so the unprivileged build
user can produce it, apt's `_apt` sandbox can read it, and `[trusted=yes]` stays
safe -- across the three ways it is consumed.

## dm threat model (decisive -- `agents/security.md`)

Build-time filesystem inputs are TRUSTED: "Files on the filesystem are considered
trusted at the time derivative-maker runs; the user is expected to have verified
the Git repo integrity." So a local-privilege-escalation via "who can write the
repo" (a malicious `debian/rules` staging a root-postinst `.deb` into a
build-user-owned `[trusted=yes]` repo that a later root `apt` consumes) is OUT OF
SCOPE for the build. That removes the entire motivation for a root-owned repo.

## How the local repo is consumed (do not conflate)

- BUILD chroot: bind-mounted to `${CHROOT_FOLDER}/mnt/local-apt-repo/<derivative>`;
  apt runs as root inside the chroot. (`create-local-temp-apt-repo`)
- TARGET ROOT (live/dev system): bind-mounted to `/mnt/local-apt-repo/<derivative>`;
  apt runs as root on the host.
- VM (image test): delivered read-only over a VBox shared folder to `/mnt/shared`;
  the guest enabler writes a `.sources` stanza. (`dm-shared-apt-repo` + `repo-add-local`)

Writer: the unprivileged build user, via reprepro (`dm-reprepro-wrapper`).
Store: `${binary_build_folder_dist}/aptrepo_local/<derivative>`, where
`binary_build_folder_dist` is `~/derivative-binary[/<dist_build_slot>]` -- so the
repo is already PER-SLOT. Made `_apt`-readable by a recursive `chmod o+r` run as
the build user.

## Constraints (verified against the code + cited research)

1. `_apt` sandbox note is BENIGN (`N:`, apt proceeds); fires when `_apt` cannot
   traverse/read the path. In all three consumption paths apt reads the repo
   through a ROOT-OWNED mountpoint (`/mnt/local-apt-repo` bind, or the `/mnt/shared`
   vboxsf), so the backing store's own location/ownership is not what apt walks.
   The existing build-user `chmod o+r` already clears the note.
2. `[trusted=yes]` reduces trust to who-can-write -- but per the dm threat model
   above, build inputs are trusted, so this is not an in-scope LPE for the build.
3. The build repo MUST remain writable by the unprivileged build user: reprepro
   (incl. `list` for the `--skip-published-packages` freshness check) needs write
   to `db/` (`db/lockfile`), and `create-local-temp-apt-repo:73-74` chmods the
   tree as the build user. A root-owned tree EPERM-aborts both.
4. The build repo MUST stay under `binary_build_folder_dist` (per-slot,
   concurrency-safe) and be visible to the docker build (`docker-run` mounts
   `derivative-binary`, not `/srv`, and uses `--rm`).

## Idiom research (for reference)

- Throwaway local repo = `[trusted=yes]`, NOT signing: live-build, pbuilder, sbuild
  (migrated to `[trusted=yes]` 2019), mmdebstrap. Written as root into a root-owned
  dir INSIDE the chroot or bind-mounted; `_apt` note left benign or defused only
  when genuinely unreadable. FHS 3.0 3.17: `/srv` is "data served by this system";
  Debian's `local-apt-repository` uses `/srv/local-apt-repository`.
- These tools write as root because they are not built under dm's unprivileged +
  per-slot + reused-across-builds model; the idiom's "root-owned" detail does not
  transfer once the dm constraints (3)+(4) are applied.

## Strategies

| # | Strategy | Verdict |
|---|---|---|
| 1 | Status quo: `${binary_build_folder_dist}/aptrepo_local`, build-user `chmod o+r`, bind-mount to `/mnt` | KEEP as the build store. Per-slot, docker-mounted, freshness + chmod work. Consistent with the trusted-build-inputs model. |
| 2 | `/srv/<base>`, build-user-OWNED, build user writes directly | REJECT as the build store: not per-slot, not docker-mounted (constraint 4), and `/srv` is root-owned so the build user cannot `mkdir` there without extra setup. Only tenable as a separate on-target copy (Strategy 4). |
| 3 | `/srv/<base>`, root-owned, reprepro writes AS ROOT, `[trusted=yes]` | REJECT. ai-review (codex+grok+coderabbit, verified): breaks the build-user chmod (EPERM), the reprepro `db/` lock (freshness ref-rebuilds everything), slot isolation + concurrency, and docker persistence -- and is NOT even LPE-safe (reprepro-as-root reads a build-user-writable `conf/distributions`). The LPE it chased is out of scope anyway. |
| 4 | Build under `binary_build_folder_dist`; COPY/sync to a root-owned `/srv` only for the TARGET-ROOT/VM enable path | POSSIBLE if an FHS on-target location is genuinely wanted, but adds a copy; and (1)+ the root-owned `/mnt` bind already gives apt a clean read path, so the `_apt` benefit is marginal. Defer unless a concrete need appears. |
| 5 | root-owned `/srv`, GROUP-writable, build user in a group | REJECT. For an unsigned `[trusted=yes]` repo group-write != root-exclusive-write (any member -> root); no conventional Debian publish group (`_apt` is the sandbox READER). Answers "add build user to group apt?" = no. |
| 6 | SIGN the local repo, drop `trusted=yes` | Does NOT make a build-user-owned repo "safe" when the build user holds the signing key (ai-review, verified) -- so it buys nothing here over the trusted-build-inputs model. Defer. |
| 7 | localhost HTTP | REJECT. Server process; defeats the network-free VBox-shared-folder design. |

## ai-review reconciliation (2026-11-05, 5 reviewers; agy skipped on an auth window)

All findings verified against the code and ACCEPTED; they invalidated the original
Strategy 3 recommendation:
- Root-owned tree EPERM-aborts the build-user `chmod` (`create-local-temp-apt-repo:73-74`)
  and the reprepro `db/` lock (freshness), rebuilds everything.
- A fixed `/srv` path omits `dist_build_slot` -> concurrent slotted builds collide;
  unqualified `reprepro remove local <pkg>` removes all arches.
- `docker-run` (`--rm`, mounts `derivative-binary` not `/srv`) loses a `/srv` repo.
- Strategy 3 is not LPE-safe (build-user-writable `conf/distributions` + reprepro-as-root);
  signing (Strategy 6) does not help when the writer holds the key.
- `docs/` is prohibited (`AGENTS.md:10`); this file moved to `agents/`.

## Revised recommendation

DROP the `/srv` storage relocation. Keep the LOCAL repo build-user-owned under
`${binary_build_folder_dist}/aptrepo_local/<derivative>` (Strategy 1) -- per-slot,
docker-visible, freshness + chmod intact, and consistent with the trusted-build-inputs
threat model. apt already reads it through a root-owned `/mnt` bind / `/mnt/shared`
share, so the `_apt` note and FHS concerns are addressed at the CONSUMPTION layer,
not by moving the store.

Therefore the remaining local-apt-repo work is NAMING + PLUMBING only, NOT a
permissions redesign:
- rename the mount `/mnt/initialdeb` -> `/mnt/local-apt-repo` (+ drop `initialdeb`/`temp`
  from the sources filename and the `dist_sources_list_*` vars); one clean value+var
  rename, no `/srv`, no CI override, no reprepro-as-root.
- the sourceable deb822 stanza generator (landed) + `repo-add-local` (rename of the
  dist-ai guest helper, home in helper-scripts).
- dead-code + hardcoded-path cleanups in the developer-meta-files tools.
- dist-ai orchestration wiring.

Open for the human: confirm dropping `/srv` (Strategy 1 as the store). If an FHS
on-target location is still wanted for the dev/target-root ENABLE path specifically,
that is Strategy 4 (a copy at enable time), decided separately and later.

## Sources

- dm threat model: `agents/security.md`.
- `_apt` note / `[trusted=yes]` / FHS: sources.list(5); apt-pkg/acquire.cc
  (`IsAccessibleBySandboxUser`); Debian bug #910079; Debian wiki
  DebianRepository/UseThirdParty; FHS 3.0 3.17.
- Idiom: live-build `chroot_archives`; pbuilderrc(5); sbuild `[trusted=yes]` move
  (debian-devel 2019/07); mmdebstrap commit 920877f; Debian `local-apt-repository`.

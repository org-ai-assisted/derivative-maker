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
Store TODAY: `${binary_build_folder_dist}/aptrepo_local/<derivative>`, where
`binary_build_folder_dist` is `~/derivative-binary[/<dist_build_slot>]` -- SLOT-AWARE
(the `/<slot>` component appears only when `--build-slot` is set; default and CI are
flat). Made `_apt`-readable by a recursive `chmod o+r` run as the build user.

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
4. The build repo MUST be build-user-OWNED (constraint 3), SLOT-AWARE (so
   concurrent `--build-slot` builds do not collide), and VISIBLE to the docker
   build (which uses `--rm` and mounts only its declared volumes). A new `/srv`
   store satisfies all three by: `chown`-ing the leaf to the build user, carrying
   the `${dist_build_slot:+/<slot>}` component, and being added to the `docker-run`
   volume mounts.

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
| 1 | Status quo: `${binary_build_folder_dist}/aptrepo_local`, build-user `chmod o+r`, bind-mount to `/mnt` | WORKS but does NOT fix the `_apt` warning at its root: it relies on world-chmod-ing a subtree under a `0700 $HOME` plus the `/mnt` bind. In `$HOME` (not FHS). Not chosen. |
| 2 | `/srv/<base>`, build-user-OWNED (`chown` the leaf), `[trusted=yes]` | CHOSEN. The reviewers' breakages below were ROOT-owned-specific; a build-user-owned `/srv` leaf keeps the build-user `chmod` and the reprepro `db/` lock working. `0755` world-traversable fixes the `_apt` warning STRUCTURALLY (no `$HOME` chmod, no bind dependency). Same trust as status quo (build-user-owned either way). |
| 3 | `/srv/<base>`, ROOT-owned, reprepro writes AS ROOT, `[trusted=yes]` | REJECT. ai-review (codex+grok+coderabbit, verified): breaks the build-user `chmod` (EPERM), the reprepro `db/` lock (freshness rebuilds everything), and is NOT even LPE-safe (reprepro-as-root reads a build-user-writable `conf/distributions`). The LPE it chased is out of scope anyway. |
| 4 | Build under `binary_build_folder_dist`; COPY to a root-owned `/srv` only for the TARGET enable path | Deferred. Strategy 2 already serves build + consumption from one place; revisit only if a persistent root-owned on-target repo is ever wanted. |
| 5 | GROUP-writable, build user in a group (e.g. `_apt`) | REJECT / non-starter. There is NO `_apt` group (`_apt` is uid 42, primary group `nogroup`), so nothing to join; group-write also != root-exclusive-write. Answers "add build user to group `_apt`?" = no. |
| 6 | SIGN the local repo, drop `trusted=yes` | Does NOT make it "safe" when the build user holds the signing key (ai-review, verified); non-idiomatic for a throwaway repo. Defer. |
| 7 | localhost HTTP | REJECT. Server process; defeats the network-free VBox-shared-folder design. |

## ai-review reconciliation (2026-11-05, 5 reviewers; agy skipped on an auth window)

All findings verified against the code and ACCEPTED. They killed the ROOT-owned
`/srv` variant (Strategy 3), NOT `/srv` itself -- a build-user-owned `/srv`
(Strategy 2) sidesteps every one:
- Root-owned tree EPERM-aborts the build-user `chmod` (`create-local-temp-apt-repo:73-74`)
  and the reprepro `db/` lock -> build-user-OWNED avoids both.
- A FIXED `/srv` path omits `dist_build_slot` -> collides for concurrent slotted
  builds -> fixed by slot-pathing the base (below).
- `docker-run` (`--rm`, mounts `derivative-binary` not `/srv`) loses a `/srv` repo
  -> fixed by adding `/srv/derivative-maker` to the docker volume mounts.
- Strategy 3 not LPE-safe; signing (6) does not help when the writer holds the key.
- `docs/` is prohibited (`AGENTS.md:10`); this file lives in `agents/`.

## Recommendation (locked)

Strategy 2: the LOCAL repo lives at **build-user-owned
`${local_apt_repository_base}/aptrepo_local/<derivative>`**, where
`local_apt_repository_base="/srv/derivative-maker${dist_build_slot:+/${dist_build_slot}}"`
(mirrors `binary_build_folder_dist` -- slot-aware for free; default is a flat
`/srv/derivative-maker`, no slot subdir). This fixes the `_apt` warning structurally
(`0755` world-traversable, no `$HOME` world-chmod), is FHS/idiomatic, `[trusted=yes]`,
and is no less secure than today (build-user-owned either way; root-exclusive-write
is unachievable without the Strategy 3 breakages and is out of scope per the threat
model). The REMOTE release repo stays `~/derivative-binary/aptrepo_remote/<derivative>`.

Plumbing:
- Only the LOCAL (codename `local`) reprepro `outdir`/`dbdir`/`logdir` +
  `dist_apt_repository_folder` move under `local_apt_repository_base`; the reprepro
  `--basedir` (conf/distributions, in the source tree) is unchanged.
- Setup step (`SUDO_TO_ROOT mkdir --parents` + `chown ${user_name}` + `chmod 0755`),
  mirroring the `1200_prepare-build-machine:270-273` approx-cache precedent.
- `docker-run`: add `/srv/derivative-maker` to the volume mounts (host volume,
  survives `--rm`).
- CI: build-user-owned `/srv` on the ephemeral runner, OR override
  `local_apt_repository_base=${binary_build_folder_dist}` to keep the existing
  `~/binary_mnt/aptrepo_local` cache. (Decide at implementation.)
- Consumption stays SLOT-FREE: the slot is resolved HOST-SIDE (the build/bind/share
  picks the slot-resolved dir) and bind-mounted to `/mnt/local-apt-repo` (build/target)
  or shared to `/mnt/shared` (VM). `repo-add-local` enables the ONE delivered repo;
  multiple slots are supported in theory (optional selector) but never the default.

Remaining naming/cleanup (independent of the above): `/mnt/initialdeb` ->
`/mnt/local-apt-repo` (+ `local_apt_repo_mount_folder`), `create/remove-local-temp-apt-repo`
-> `create/remove-local-apt-repo`, the sourceable stanza generator (landed) +
`repo-add-local`, dead-code + hardcoded-path cleanups, dist-ai orchestration.

## Sources

- dm threat model: `agents/security.md`.
- `_apt` note / `[trusted=yes]` / FHS: sources.list(5); apt-pkg/acquire.cc
  (`IsAccessibleBySandboxUser`); Debian bug #910079; Debian wiki
  DebianRepository/UseThirdParty; FHS 3.0 3.17.
- Idiom: live-build `chroot_archives`; pbuilderrc(5); sbuild `[trusted=yes]` move
  (debian-devel 2019/07); mmdebstrap commit 920877f; Debian `local-apt-repository`.

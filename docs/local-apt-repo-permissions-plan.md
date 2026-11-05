# Local APT repo: placement + permissions (PLAN, transient)

Status: DESIGN. Delete this file once the local-apt-repo relocation lands.

## Problem (one sentence)

Where should the derivative-maker LOCAL `file:` apt repo (reprepro codename
`local`) physically live, and how should it be owned/permissioned, so the
unprivileged build user can produce it, apt's `_apt` sandbox can read it, and
`[trusted=yes]` stays safe -- across all three ways it is consumed.

## How the local repo is consumed (do not conflate)

- BUILD chroot: bind-mounted to `${CHROOT_FOLDER}/mnt/local-apt-repo/<derivative>`;
  apt runs as root inside the chroot. (`create-local-apt-repo`)
- TARGET ROOT (live/dev system): bind-mounted to `/mnt/local-apt-repo/<derivative>`
  on the real host; apt runs as root on the host.
- VM (image test): delivered read-only over a VBox shared folder to `/mnt/shared`;
  the guest enabler writes a `.sources` stanza. (`dm-shared-apt-repo` + `repo-add-local`)

Writer today: the unprivileged build user, via reprepro (`dm-reprepro-wrapper`).
Store today: `~/derivative-binary/aptrepo_local/<derivative>` (same tree as the
REMOTE release repo), made `_apt`-readable by a recursive `chmod o+r`.

## Two hard constraints (from research, cited below)

1. apt `_apt` download sandbox. The note
   `N: Download is performed unsandboxed as root ... couldn't be accessed by user '_apt'`
   is BENIGN (`N:`, apt proceeds). It fires when `_apt` cannot TRAVERSE/READ the
   path -- classically a mode-0700 `$HOME` ancestor. The CORRECT fix is a
   world-traversable path (every ancestor `o+x`, files `o+r`) so the sandbox
   stays ON. `APT::Sandbox::User "root"` is a workaround, not a fix (it parses
   untrusted download data as root). A world-readable repo under `$HOME` "works"
   only by chmod-ing a `$HOME` subtree readable to all local users -- leaky.

2. `[trusted=yes]` removes ALL authentication, so trust reduces entirely to who
   can WRITE bytes into the tree. The repo dir AND EVERY ANCESTOR up to `/` must
   be root-exclusive-write; otherwise a lower-privileged account can stage a
   `.deb` whose maintainer script runs as root on the next `apt` run -- a local
   privilege escalation. (dm's existing local repo already uses `Trusted: yes`.)

Corollary: an unsigned `[trusted=yes]` repo is safe ONLY when its whole path
chain is root-exclusive-write, OR the repo is SIGNED (then write-access no longer
equals root).

## Dominant ecosystem idiom (what comparable tools do)

- Throwaway local repo = `[trusted=yes]` on the source line, NOT signing: live-build
  (`chroot_archives`, fixed in-chroot `/root/packages`, `deb [trusted=yes]
  file:/root/packages ./`), pbuilder (`OTHERMIRROR "deb [trusted=yes] file:..."`
  + `BINDMOUNTS`), sbuild (migrated from a signed dummy archive to `[trusted=yes]`,
  2019), mmdebstrap (`[trusted=yes]` hook; `copy://` to sidestep `file:`-in-chroot).
- All write the repo AS ROOT into a root-owned dir (~0755), inside the chroot or
  bind-mounted. The `_apt` note is left benign or defused by mmdebstrap's model
  (auto `APT::Sandbox::User "root"` ONLY when the chroot is genuinely unreadable).
- Served/persistent repo = `/srv`, dedicated system user, SIGNED, read-only to apt:
  Debian's own `local-apt-repository` package defaults to `/srv/local-apt-repository`;
  DebOps reprepro uses `/srv/reprepro` with a dedicated `reprepro` system user+group.
- FHS 3.0 3.17: `/srv` is "site-specific data served by this system" -- the
  correct home. `/var/cache` is wrong (repo is authoritative, not regenerable);
  `/var/lib` is for a program's own private state (reprepro's `db/` is arguably
  `/var/lib`-shaped, the published `dists/`+`pool/` are not).

## Strategies

| # | Strategy | Verdict |
|---|---|---|
| 1 | Status quo: `~/derivative-binary/aptrepo_local`, recursive `chmod o+r`, bind-mount to `/mnt` | REJECT. Repo in `$HOME` (backed up, not FHS, world-readable `$HOME` subtree); build-user-owned so not root-exclusive-write -> `[trusted=yes]` LPE on a shared box. |
| 2 | `/srv/<base>`, build-user-OWNED, build user writes directly, `[trusted=yes]` | OK for a single-operator dev box ONLY (the build user already has sudo, so no ESCALATION). FHS-correct, `_apt`-clean. NOT multi-user safe; not root-exclusive-write. |
| 3 | `/srv/<base>`, root-owned 0755, reprepro writes AS ROOT (`SUDO_TO_ROOT`), `[trusted=yes]` | RECOMMEND. Satisfies the LPE rule, `_apt` note vanishes (0755/0644), FHS + ecosystem idiom (live-build/pbuilder/sbuild all write as root, unsigned). Cost: the local-repo publish step gains `SUDO_TO_ROOT`. |
| 4 | Build in `$HOME` (CI cache intact), SYNC to root-owned `/srv` at enable time | Viable but adds a copy (drift risk, double storage). Keep as fallback if (3)'s reprepro-as-root proves too invasive. |
| 5 | root-owned `/srv`, GROUP-writable (`root:<grp>` 2775), build user in `<grp>` | REJECT for unsigned. Group-write != root-exclusive-write: ANY group member can stage a root-installed package (LPE). No conventional Debian publish group exists (`_apt` is the sandbox READER). Only acceptable if the repo is SIGNED and group members are treated as root-equivalent. (Directly answers "add build user to group apt?" = no.) |
| 6 | SIGN the local repo (reprepro `SignWith`, `Signed-By=` keyring), drop `trusted=yes` | Strongest: build-user-owned becomes safe because apt authenticates. RUNNER-UP. Heavier (throwaway-key management) and against the ecosystem norm for a throwaway dev repo; defer unless a multi-user/service model is wanted. |
| 7 | localhost HTTP instead of `file:` | REJECT. Needs a server process; defeats the network-free VBox-shared-folder design; overkill. |

## Recommendation

Default = Strategy 3: `/srv/derivative-maker/aptrepo_local/<derivative>`,
`root:root`, dirs `0755` + files `0644`, written AS ROOT in the local-repo publish
step, consumed `[trusted=yes]`. This is root-exclusive-write (LPE-safe even
multi-user), makes the `_apt` note disappear with no flag to remember (safe path
is the default), is FHS-correct, and matches the live-build/pbuilder/sbuild idiom.
The REMOTE release repo stays `~/derivative-binary/aptrepo_remote/<derivative>`
(persistent, `$HOME`-backed-up).

- New var `local_apt_repository_base`, default `/srv/derivative-maker`. Only the
  LOCAL (codename `local`) reprepro `outdir`/`dbdir`/`logdir` + `dist_apt_repository_folder`
  move under it; the reprepro `--basedir` (conf/distributions, in the source tree)
  is unchanged.
- A one-time setup step ensures `/srv/derivative-maker` exists `root:root 0755`
  (via `SUDO_TO_ROOT`).
- CI OVERRIDE: `local_apt_repository_base=${binary_build_folder_dist}` -> the
  existing `~/binary_mnt/aptrepo_local` cache + chowns stay byte-identical. CI
  consumes the repo only inside the build chroot (apt-as-root, note benign) on an
  ephemeral single-tenant runner, so no `/srv`, no reprepro-as-root, no cache
  rewiring there. Safe default on a dev/OVH box; CI keeps its fast path.

Runner-up = Strategy 6 (sign) if dm ever wants a non-root publisher or a
multi-user served repo; revisit then, do not build now.

Open sub-decision for the human: the `/srv` leaf name
(`/srv/derivative-maker/aptrepo_local/<derivative>` vs a flat
`/srv/derivative-local-apt-repo/<derivative>`), and whether reprepro-as-root for
the local publish is acceptable (Strategy 3) or we take the sync fallback
(Strategy 4).

## Sources

- apt `_apt` sandbox note (benign `N:`, 0700-$HOME cause): `CheckDropPrivsMustBeDisabled()`
  / `IsAccessibleBySandboxUser()` in apt-pkg/acquire.cc; Debian bug #910079.
- `[trusted=yes]` semantics + LPE: sources.list(5); Debian wiki DebianRepository/UseThirdParty;
  debian-devel 2021/08 (sudo-on-apt -> root shell).
- FHS 3.0 3.17 (/srv), 5.5 (/var/cache), 5.8 (/var/lib).
- Placement precedent: Debian `local-apt-repository` -> /srv/local-apt-repository;
  DebOps reprepro role (`reprepro` system user+group) -> /srv/reprepro.
- Throwaway-repo idiom: live-build `scripts/build/chroot_archives`; pbuilderrc(5)
  OTHERMIRROR/BINDMOUNTS; sbuild move to `[trusted=yes]` (debian-devel 2019/07);
  mmdebstrap `--setup-hook` + sandbox-user auto-disable (commit 920877f).

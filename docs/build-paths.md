# Build paths: who calls what

Text-only flow charts of how a derivative-maker build is actually driven, and
where the dry-run path stops matching the real one.

Every arrow below was read out of the files on THIS branch. `ci/build` no longer
exists here -- `ee8711e7f` retired it so there is one build path, with
`ci/reproducible-build-twice` driving `dm-build-official` directly. It is still
present on master, so a reader coming from master will find one entry point more
than this document describes.

## The single build spine

All four lanes converge here. Nothing bypasses it.

```
help-steps/dm-build-official
  -> ./derivative-update --update-only         # CI=true only
  -> help-steps/dm-build-official-one          # the orchestrator
       Phase 1  build-steps.d/*_sanity-tests           (per flavor)
       Phase 2  *_prepare-build-machine
                *_cowbuilder-setup
                *_local-dependencies
                *_create-debian-packages   --flavor source
                                           --target iso + the union target set
       Phase 3  ./derivative-maker  <build_args> <skip_shared_args> --flavor F
                  -> for step in ./build-steps.d/* ; do "$step" ; done
                     ...
                     1100_sanity-tests
                     1600_export-libvirt-xml           (raw/qcow2 only)
                     3600_convert-raw-to-iso           (iso only)
                     5200_prepare-release -> dm-prepare-release
                          -> dm-reproducible-buildinfo   # per-image .dm-buildinfo
                     ...
       Phase 4  if flavor_built "$dist_build_source_release_flavor":
                   dm-prepare-release --target source
                   dm-upload-images   --target source
                dm-upload-images   <image targets>
```

Two things that read like gaps and are not:

- Phase 4 runs `dm-prepare-release` for `--target source` ONLY. Per-image
  buildinfo comes from `5200_prepare-release` inside each Phase 3
  `./derivative-maker` run. `./derivative-maker` iterates ALL of
  `build-steps.d/`, and Phase 3's `$skip_shared_args` skips only
  prepare-build-machine, cowbuilder-setup, local-dependencies and
  published-packages -- never 5200.
- The source release is gated on `flavor_built "$dist_build_source_release_flavor"`
  (default `kicksecure-lxqt`), so a flavors_list without it publishes no source.

## 1. CI reproducibility lane

```
.github/workflows/local-build.yml
  -> job build (matrix: copy = a, b)          # two SEPARATE runners
       ci/configure-fork-mirror
       git submodule update --init --recursive
       ci/checkout-fork-submodule-branches
       docker/derivative-maker-docker-run --
         help-steps/signing-key-create
         help-steps/sign-and-tag
         help-steps/dm-build-official --freshness frozen
  -> job compare
       dm-reproducible-compare-artifacts   (a vs b, whole-file sha256
                                            + best-effort diffoscope)
```

## 2. CI dry-run lane

```
.github/workflows/local-build-dry-run.yml
  -> docker image: kicksecure/derivative-maker-docker (the REAL build image)
  -> launch-systemd-container
  -> ci/dry-run
       -> run-parts ci/dry-run.d/
            100_install
            150_signing-key-create
            200_sign-and-tag
            300_run-derivative-maker
              -> help-steps/run-as-user -- builder
                   env flavors_list=kicksecure-lxqt
                       dist_build_multi_target_list=qcow2
                   help-steps/dm-build-official --dry-run true --freshness frozen
              -> assert_produced '*.qcow2.libvirt.xz'
              -> assert_produced '*.dm-buildinfo'
              -> assert provenance is not 'unrecorded'
            400_reproducible-buildinfo        (hand-rolled stand-in)
```

## 3. CI boot-test lane

```
.github/workflows/local-boot-test.yml
  -> job build (matrix: image_kind = qcow2, iso)
       docker/derivative-maker-docker-run --
         signing-key-create && sign-and-tag && dm-build-official --freshness frozen
  -> job boot-test (matrix: kind x firmware x session = 12 legs)
       ci/locate-boot-image
       dist-ai/usr/bin/dm-boot-test
         -> dm-image-boot-tests
              -> dm-image-test  -> dm-qemu   (SMBIOS cmdline injection)
```

## 4. Local developer paths

```
ci/reproducible-build-twice --target T --arch A
  -> help-steps/signing-key-create
  -> help-steps/sign-and-tag          # once, before BOTH builds
  -> build a: CI=true ... help-steps/dm-build-official --freshness frozen
  -> build b: CI=true ... help-steps/dm-build-official --freshness frozen
  -> dm-reproducible-compare-artifacts a/ b/

docker/derivative-maker-docker-run -- <any command>    # bare entry point
```

## 5. dry-run YES vs NO

`--dry-run true` sets `build_dry_run=true` in `help-steps/parse-cmd`. What that
changes, by file:

```
                         dry-run NO            dry-run YES
help-steps/variables
  VMSIZE                 100G   (:745)         1M   (:743)
  rsync_cmd              rsync  (:1068)        echo simulate-only rsync  (:1066)

help-steps/dm-build-official-one
  ~/.ssh required        yes    (:80)          skipped  (:78)
```

`rsync_cmd` has THREE mock sites, and the first one to set it wins (each is
`[ -n "${rsync_cmd:-}" ] ||` or an `if [ -z ... ]`):

```
help-steps/pre:134                 CI=true                  -> echo simulate-only rsync
help-steps/dm-build-official-one:164  CI=true, or
                                   remote-derivative-packages true
                                                            -> true simulate-only
help-steps/variables:1066          build_dry_run=true       -> echo simulate-only rsync
```

So an upload is prevented by CI alone, by dry-run alone, or by both. The dry-run
lane sets both. `~/.ssh` is likewise skipped under CI=true independently of
dry-run.

Everything else -- package building in cowbuilder, the raw image, qcow2
conversion, sign-and-tag, dm-prepare-release, dm-reproducible-buildinfo -- runs
the same code in both modes. The image is simply 1M instead of 100G, so it is
built and packaged for real but contains almost nothing.

### Where the dry-run therefore stops being evidence

- Anything that depends on the image CONTENTS: a 1M image cannot hold a
  filesystem worth booting, so the boot-test lane cannot be replaced by it.
- Anything that depends on a real upload.
- Reproducibility: comparing two dry-run images compares two 1M placeholders.

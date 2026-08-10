# Verifying a reproducible image build

You rebuilt an official image with `--freshness frozen` and want to confirm it
matches ours bit-for-bit -- or find out what differs.

## The model: rebuild and compare, do not trust

Reproducible builds let you verify a published image **without trusting us, the
builder**. You rebuild from the same source and inputs and compare *your* image
to *ours*. There is deliberately no artifact we sign that "proves" the image is
good -- a compromised builder could sign that too. The only trustworthy check is
your own rebuild against our published image.

To rebuild with the same inputs, fetch the signed `<image>.dm-buildinfo` sidecar
(it records the build parameters) and build from the recorded commit; see
`dm-reproducible-verify`.

## Pass / fail: compare the whole-image hash

```
sha512sum your-image.iso
# compare against our published, signed <image>.sha512sums
```

Equal hash means bit-for-bit reproducible. Done.

## If they differ: diffoscope on the two images

To see *which* files differ, run diffoscope on both images. diffoscope unpacks
the container itself (ISO -> squashfs -> files, qcow2 via `qemu-img`, `.ova`
tar -> vmdk), so no manual extraction is needed:

```
diffoscope our-image.iso your-image.iso
```

### On a RAM-limited host

diffoscope only uses significant memory when the images **differ inside a large
binary member** -- identical images short-circuit cheaply. Two mitigations:

- Use **diffoscope >= v302** (Debian trixie-backports:
  `apt-get install -t trixie-backports diffoscope`). It streams the diff instead
  of buffering it whole, avoiding the OOM older versions hit on a large differing
  member (upstream: diffoscope salsa issue #342 / MR !145).
- Bound the diff and keep the temp dir on real disk (not a tmpfs like
  `/run/shm`):

```
TMPDIR=/var/tmp diffoscope \
  --max-diff-input-lines 100000 --max-diff-block-lines-saved 10000 \
  --exclude 'boot/initrd*' --exclude 'boot/vmlinuz*' \
  --text report.txt our-image.iso your-image.iso
```

### The automated comparator on large ISO / OVA images

`dm-reproducible-compare-artifacts --target iso|virtualbox` decides the verdict
from the whole-image sha256 and runs diffoscope only as a best-effort
EXPLANATION. On a real multi-gigabyte DIFFERING image that explanation can still
exhaust memory even with the bounds above. Measured on a 1.4 GB differing `.iso`
pair: diffoscope descends into the image (parsing the filesystem and the unified
kernel image), then exits 2 -- `Out of memory. Diffoscope exiting.` -- under the
tool's address-space cap. The tool degrades gracefully: it prints
`diffoscope could not explain the diff (exit 2); the sha256 mismatch above is the
verdict` and the sha256 result stands. The VERDICT is never affected (it is a
whole-file hash); only the per-file explanation is lost.

The `qcow2` target already avoids this by attaching each image read-only and
handing diffoscope the two MOUNT POINTS, so its working set is one file pair at a
time. Extending that mounted route to `iso`/`virtualbox` would give them a
memory-safe explanation too -- a future improvement, not required for a correct
verdict.

## Reporting a difference

If the images differ and it is not an intentional variation, that is a
reproducibility bug worth reporting: include the diffoscope report and the commit
recorded in the signed `<image>.dm-buildinfo` -- enough for a maintainer to
reproduce and localize without your image.

## See also

- `ci/reproducible-build-twice` -- build an image twice locally and diff it (the
  developer determinism check: sha256 pass/fail, diffoscope on a mismatch).
- `.github/workflows/local-build.yml` -- the CI lane: two independent
  builds compared with diffoscope.
- `dm-reproducible-verify` / `dm-reproducible-buildinfo` -- fetch inputs, rebuild,
  diffoscope.

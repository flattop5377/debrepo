# Publishing a package to this repository

`reprepro` does the work and signs on its own — **except where it cannot**, which is the case
this file exists for. The 2026-09-01 rclone publish hit it, was worked out in a conversation, and
was not written down anywhere afterwards.

## The ordinary case

On a machine holding the secret half of `208BC127…D4FB` directly:

    reprepro -b <repo> includedeb stable <package>_<version>_<arch>.deb
    git add -A && git commit -m "<package> <version>" && git push origin HEAD:master

`SignWith` in `conf/distributions` names the key, and reprepro signs `Release` as it exports.
The push **is** the deploy: `raw.githubusercontent.com/flattop5377/debrepo/master` is the served
tree.

`afirewall/tools/release.sh` does exactly this and needs nothing from this file.

## When gpg is split — reprepro cannot sign, and the error does not say why

On a host using Qubes Split GPG, this is what happens:

    Exporting indices...
    Could not find any key matching '208BC1279656FEEA7C2F9A1D830604D51CA7D4FB'!
    ERROR: Could not finish exporting 'stable'!

**The key is not missing.** reprepro signs through *gpgme*, which spawns `/usr/bin/gpg` itself —
so it bypasses `qubes-gpg-client-wrapper` however that is wired into the shell, and asks a local
keyring that legitimately holds no secret key. Qubes documents the wrapper for programs that call
the `gpg` binary and documents nothing for GPGME, so there is no configuration that fixes this;
it has to be worked around.

The failure is partial and the message says so, easily missed: **the pool and the database were
updated and the indices were not.** From outside, the repository still looks exactly as it did.

### The release script handles this

`afirewall/tools/release.sh` detects it — if `gpg` holds no secret half of the `SignWith` key, it
exports through a copied config directory and signs through the client. Nothing below is needed
for an afirewall release.

### By hand, for anything else

Export without signing, then sign what was exported. **Copy `conf/` rather than editing it** — the
obvious version comments `SignWith` out with `sed` and puts it back, which leaves a window where a
failure, or a `git add -A`, commits a repository configured not to sign itself:

    cd <repo>
    ls pool/main/<l>/<package>/            # the .deb is already here from the failed run

    K=208BC1279656FEEA7C2F9A1D830604D51CA7D4FB
    CONF=$(mktemp -d)
    cp -a conf/. "$CONF/"
    sed -i 's/^SignWith:/#SignWith:/' "$CONF/distributions"
    reprepro --confdir "$CONF" -b . export stable
    rm -rf "$CONF"

    qubes-gpg-client-wrapper --batch --yes --local-user $K \
        --clearsign -o dists/stable/InRelease.new dists/stable/Release
    mv dists/stable/InRelease.new dists/stable/InRelease

    qubes-gpg-client-wrapper --batch --yes --local-user $K \
        --armor --detach-sign -o dists/stable/Release.gpg.new dists/stable/Release
    mv dists/stable/Release.gpg.new dists/stable/Release.gpg

Those two signing commands are the ones in `RESIGN.md`, with the wrapper in place of `gpg`. That
document is framed as a one-time note about the 2026-08-18 rotation; the procedure in it is the
general answer to *sign an export reprepro could not sign*.

### Verify before pushing

    gpg --status-fd=1 --verify dists/stable/InRelease
    gpg --status-fd=1 --verify dists/stable/Release.gpg dists/stable/Release

Each must print `VALIDSIG` with that fingerprint and must **not** print `KEYREVOKED`. Read the
status output rather than the exit code: **`gpg --verify` exits 0 for a signature made by a revoked
key**, and this repository's previous key is revoked rather than deleted. Then commit and push.

## Architectures

`conf/distributions` declares `source i386 amd64`. The description says the packages here are
`Architecture: all` and pure Python — that is a description of what has been published, not a
constraint. rclone is `amd64` and the `binary-amd64` index has always existed.

## What publishing here now means for the fleet

Since 2026-09-01 the ansible fleet's `unattended-upgrades` includes `origin=Flattop5377` in its
`Origins-Pattern`. **A package published here installs itself across the fleet**, on its own
schedule, without anyone converging anything. That is the point — it is how rclone stays current
when upstream ships no apt repository of its own — and it is worth knowing before pushing
something half-tested.

The collector's `package-stale` signal reads this repository's `binary-amd64/Packages` and
compares it against upstream for each package in the ansible inventory's
`tracked_upstream_packages`. It goes red when this repository is behind. Nobody has to remember to
check for a new rclone; that check is what remembers.

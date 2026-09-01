# Re-sign after the 2026-08-18 key rotation

> The two signing commands below are also the general answer to *reprepro could not sign this
> export* — which is what happens on a host using Split GPG, because reprepro signs through gpgme
> and gpgme spawns `/usr/bin/gpg` directly. See `PUBLISH.md`.

`SignWith` and the install instructions are already updated on this branch. The
two signatures below still carry the revoked key `57C45ECD…F917F71F` and must be
replaced on a machine holding the secret half of `208BC127…D4FB`.

**No copy of the public key is served from this repository any more.** It was in
two places — `conf/flattop5377.public.asc` and inline in the `Signed-By:` block of
`conf/flattop5377.sources` — and the rotation updated one of them, so a host
following the README installed the revoked key while the file beside it said
otherwise. Both are gone. `conf/flattop5377.sources` now names
`/etc/apt/keyrings/flattop5377.gpg`, and the README has the operator fetch the key
from a keyserver by fingerprint. A keyserver carries a revocation; a copy checked
in beside the thing it authorises cannot.

**Do not push the instructions without the signatures.** They tell apt to trust
the new key; the signatures are still made by the old one. Landing them alone
makes every host fail verification — fail-closed, so safer than today, but it
breaks `apt update` fleet-wide until the signatures follow.

    cd debrepo
    gpg --batch --yes --local-user 208BC1279656FEEA7C2F9A1D830604D51CA7D4FB \
        --clearsign -o dists/stable/InRelease.new dists/stable/Release
    mv dists/stable/InRelease.new dists/stable/InRelease

    gpg --batch --yes --local-user 208BC1279656FEEA7C2F9A1D830604D51CA7D4FB \
        --armor --detach-sign -o dists/stable/Release.gpg.new dists/stable/Release
    mv dists/stable/Release.gpg.new dists/stable/Release.gpg

`reprepro` signs with `SignWith` on its own when it publishes, so a release cut
through `afirewall/tools/release.sh` does not need the two commands above — they
are for re-signing an existing `Release` without rebuilding one.

Verify before pushing — this must name 208BC127 and must NOT warn about
revocation:

    gpg --verify dists/stable/InRelease
    gpg --verify dists/stable/Release.gpg dists/stable/Release

Then commit and push. `raw.githubusercontent.com/flattop5377/debrepo/master`
is the served tree, so the push *is* the deploy.

Afterwards, on a host that already has the old key installed, the new one has to
be fetched before `apt update` can verify anything — the README has the commands:

    gpg --show-keys /etc/apt/keyrings/flattop5377.gpg   # must be 208BC127, not revoked
    sudo apt update                                     # no NO_PUBKEY, no bad signature

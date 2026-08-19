# Re-sign after the 2026-08-18 key rotation

The anchor and `SignWith` are already updated on this branch. The two signatures
below still carry the revoked key `57C45ECD…F917F71F` and must be replaced on a
machine holding the secret half of `208BC127…D4FB`.

**Do not push the anchor without the signatures.** The anchor tells apt to trust
the new key; the signatures are still made by the old one. Landing the anchor
alone makes every host fail verification — fail-closed, so safer than today, but
it breaks `apt update` fleet-wide until the signatures follow.

    cd debrepo
    gpg --batch --yes --local-user 208BC1279656FEEA7C2F9A1D830604D51CA7D4FB \
        --clearsign -o dists/stable/InRelease.new dists/stable/Release
    mv dists/stable/InRelease.new dists/stable/InRelease

    gpg --batch --yes --local-user 208BC1279656FEEA7C2F9A1D830604D51CA7D4FB \
        --armor --detach-sign -o dists/stable/Release.gpg.new dists/stable/Release
    mv dists/stable/Release.gpg.new dists/stable/Release.gpg

Verify before pushing — this must name 208BC127 and must NOT warn about
revocation:

    gpg --verify dists/stable/InRelease
    gpg --verify dists/stable/Release.gpg dists/stable/Release

Then commit and push. `raw.githubusercontent.com/flattop5377/debrepo/master`
is the served tree, so the push *is* the deploy.

Afterwards, on any host:

    sudo apt update      # must not report NO_PUBKEY or a bad signature

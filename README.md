# Flattop5377's Debian Repository

## Add to APT sources

**The signing key is fetched from a keyserver by fingerprint, not shipped in this
repository.** A copy of a key checked in beside the thing it authorises cannot
express a revocation: the day the key is withdrawn, every host holding that copy
keeps trusting it and nothing says otherwise. A keyserver carries the revocation.
That is the whole reason for the extra step.

The fingerprint is the thing to check. Which keyserver, which mirror — none of it
matters as long as what arrives matches, and `gpg` refuses anything that does not.

    208BC127 9656FEEA 7C2F9A1D 830604D5 1CA7D4FB

### Command line

    sudo mkdir -p /etc/apt/keyrings

    tmp=$(mktemp -d)
    gpg --homedir "$tmp" --keyserver hkps://keys.openpgp.org \
        --recv-keys 208BC1279656FEEA7C2F9A1D830604D51CA7D4FB
    gpg --homedir "$tmp" --export 208BC1279656FEEA7C2F9A1D830604D51CA7D4FB \
        | sudo tee /etc/apt/keyrings/flattop5377.gpg > /dev/null
    rm -rf "$tmp"

    sudo chmod 0644 /etc/apt/keyrings/flattop5377.gpg

    sudo curl -fsSL \
        https://raw.githubusercontent.com/flattop5377/debrepo/master/conf/flattop5377.sources \
        -o /etc/apt/sources.list.d/flattop5377.sources

    sudo apt update

`hkps://keyserver.ubuntu.com` serves the same key if the first is unreachable.

**Why a dearmored `.gpg` and not an armored `.asc`.** apt has accepted armoured
keyrings under `Signed-By:` only since 2.4, so an `.asc` fails on anything older
than Debian 12 — and fails at `apt update`, naming the keyring rather than the
format, which is a poor way to find out. `gpg --export` without `--armor` writes
the binary form every version reads.

**Why the `chmod`.** apt fetches as the unprivileged `_apt` user, so the keyring
has to be world-readable. Under a restrictive umask `tee` will not make it so, and
the repository then fails verification for a reason that has nothing to do with
the key.

### Check what you trusted

    gpg --show-keys /etc/apt/keyrings/flattop5377.gpg

It must print the fingerprint above, and must **not** report the key revoked. If
it does, stop — do not run `apt update` against this repository until a new
fingerprint is published here.

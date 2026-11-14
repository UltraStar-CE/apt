# USCE APT repository

The publication workflow builds a signed APT repository from published GitHub
release packages. The package target is **Ubuntu 22.04 (Jammy), amd64**, matching
the native DEB build from UltraStar-CE/USCE#313.
Other Ubuntu series, Debian releases and architectures need their own package
build and installation checks before being added. An AppImage remains available
for other Linux installations.

The archive is hosted at [ultrastar-ce.github.io/apt](https://ultrastar-ce.github.io/apt/).

## Installation

On Ubuntu 22.04 amd64:

```sh
(
set -e
temporary=$(mktemp -d)
trap 'rm -rf -- "$temporary"' EXIT
expected=3ABECBE6C88AA77D8356596A7CBAC6F702CA6EE5
sudo apt install ca-certificates curl gnupg
sudo install -d -m 0755 /etc/apt/keyrings
curl --fail --location --output "$temporary/usce.asc" \
  https://ultrastar-ce.github.io/apt/usce-archive-keyring.asc
actual=$(gpg --batch --show-keys --with-colons "$temporary/usce.asc" |
  awk -F: '$1=="pub" {primary=1; next} primary && $1=="fpr" {print $10; primary=0}')
if [ "$actual" != "$expected" ]; then
  printf 'USCE archive key fingerprint mismatch. Check the project instructions.\n' >&2
  exit 1
fi
sudo install -m 0644 "$temporary/usce.asc" /etc/apt/keyrings/usce.asc
sudo tee /etc/apt/sources.list.d/usce.sources >/dev/null <<'EOF'
Types: deb
URIs: https://ultrastar-ce.github.io/apt
Suites: jammy
Components: main
Architectures: amd64
Signed-By: /etc/apt/keyrings/usce.asc
EOF
sudo apt update
sudo apt install usce
)
```

Later versions arrive through normal APT updates. The package name is `usce`,
so it does not replace the distribution's `ultrastardx` package.

## One-time maintainer setup

1. Stable release downloads must include `USCE-linux-deb-VERSION.deb` and its
   entry in `SHA256SUMS`.
2. Enable **Settings / Pages / Build and deployment / GitHub Actions** for
   `UltraStar-CE/apt`. Check the `github-pages` deployment environment rules.
3. Create a dedicated archive signing key on a trusted local machine, keeping
   a backup of the key and its revocation certificate. For example:

   ```sh
   gpg --quick-generate-key 'USCE package archive' rsa3072 sign 2y
   gpg --list-secret-keys --with-fingerprint
   gpg --armor --export-secret-keys FULL_PRIMARY_FINGERPRINT > usce-signing-key.asc
   ```

   Add its armored secret export as the Actions secret `APT_SIGNING_KEY` and
   its passphrase as `APT_SIGNING_PASSPHRASE`. Never commit the secret export.
   An unprotected dedicated key can omit the passphrase secret.
4. Set the repository variable `APT_SIGNING_FINGERPRINT` to the full 40-digit
   primary-key fingerprint. This enables publication. Record that public
   fingerprint here before advertising the install command to users.
5. Run **Publish signed APT repository** once from `main`, or wait for its hourly check after publishing a stable USCE release. No workflow publishes a
   draft release on the maintainer's behalf.

## Publication and maintenance

`apt-repository.yml` checks published USCE releases hourly, or runs manually
from `main`. It checks out trusted `main`, enumerates all release pages, ignores
drafts/prereleases, and checks DEB bytes against each release's checksum file.
Releases without a DEB are omitted; they never appear as an available package.

The newest twelve available stable package versions are retained, within an
800 MiB package budget to fit GitHub Pages. Packages use versioned
pool paths; APT metadata is rebuilt and published as one Pages deployment.
The signing job has no repository write permission and uses a temporary GnuPG
home. Only the separate deployment job receives Pages/OIDC write permissions.
Pull-request tests receive no signing secrets or deployment permissions.

Metadata expires after 180 days. A manual publication renews it without creating
a game release. Keep the archive key valid; changing its primary key requires
an explicit client trust/key migration, not just replacing the server key file.
If any preparation, verification or signing step fails, deployment does not run
and the previously published site remains in place.

`apt-repository-check.yml` tests a local signed repository with a real APT
client and a disposable test key. It downloads a package without installing or
running its contents, and verifies rejection of signature tampering. Run locally:

```sh
python3 -m unittest discover -s tools -p test_apt_repository.py -v
```

# affectra-demo

A small ESPHome device whose CI builds the firmware and writes its SBOM, for
trying out [Affectra](https://github.com/affectra/affectra).

## The device

[`firmware/climate-sensor.yaml`](firmware/climate-sensor.yaml): an ESP32
(`esp32dev`) reading a BME280 temperature, humidity and pressure sensor over I²C,
built with ESPHome 2024.10.3 on the ESP-IDF framework (ESP-IDF 4.4.8).

## CI

[`.github/workflows/firmware.yml`](.github/workflows/firmware.yml) runs on every
push and pull request:

1. Builds the firmware in the pinned ESPHome image.
2. Writes a CycloneDX 1.6 SBOM with Espressif's
   [`esp-idf-sbom`](https://github.com/espressif/esp-idf-sbom). It lists ESP-IDF and
   what it bundles (FreeRTOS, lwIP, ...) with their CPEs; libraries compiled but not
   linked into the firmware are left out.
3. Uploads the `climate-sensor` artifact (kept 90 days):

   | File | |
   |---|---|
   | `climate-sensor.cdx.json` | The SBOM |
   | `climate-sensor.cdx.json.sigstore.json` | SBOM attestation bundle (not on pull requests) |
   | `climate-sensor.provenance.sigstore.json` | Build provenance bundle (not on pull requests) |
   | `climate-sensor.factory.bin` | Full image, flash over USB at offset `0x0` |
   | `climate-sensor.ota.bin` | Over-the-air update |

### Signatures

On pushes to `main`, tags and manual runs (not pull requests) the build signs two
GitHub [artifact attestations](https://docs.github.com/actions/security-for-github-actions/using-artifact-attestations)
for both firmware images, keyless with [Sigstore](https://www.sigstore.dev). They name
this repository, workflow and commit, and are recorded in Sigstore's public
transparency log:

- **Build provenance:** this workflow built these images. Saved as
  `climate-sensor.provenance.sigstore.json`.
- **SBOM:** this SBOM describes exactly these images. Saved as
  `climate-sensor.cdx.json.sigstore.json` and sent to Affectra (below).

Check a downloaded image with:

```sh
gh attestation verify climate-sensor.ota.bin --repo affectra/affectra-demo
gh attestation verify climate-sensor.ota.bin --repo affectra/affectra-demo \
  --predicate-type https://cyclonedx.org/bom   # also shows the attested SBOM
```

With the bundle files, `gh attestation verify` doesn't fetch the attestation from
GitHub: add
`--bundle climate-sensor.provenance.sigstore.json` (or the SBOM's bundle with the
`--predicate-type` above).

Attestations need the repository to be public (or on GitHub Enterprise Cloud), and
Affectra accepts them from public repositories only.

### Releases

A tag `v*` also publishes a [GitHub release](https://github.com/affectra/affectra-demo/releases)
with both firmware images, the SBOM, both attestation bundles and `SHA256SUMS`.
Artifacts expire after 90 days; releases stay, so each version can be downloaded and
checked for as long as it is supported. Running the job again replaces the release's
files.

### Upload to Affectra

On a tag `v*` the SBOM goes to Affectra with its attestation, into product
**Climate Sensor**, hardware revision **esp32dev**, version = the tag. Affectra checks
the attestation, marks the SBOM **signed** because the same trusted repository made
it, and lists the two firmware images it describes. To turn that on:

1. In Affectra, as an organization admin: **Trusted publishers**, trust this
   repository (optionally only `firmware.yml` and `refs/tags/v*`; leave the
   environment empty, since attestations carry none). Optionally switch on **Accept
   only SBOMs signed by a trusted repository**.
2. In this repository: **Settings → Secrets and variables → Actions → Variables**,
   set `AFFECTRA_URL` (your instance) and `AFFECTRA_ORG` (the organization slug).

Without `AFFECTRA_URL` the upload job is skipped. The build provenance attestation is
not sent; Affectra does not use it yet.

ESP-IDF 4.4.8 has known vulnerabilities (e.g. CVE-2025-55297, CVE-2025-66409), so
the release in Affectra has findings to look at and assess.

## Build locally

Needs Docker or Podman; the first build takes a few minutes.

```sh
docker run --rm -v "$PWD":/src -w /src --entrypoint sh \
  ghcr.io/esphome/esphome:2024.10.3 firmware/build.sh
```

The firmware and SBOM land in `dist/`.

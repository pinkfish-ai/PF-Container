# Open-source license notices

These files accompany the Pinkfish self-hosted container distribution and satisfy the open-source license obligations for the binaries shipped out of band (`pinkconnect-v0.3.0`, `mcpfarm-v0.3.0`).

| File | Purpose |
| --- | --- |
| `THIRD-PARTY-NOTICES-pinkconnect.txt` | Component + version + license manifest for the PinkConnect image (application + Debian base-OS). Satisfies permissive-license (MIT/Apache-2.0/BSD/ISC) attribution. |
| `THIRD-PARTY-NOTICES-mcpfarm.txt` | Same, for the MCP-farm image. |
| `WRITTEN-OFFER-FOR-SOURCE.txt` | GPL/LGPL written offer for the Debian base-OS packages, pointing to Debian's published sources. |

Generated 2026-06-01 from the image SBOMs (Trivy/CycloneDX). The application layer is permissively licensed; no AGPL/SSPL, and no LGPL `libvips`/`sharp`. The only copyleft present is the standard Debian base-OS system packages (GPL/LGPL), included by mere aggregation.

**Maintenance:**
- Regenerate these from the SBOM whenever an image version changes, and update the version in the filenames / `WRITTEN-OFFER-FOR-SOURCE.txt`.
- Recommended: also bake these files into each image (e.g. `/usr/share/doc/pinkfish/`) so they travel with the binary as well as the distribution repo.

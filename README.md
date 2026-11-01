# ruio-f286

Notes on the Ruio F286, a 4G feature phone built on the Unisoc UMS9117 (T107) with MocorOS.
They cover the hardware, the boot chain, the stock firmware's internals and how to back the phone
up, written down while adding drivers to FPLinux and writing apps for the stock OS.

Start at [docs/index.md](docs/index.md).

The pages are plain Markdown. To serve them with GitHub Pages, set the Pages source to the `docs/`
folder of the default branch; `docs/_config.yml` picks the default theme.

Everything here was found on one phone with one firmware build. Addresses only apply to that
build (stock `mmi_app` sha256 starts `5d534329`).

## Licence

CC BY 4.0, see [LICENSE](LICENSE).

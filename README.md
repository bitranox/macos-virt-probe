# macos-virt-probe

Runs [ipscout](https://github.com/bitranox/ipscout)'s root-only macOS tests on GitHub's hosted
macOS runners.

ipscout's own CI cannot run them. Its workflows come from a shared CI template, and its test step
runs pytest unprivileged, while these tests need root, QEMU and a guest VM. This repo is public
because macOS runner minutes are free only for public repositories.

## What the workflow does

The `probe` workflow runs on every push to `main` and on manual dispatch, on four runners:
`macos-latest` and `macos-26` (Apple Silicon), `macos-15-intel` and `macos-26-intel` (Intel).
Each job:

1. Prints the runner's hypervisor flags (`kern.hv_support`, `kern.hv_vmm_present`) and CPU.
2. Installs QEMU with Homebrew.
3. Starts QEMU twice for 20 seconds: once with software emulation (`-accel tcg`), the control,
   and once with Apple's Hypervisor.framework (`-accel hvf`). A guest that starts runs until the
   timeout (exit 124); a refused accelerator exits at once with its own error.
4. Boots a paused x86 guest on `-netdev vmnet-shared`, so macOS creates `bridge100`, adds the
   guest's `vmenetN` port and serves DHCP itself. It runs `ipscout observe-dhcp` and `tcpdump`
   on that member port, resumes the guest and prints both, then runs `observe-dhcp` on
   `bridge100` itself and prints its exit code (2 when refused).
5. Runs `test_dhcp_macos_vmnet_live.py` as root three times in a row. The file is a copy of
   ipscout's `tests/test_dhcp_macos_vmnet_live.py`, run against the ipscout release the workflow
   installs from PyPI.

Steps 3 to 5 always exit 0, so a green job proves nothing. Read the `RESULT` lines in the log:
`rc=124` for a QEMU boot that ran, `RESULT observe rc=0` for a capture that answered,
`RESULT bridge rc=2` for the refusal, and `RESULT pytest round=N rc=0` for each test round.

## What it has shown

Measured 2026-10-28:

- The Intel runners report `kern.hv_support=1` and run an `-accel hvf` guest. The Apple Silicon
  runners report `0` and refuse hvf with `HV_UNSUPPORTED`; `-accel tcg` works on all four.
- On all four runners, ipscout's capture on the guest's `vmenetN` port returned the address
  macOS's DHCP server leased the guest, and a capture on `bridge100` was refused. `tcpdump` on
  the same port saw the same requests and replies.

## Keeping it current

When ipscout changes the test, copy `tests/test_dhcp_macos_vmnet_live.py` over the file here.
After an ipscout release, raise the `ipscout==` pin in `.github/workflows/probe.yml`. The push
starts a run.

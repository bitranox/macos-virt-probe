# macos-virt-probe

A throwaway probe: can a GitHub-hosted macOS runner start a hardware-accelerated guest VM?

The `probe` workflow (manual dispatch) runs on the free macOS runners, Apple Silicon and Intel.
It prints the runner's hypervisor flags (`kern.hv_support`, `kern.hv_vmm_present`), then starts
QEMU twice for 20 seconds each: once with software emulation (`-accel tcg`), the control that must
keep running until the timeout, and once with Apple's Hypervisor.framework (`-accel hvf`). A guest
that starts keeps running until the timeout (exit 124); a refused accelerator exits at once with
its own error.

It exists to decide whether the macOS DHCP capture of
[ipscout](https://github.com/bitranox/ipscout) can be tested against a guest booting behind a
macOS bridge on a CI runner. It can be deleted once that is answered.

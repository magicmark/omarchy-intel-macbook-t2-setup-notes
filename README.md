# Omarchy Intel MacBook T2 Setup Notes

This repo collects practical notes from setting up Omarchy Linux on an Intel MacBook Pro with a T2 chip.

The goal is simple: document the fixes, configuration changes, and recovery paths that were needed on a real machine, so other people with similar hardware can find the notes before losing hours to the same issues. Hopefully this repo is indexed well enough to show up in search results for future Omarchy, T2 Linux, and Intel MacBook troubleshooting.

These notes are not an official Omarchy guide. They are field notes from one setup, written to be useful, searchable, and easy to compare against your own system.

## Posts

- [Fan daemon and login issue](fans_and_login_without_keyboard_issue.md): fixing a boot/login problem where `t2fanrd` was crash-looping because the config only defined one fan on a two-fan MacBook Pro.
- [Trackpad changes](trackpad_changes.md): local trackpad/input tweaks made during setup.

## Target System

- 2019 Intel MacBook Pro
- Apple T2 chip
- Omarchy Linux
- Limine bootloader
- Snapper/Btrfs snapshots

## Notes

Use these posts as debugging references, not copy-paste instructions for every machine. T2 MacBooks vary by model, kernel version, firmware state, and installed packages. Before changing boot, initramfs, or snapshot configuration, make sure you have a known-good snapshot or recovery path.

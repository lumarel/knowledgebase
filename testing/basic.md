# Basic testing routines

These are the most basic tests which will be (if the time is there) run on every new release:

- Installation of multiple new systems (minimal, graphical server, workstation, BIOS, UEFI, ...), this is just a very basic check, if the main installation works without errors on VMware ESXi and an old baremetal machine
- Installation on aarch64 on ESXi on ARM and if already available directly on a Raspberry Pi 4
- Upgrading the pre-setup systems (all different variations and VM hardware configurations on VMware ESXi)
- Tests with SecureBoot (more details in the secureboot docs)
- Kickstart tests (scripts pending, up to now only the main templating pipeline)
- Visual inspections of the graphical server/workstation installs
- Run of all OpenQA tests locally (more detaily in the openqa docs)

## Tests for the future?

Like [these](https://pagure.io/fedora-qa/qa-misc/tree/master), or [these](https://pagure.io/fedora-qa/modularity_testing_scripts/tree/master), or [one of these](https://fedoraproject.org/wiki/Category:Test_Cases)

- ISO repoclosure tests to verify all dependencies inside of a ISO and find broken ones
- Basic ISO checksum tests, if these are needed?
- More basic installs, maybe even inside of a chart, where every major possible options get's tested by somebody (BIOS, UEFI, platform (ESXi, KVM, Xen, Parallels, really old hardware, current hardware, really new hardware, ...), major package groups, + minor package groups, all ISOs, all current disk types, ...)
- Check of basic system functions:
  - mount/umount/creation of disks/partitions
  - logging system check (journal + files)
  - install/remove/update of packages
  - failed services
- Modularity checks (not sure how, but maybe some installations -> reset -> upgrade tests)
- Per application guides and tests

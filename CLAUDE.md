# CLAUDE.md — ocp-diag-autoval-ssd

SSD/NVMe test suite built on the AutoVal framework (PyPI name `ocptv-autoval-ssd`, import name `autoval_ssd`). Depends on the core framework `ocptv-autoval==0.0.1` — expected to be cloned alongside at `../ocp-diag-autoval` (see that repo's CLAUDE.md for `TestBase`, validators, runner and CLI).

## Layout (`src/autoval_ssd/`)
- `tests/<test_name>/` — one directory per test: `<test_name>.py` (exactly one test class), `<test_name>.rst` (docs), and one or more control JSONs (`control*.json` or descriptive names like `fio_fb_stress_6h.json`). Tests include `nvme_cli`, `nvme_format`, `nvme_sanitize`, `nvme_ns_resize`, `fio_fb`, `fio_synth_flash`, `fio_internal_flush`, `drive_data_integrity`, `drive_md5_verify`, `drive_cache_check`, `flash_firmware_update`, `smart_impact_on_perf`, `namespace_utilization_test`, `sed_check`, `ber_test`, `fsync`, `iogo`.
- `tests/storage_hw_eng/` — separate hardware-engineering sub-suite with its own `libs/` (data_types, ssd_lib, component_common_lib, ext_test_base) and `tests/ssd_tests/` (synthflash workloads incl. FDP/DIX, iogo, cachebench, fileappend).
- `lib/utils/storage/storage_test_base.py` — `StorageTestBase(TestBase)`: base class for nearly every test here. Handles drive discovery, boot-drive exclusion, test-drive allocation, required RPM install (fio, smartmontools, hdparm, sdparm, sg3_utils, …), SMART/health/firmware checks and drive log collection. Override `storage_test_setup()` (call `super()` first) rather than `setup()`.
- `lib/utils/storage/` — `drive.py`, `storage_device_factory.py`, `storage_utils.py`, `smart_validator.py`, firmware update utils, and `nvme/` (`nvme_utils.py`, `nvme_drive.py`, `fdp_utils.py`, `dix_utils.py`, `latency_monitor_utils.py`, `lmparser.py`, `vendor/`), plus `sas/`, `sata/`.
- `lib/utils/` — `fio_runner.py`, `fio/`, `jobfile_templates/` (`*.fio`, `*.job`), disk/filesystem/md/pci/sg/sdparm/hdparm/lsi/sed/switchtec/system utils.
- `cfg/` — `test_suites/*.yaml` (suite definitions: each entry has `name`, `cmd` = dotted test module, `args.test_params`, `timeout`, `min_hosts`/`max_hosts`, etc.), SMART validation JSONs (`nvme_smart/`, `nvme_smart_fdi/`), FDP/latency/workload configs.
- `unittest/` — `test_*.py` unit tests with mocks in `unittest/mock/` (`lib/mock_host.py` etc.; canned command output in `mock/util_outputs/`). Mocks map exact command strings (`CMD_MAP`) to results/files.
- `tools/` — `fsync.c`, `ioT6.go` shipped as package data and deployed to DUTs.
- `docs/` — `rpm_repo_hosting.md`, `gluster_configuration.md` (both written for CentOS Stream 9). `documentation/autoval_ssd.md` — suite overview.
- CentOS Stream 9 → 10 migration plan (controller and DUTs): `../ocp-diag-autoval/documentation/centos10_migration_plan.md`.

## Setup
```bash
python -m venv env && source env/bin/activate && pip install build
(cd ../ocp-diag-autoval && python -m build && pip install ./dist/ocptv_autoval-0.0.1.tar.gz)
python -m build && pip install --no-deps ./dist/ocptv_autoval_ssd-0.0.1.tar.gz
```
For development without rebuilding, use `PYTHONPATH=src:../ocp-diag-autoval/src` and install core's runtime deps (`setuptools==70.0.0 ocptv==0.1.6 six iopath paramiko`).

Undeclared dependencies (not in either `pyproject.toml` yet): `PyYAML` (core `suite_runner.py`, needed for `--suite`), and `attrs`, `pandas`, `packaging`, `PyYAML` for `tests/storage_hw_eng/`. Install them explicitly: `pip install PyYAML attrs pandas packaging`. On Python 3.12+ venvs, `setuptools` must also be installed explicitly (core uses `pkg_resources`).

## Unit tests
`unittest/` has no `__init__.py`, so `unittest discover` fails ("Start directory is not importable"). Pass modules explicitly, **from `src/`** (mock output paths are relative to `src/`, e.g. `autoval_ssd/unittest/mock/util_outputs/id_ctrl`), with `SITE_SETTINGS` set:
```bash
cd src
export SITE_SETTINGS=../../ocp-diag-autoval/src/autoval/cfg/site_settings/site_settings.json
export PYTHONPATH=.:../../ocp-diag-autoval/src
python -m unittest autoval_ssd.unittest.test_nvme_utils          # one module
python -m unittest $(ls autoval_ssd/unittest/test_*.py | sed 's|/|.|g; s|\.py$||')   # all
```
Known state (Nov 2026, Python 3.10 and 3.12): everything passes except 3 pre-existing failures in `test_system_utils` (expected `[SYSTEM INFO ERROR]` vs core's `[HAVOC SYSTEM INFO ERROR]` prefix, plus a dmidecode parse mismatch). `test_nvme_drive` takes ~80s. Don't treat those as regressions you caused.

When you add or change a util, add/adjust a unit test with a `CMD_MAP` entry or a file in `mock/util_outputs/`; the mocked command string must match exactly what the util runs.

## Running tests against hardware
```bash
export SITE_SETTINGS=path/to/site_settings.json   # needs "yum_repo": "autoval-tools" for RPM installs
python -m autoval.autoval_test_runner autoval_ssd.tests.nvme_cli.nvme_cli \
    --config ./hosts.json --test_control src/autoval_ssd/tests/nvme_cli/control.json \
    [--args '{"include_boot_drive": true}']
python -m autoval.autoval_test_runner --suite src/autoval_ssd/cfg/test_suites/sanity_ussdt_on_diffs.yaml --config ./hosts.json
```
- Control files need `boot_drive_physical_location` in BDF form (`0000:64:00.0`); `--args` overrides control-file values; `include_boot_drive` / `only_boot_drive` control boot-drive use.
- **These tests are destructive** (format, sanitize, namespace resize, firmware flash, raw fio writes). Never run them against a host unless the user explicitly asks and names the host.
- Tests pull RPMs (fio, fio-synth, …) from a DNF repo on the DUT (see `docs/rpm_repo_hosting.md`).

## CentOS Stream 10 notes
The controller and DUTs are moving from CentOS Stream 9 to 10 (Python 3.12, kernel 6.12). When working on anything OS-dependent:
- `SystemUtils.install_rpms` runs `dnf --disablerepo=* --enablerepo=<yum_repo>`, so every package *and its dependencies* must exist in the `autoval-tools` repo for that DUT's release (el9 and el10 repos are kept separate during the rollout).
- Version pins are el9-specific: `tests/nvme_cli/nvme_cli_with_nvme_version.json` (`nvme-cli-2.8-1.hs.el9`) and `lib/utils/storage/nvme/fdp_utils.py` (`nvme-cli-2.10.2`, `libnvme-1.10`). Update them together with the repo contents.
- Mocks in `unittest/mock/util_outputs/` are el9 captures. When el10 tool output differs, add el10 cases rather than replacing the el9 ones while both fleets run.
- `fsync` compiles with `gcc` on the DUT and tests don't install it; `iogo` installs `golang` from the repo.
- GlusterFS isn't available on Stream 10; NFS is the recommended replacement for `/shared/autoval` (not decided yet). Avoid adding new Gluster-specific code.

## Lint / format
Same as core: `make format` (ufmt), `make lint_all` (ufmt check), `make lint` (pre-commit, ufmt v2.0.0). Format before committing.

## Conventions
- Files start with `#!/usr/bin/env python3` and `# pyre-unsafe`.
- New test: new dir under `tests/`, subclass `StorageTestBase` (or `TestBase` if not drive-oriented), read params with `self.test_control.get("key", default)` in `__init__`, implement `execute()`, use `self.validate_*` (forwarded to core `AutovalUtils`) with `component=COMPONENT.STORAGE_DRIVE`/etc. and `ErrorType`, add a `.rst` and at least one control JSON. If it belongs in a suite, add an entry to the relevant `cfg/test_suites/*.yaml` with `cmd: autoval_ssd.tests.<dir>.<module>`.
- New non-Python assets must match the package-data globs in `pyproject.toml` (`*.json, *.fio, *.job, *.go, *.c`) or they won't ship.
- Code is synced from an internal upstream ("havoc" names, e.g. `/mnt/havoc` mount point); recent commits like "Sync storage utils, configs, and tests" are bulk syncs. Keep changes minimal and don't rename those identifiers unless asked.
- Changes that need core framework changes go in `../ocp-diag-autoval`; keep the two repos compatible.

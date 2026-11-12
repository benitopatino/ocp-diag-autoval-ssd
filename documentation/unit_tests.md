# AutoVal SSD Unit Tests

Written Nov 11, 2026. ocp-diag-autoval-ssd ships a unit-test suite of **19 modules and 273 tests** in `src/autoval_ssd/unittest/`. The tests use a mock host that returns canned command output, so they need **no DUT, no drives and no root access**. They run anywhere the Python code runs, in about 20 seconds plus about 80 seconds for `test_nvme_drive`.

SSD code calls into the core framework (`ocp-diag-autoval`) all the time, so this suite also exercises a good part of core. Core has no unit tests of its own (the `test_*.py` files under its `lib/` are framework modules, not tests).

## Quick start

The two repos must be cloned side by side:

```text
repos/
├── ocp-diag-autoval/
└── ocp-diag-autoval-ssd/
```

**1. Python environment.** Either use the AutoVal venv from the setup guide (core and SSD installed), or point `PYTHONPATH` at both source trees and install core's runtime dependencies:

```bash
python3 -m venv ~/autoval-env && source ~/autoval-env/bin/activate
pip install setuptools==70.0.0 ocptv==0.1.6 six iopath paramiko PyYAML attrs pandas packaging
```

**2. Run from `src/`**, with `SITE_SETTINGS` and `PYTHONPATH` set:

```bash
cd ocp-diag-autoval-ssd/src
export SITE_SETTINGS=../../ocp-diag-autoval/src/autoval/cfg/site_settings/site_settings.json
export PYTHONPATH=.:../../ocp-diag-autoval/src

# everything
python -m unittest $(ls autoval_ssd/unittest/test_*.py | sed 's|/|.|g; s|\.py$||')

# everything except the slow nvme_drive module
python -m unittest $(ls autoval_ssd/unittest/test_*.py | grep -v test_nvme_drive | sed 's|/|.|g; s|\.py$||')

# one module
python -m unittest autoval_ssd.unittest.test_nvme_utils

# one test class, or one test method (-v lists each test as it runs)
python -m unittest -v autoval_ssd.unittest.test_sdparm_utils.SdparmUtilsUnitTest
python -m unittest -v autoval_ssd.unittest.test_sdparm_utils.SdparmUtilsUnitTest.<test_method_name>
```

Why these details matter:

- **Run from `src/`.** Mock output files are read by relative path (`autoval_ssd/unittest/mock/util_outputs/...`), so running from anywhere else gives file-not-found errors.
- **List the modules explicitly.** `python -m unittest discover` fails with "Start directory is not importable" because `unittest/` has no `__init__.py`.
- **Set `SITE_SETTINGS`.** Code under test reads site settings. The default file in the core repo is enough; no real paths are touched.

## Expected result

On Python 3.10 and 3.12 (checked Nov 2026), everything passes **except 3 failures in `test_system_utils`** that predate any of our changes:

| Test | Why it fails |
|---|---|
| `test_match_in_dmidecode` | Expects the error prefix `[SYSTEM INFO ERROR]`, but core now raises `[HAVOC SYSTEM INFO ERROR]` |
| `test_parse_dmidecode_output` | dmidecode parse result doesn't match the expected list |
| `test_get_acpi_interrupt` | Parsed output doesn't match the expected dictionary |

Treat anything beyond these three as a regression.

## What each module covers

| Module | Tests | Code under test |
|---|---|---|
| `test_disk_utils` | 32 | `lib/utils/disk_utils.py`: block devices, partitions, mounts, disk I/O helpers |
| `test_nvme_drive` | 33 | `lib/utils/storage/nvme/nvme_drive.py`: NVMe drive object, SMART and health data (slowest, ~80 s) |
| `test_sg_utils` | 31 | `lib/utils/sg_utils.py`: sg3_utils commands (`sg_inq`, `sg_readcap`, `sg_logs`, ...) |
| `test_nvme_utils` | 26 | `lib/utils/storage/nvme/nvme_utils.py`: nvme-cli wrappers and parsers |
| `test_pci_utils` | 26 | `lib/utils/pci_utils.py`: `lspci` / `setpci` parsing, PCI topology |
| `test_system_utils` | 16 | `lib/utils/system_utils.py`: package manager detection, RPM install/query, dmidecode, ACPI |
| `test_drive` | 13 | `lib/utils/storage/drive.py`: the base drive class |
| `test_filesystem_utils` | 12 | `lib/utils/filesystem_utils.py`: `mkfs`, mount, filesystem handling |
| `test_scrtnycli_utils` | 12 | `lib/utils/scrtnycli_utils.py`: Broadcom scrtnycli wrapper |
| `test_md_utils` | 11 | `lib/utils/md_utils.py`: software RAID (`mdadm`) |
| `test_hdparm_utils` | 10 | `lib/utils/hdparm_utils.py`: hdparm wrappers |
| `test_nvme_resize_utils` | 10 | `lib/utils/storage/nvme/nvme_resize_utils.py`: namespace resize |
| `test_storage_utils` | 9 | `lib/utils/storage/storage_utils.py`: drive discovery and shared storage helpers |
| `test_fdp_utils` | 8 | `lib/utils/storage/nvme/fdp_utils.py`: NVMe Flexible Data Placement |
| `test_sdparm_utils` | 6 | `lib/utils/sdparm_utils.py`: write cache (WCE) and read-ahead (DRA) settings |
| `test_lsi_utils` | 5 | `lib/utils/lsi_utils.py`: LSI HBA tool wrapper |
| `test_storage_device_factory` | 5 | `lib/utils/storage/storage_device_factory.py`: picking the drive class for a device |
| `test_sed_util` | 4 | `lib/utils/sed_util.py`: self-encrypting drive (`sedutil-cli`) |
| `test_smart_validator` | 4 | `lib/utils/storage/smart_validator.py`: SMART threshold validation |

## How the mocks work

Everything lives under `src/autoval_ssd/unittest/mock/`:

| Path | Purpose |
|---|---|
| `lib/mock_host.py` | `MockHost`: stands in for a real host. `run()` / `run_get_result()` look the command up in a command map instead of running it |
| `lib/mock_connection_dispatcher.py`, `lib/mock_openbmc.py` | Fake connection and BMC objects the mock host needs |
| `lib/mock_autoval.py`, `lib/mock_test_base_init.py`, `lib/mock_threadpool_executor.py` | Helpers for code that touches `AutovalUtils`, `TestBase` or thread pools |
| `util_outputs/` | Canned command output captured from real systems (`id_ctrl`, `nvme_list.json`, `lsblk_J`, `lspci_vvv_1`, `smartctl_x_sda`, ...) |
| `testbed/testbed.json` | Mock testbed / host description |

### The command map

Each test module defines a `cmd_map`: a list of entries, one per command the code under test will run.

```python
cmd_map = [
    {"cmd": "sdparm --get WCE /dev/sda", "result": "WCE         1  [cha: y, def:  1]"},
    {"cmd": "nvme id-ctrl /dev/nvme0", "file": "id_ctrl"},             # from util_outputs/
    {"cmd": "nvme format /dev/nvme1n1", "result": "", "return_code": 1},
    {"cmd": "cat /proc/foo", "result": ["first", "second"], "iterate": True},
]
```

| Key | Meaning |
|---|---|
| `cmd` | **Must match the command string exactly**, character for character, as the code builds it. An unmatched command returns empty output with exit code 0, which often fails quietly |
| `result` | Literal stdout to return. A `CmdError` instance here is raised instead, to test error paths |
| `file` | Return the contents of `util_outputs/<file>` (used instead of `result` for long output) |
| `return_code` | Exit code to report (default 0) |
| `iterate` | With a list in `result`, return the next item on each call (for polling loops) |

A test builds the host with `MockHost(cmd_map)` and passes it to the utility under test. `update_cmd_map(cmd, output, is_file=False, return_code="0")` changes or adds an entry inside one test, and `revert_cmd_map(original)` restores it. Tests also use `unittest.mock.patch` for functions that shouldn't run at all.

## Adding or changing a test

When you add or change a utility, add or adjust its unit test in the same change:

1. Run the real command on a DUT and save its output. Short output goes in the `result` string. Long output goes in a new file under `mock/util_outputs/`.
2. Add a `cmd_map` entry whose `cmd` is **exactly** the string the code runs (copy it from the code or a debug log).
3. Write the test against `MockHost(cmd_map)`:

```python
#!/usr/bin/env python3
# pyre-unsafe
import unittest

from autoval_ssd.lib.utils.sdparm_utils import SdparmUtils
from autoval_ssd.unittest.mock.lib.mock_host import MockHost

cmd_map = [
    {"cmd": "sdparm --get WCE /dev/sda", "result": "WCE         1  [cha: y, def:  1]"},
]


class SdparmWceTest(unittest.TestCase):
    def setUp(self):
        self.host = MockHost(cmd_map)

    def test_write_cache_enabled(self):
        # call the utility exactly as the tests do; the mock returns the canned output
        ...
```

4. Run the module, then the whole suite, and check only the 3 known failures remain.
5. `make format` before committing (ufmt).

## CentOS Stream 10 notes

- The canned outputs in `util_outputs/` were captured on CentOS Stream 9 tool versions. Kernel 6.12 and newer nvme-cli, smartctl, lspci, dmidecode, hdparm and sdparm can format output differently.
- When el10 output differs, **add** el10 cases (for example `id_ctrl_el10`) with their own tests rather than replacing the el9 files. Both kinds of DUT run side by side during the rollout.
- A quick way to find differences: on an el10 DUT, run the commands from the `cmd_map` entries and diff the output against the matching `util_outputs/` file.
- Some mocks contain el9 strings on purpose, such as `bash-5.1.8-6.1.hs+fb.el9.x86_64` in `test_system_utils`. These are test data and don't need changing.

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Start directory is not importable` | Used `unittest discover` | List modules explicitly (commands above) |
| `FileNotFoundError: autoval_ssd/unittest/mock/util_outputs/...` | Not running from `src/` | `cd ocp-diag-autoval-ssd/src` |
| `No module named 'autoval'` | Core not on the path | `export PYTHONPATH=.:../../ocp-diag-autoval/src`, or install core into the venv |
| `Site setting ... not found` | `SITE_SETTINGS` unset | Export it as shown above |
| `No module named 'pkg_resources'` | setuptools missing (Python 3.12 venv) | `pip install setuptools==70.0.0` |
| `No module named 'yaml'` / `attr` / `pandas` | Undeclared dependencies | `pip install PyYAML attrs pandas packaging` |
| A new test passes but asserts nothing useful | The `cmd` string didn't match, so the mock returned empty output | Compare the exact command the code builds with the `cmd_map` entry |

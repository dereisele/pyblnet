# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

PyBLNET is a Python client for the BL-NET gateway by Technische Alternative, which sits in front of a UVR1611 heating controller. It reads analog/digital values and can set switches to ON/OFF/AUTO. Packaged with flit (`pyproject.toml`); runtime deps are `requests` and `htmldom`. CI tests Python 3.8–3.13. Pushing a `*.*.*` tag triggers the release workflow (GitHub release plus PyPI via flit). When releasing, bump the version in both `pyproject.toml` and `VERSION` in `pyblnet/__init__.py`.

## Commands

```bash
pip install -e '.[test]'            # or: uv sync / use the existing .venv
pytest tests                        # full suite
pytest tests/test_web.py::BLNETWebTest::test_blnet_fetch   # single test
coverage run --source=pyblnet -m pytest tests && coverage report   # what CI runs
ruff check pyblnet tests            # ruff is installed in .venv; there is no project config for it
```

## Architecture

Two independent transports, plus a facade over both:

- **`BLNETWeb`** (`blnet_web.py`) scrapes the device's German-language HTTP interface with `requests` and `htmldom`. Login POSTs to `/main.html` and stores the `TAID` cookie. `logged_in()` works by checking whether a restricted page sends back `Set-Cookie`. Use it as a context manager: `__enter__`/`__exit__` call `log_in`/`log_out`. Logging out matters because the device blocks all web access for about 150s after a session that wasn't logged out. The constructor calls `blnet_test()` and raises `ValueError` if no BL-NET answers. Switch values use the device's German strings `"EIN"`, `"AUS"`, `"AUTO"`. Only this transport can set switches, read manual/auto mode, and select a CAN node.
- **`BLNETDirect`** (`blnet_conn.py`) speaks TA's binary bootloader protocol over TCP (port 40000). It is a Python port of berwinter's PHP `blnet-connection.inc.php`. On construction it queries the device mode: `CAN_MODE`, `DL_MODE` or `DL2_MODE`. That mode decides frame counts, dataset sizes and how `_split_latest`/`_split_datasets` slice the response. The upstream README calls this transport buggy and untested. Note that it sets a process-wide `socket.setdefaulttimeout`.
- **`BLNETParser`** (`blnet_parser.py`) decodes one binary dataset (55 bytes, or 61 with a timestamp) into `{'analog','digital','speed','energy','power'}` dicts keyed by 1-based ID. It is used only by `BLNETDirect`.
- **`BLNET`** (`blnet.py`) is the high-level facade, with each transport switched on by `use_web` (default on) and `use_ta` (default off). `fetch()` returns a dict with all five domains. The web transport supplies analog/digital entries as dicts (`id`, `name`, `value`, `mode`). Direct data then overwrites `value` on those entries. If web is disabled, `fetch()` creates `{"value": ...}` entries from direct data alone; this "DL-only" path is what the current branch is fixing. `speed`/`energy`/`power` come only from direct. `get_value()` matches on `name` or `id` and assumes web-style entries.

## Device documentation

- Official BL-NET manual (German, v2.19): https://www.ta.co.at/download/datei/416. Its "Zugriff per Browser" chapter describes the web interface that `BLNETWeb` scrapes. Facts from it that the code relies on:
  - Only one user can be logged in at a time.
  - A session ends after 2.5 minutes without traffic. If a session isn't logged out, nobody can log in for those 2.5 minutes.
  - There are three access levels: Experte, Anwender and "nur ansehen". The login form field `blu=1` selects Experte.
  - From the factory no password is set. Code passes `password=None` to skip login.
  - The default ports are HTTP 80 and TA port 40000.
  - For CAN data logging, one UVR1611 must be node 1.
- The binary protocol used by `BLNETDirect` isn't in that manual. TA's spec "Schnittstelle Winsol <> Bootloader" (2006) describes it. The haus-terra link in the README is dead (404); an archived copy is at http://web.archive.org/web/20160122124754/http://www.haus-terra.at:80/heizung/download/Schnittstelle/Schnittstelle_PC_Bootloader.pdf. The spec covers:
  - The DL modes only: 1DL `0xA8` and 2DL `0xD1`.
  - The commands `0x81` (mode), `0x82` (firmware), `0xAA` (header), `0xAB` (latest data), `0xAC` (read logged block), `0xAD` (end read) and `0xAF` (reset logged data).
  - The dataset layouts.
- Not in the spec: CAN mode (`0xDC`), the `0xBA` "wait" response, and the frame-number byte sent after `0xAB`. These come from berwinter's reverse-engineered PHP code. In DL mode the spec's `0xAB` request is one byte, but the code sends two, and a real device accepts that.

## Tests

- `tests/test_parser.py` checks `BLNETParser` against raw byte fixtures.
- `tests/test_web.py` runs `BLNETWeb`/`BLNET` against a local mock HTTP server (`tests/test_structure/blnet_mock_server.py`, started in a thread by `server_control.py`). The mock serves the saved device pages in `tests/test_structure/*.htm` and emulates login cookies, password (`0123`), node switching and a "blocked" state. Expected parse results live in `tests/web_raw/web_state.py`.
- `tests/test_direct.py` runs `BLNET` in DL-only mode against a fake TCP BL-NET (`tests/test_structure/blnet_direct_mock_server.py`). The fake replays mode, header and latest-data responses recorded from a real device in DL mode. Only `GET_MODE`, `GET_HEADER`, `GET_LATEST` and `END_READ` are emulated; CAN/DL2 modes and reading stored data (`_get_data`) are untested.

# Python and JavaScript Guide (Automation Scripting)

Scope: Python for automation (data structures, JSON, `requests`, error handling and retries, `boto3`, typing/dataclasses, files/env/YAML, `pytest`, custom Ansible module and filter), plus the JavaScript you actually need inside ServiceNow, and Bash basics.

How to use: type the examples, run them, break them on purpose. Python 3.10+ assumed. Use a virtual environment for everything:

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install requests boto3 pyyaml pytest
```

---

## 0. What "good enough" means for your role

You are not writing web apps or algorithms. You write **glue**: call APIs, reshape JSON, talk to AWS, handle failures, and test it. If you can do the following without looking things up, you are strong enough:

1. Call an authenticated REST API, handle non-2xx, timeouts and retries.
2. Navigate nested JSON safely.
3. Query/modify AWS resources with `boto3`.
4. Structure a script (functions, typing, logging, exit codes) so someone else can maintain it.
5. Write a `pytest` test with mocking.
6. Write a small custom Ansible module.

---

## 1. Core data structures

```python
# list / dict / set / tuple
servers = ["web01", "web02", "db01"]
env_map = {"web01": "dev", "db01": "prod"}
unique_envs = {"dev", "prod", "dev"}            # set → {'dev', 'prod'}
point = (10, 20)                                  # tuple, immutable

# slicing
servers[0]; servers[-1]; servers[:2]; servers[::2]; servers[::-1]

# unpacking
first, *rest = servers                            # first='web01', rest=[...]
name, env = ("web01", "dev")
for i, s in enumerate(servers, start=1): ...
for k, v in env_map.items(): ...
for a, b in zip(servers, ["10.0.0.1", "10.0.0.2", "10.0.0.3"]): ...

# comprehensions
web = [s for s in servers if s.startswith("web")]
upper = {s: s.upper() for s in servers}
envs = {env_map[s] for s in env_map}

# dict operations
env_map.get("web99", "unknown")                   # no KeyError
env_map.setdefault("web03", "dev")
merged = {**env_map, "web04": "test"}             # or env_map | {...} on 3.9+
env_map.pop("web01", None)

# sorting
sorted(servers, key=len, reverse=True)
sorted(records, key=lambda r: (r["env"], r["name"]))

# any / all / sum / min / max
any(s.startswith("db") for s in servers)
```

### `collections` you will use constantly

```python
from collections import Counter, defaultdict

Counter(["dev", "prod", "dev"])                   # Counter({'dev': 2, 'prod': 1})
Counter(log_levels).most_common(3)

by_env = defaultdict(list)
for rec in records:
    by_env[rec["env"]].append(rec["name"])
```

### Interview classics (log parsing and counting)

```python
import re
from collections import Counter

line_re = re.compile(r"(?P<ip>\d+\.\d+\.\d+\.\d+) .* \"(?P<method>GET|POST) (?P<path>\S+)\" (?P<status>\d{3})")

def top_failed_ips(path: str, n: int = 5) -> list[tuple[str, int]]:
    failures: Counter[str] = Counter()
    with open(path) as f:                          # iterate lazily, never f.read() on big logs
        for line in f:
            m = line_re.search(line)
            if m and m["status"].startswith("5"):
                failures[m["ip"]] += 1
    return failures.most_common(n)
```

String basics: `str.split/join/strip/replace/startswith`, f-strings (`f"{name:>10}"`), `str.format`, `re.search/findall/sub`.

---

## 2. JSON handling

```python
import json

raw = '{"result": {"number": "INC001", "assigned_to": {"link": "...", "value": "abc"}, "tags": []}}'
data = json.loads(raw)                             # str → dict
text = json.dumps(data, indent=2, sort_keys=True)  # dict → str

# file I/O
with open("out.json", "w") as f:
    json.dump(data, f, indent=2)
with open("out.json") as f:
    data = json.load(f)
```

### Safe navigation of nested API responses

```python
number = data.get("result", {}).get("number", "UNKNOWN")
assignee = (data.get("result", {}).get("assigned_to") or {}).get("value")   # handles None AND missing

def dig(obj, *keys, default=None):
    """dig(data, 'result', 'assigned_to', 'value')"""
    for k in keys:
        try:
            obj = obj[k]
        except (KeyError, IndexError, TypeError):
            return default
    return obj
```

### Common reshaping

```python
# list of dicts → dict keyed by field
instances = [{"id": "i-1", "name": "web01"}, {"id": "i-2", "name": "web02"}]
by_id = {i["id"]: i for i in instances}

# AWS-style tags [{'Key':..., 'Value':...}] → dict
tags = {t["Key"]: t["Value"] for t in ec2_tags}

# flatten nested lists
flat = [i for sub in nested for i in sub]

# pretty print for debugging
print(json.dumps(resp.json(), indent=2))
```

Gotchas: JSON has no tuples/sets/datetimes. Convert with `default=str` when dumping (`json.dumps(obj, default=str)`). JSON `null` is `None`; `true/false` are `True/False`. ServiceNow returns many values as **strings** (`"2"`, `"true"`), so cast.

---

## 3. HTTP calls with `requests`

```python
import requests

BASE = "https://devXXXXX.service-now.com"
session = requests.Session()                       # reuse connections, shared headers
session.headers.update({"Accept": "application/json"})
session.auth = ("user", "pass")                    # dev only

# GET with params
r = session.get(f"{BASE}/api/now/table/incident",
                params={"sysparm_query": "active=true", "sysparm_limit": 5},
                timeout=(3.05, 30))                # (connect, read) → ALWAYS set a timeout
r.raise_for_status()                               # raises HTTPError on 4xx/5xx
items = r.json()["result"]

# POST JSON
r = session.post(f"{BASE}/api/now/table/incident",
                 json={"short_description": "test"}, timeout=30)

# PATCH
r = session.patch(f"{BASE}/api/now/table/incident/{sys_id}",
                  json={"state": "2"}, timeout=30)

# Bearer token
headers = {"Authorization": f"Bearer {token}"}

# Useful attributes
r.status_code; r.ok; r.headers["Content-Type"]; r.text; r.json(); r.elapsed
```

Rules:
- `requests` has **no default timeout**; always pass `timeout=`.
- `json=` sets `Content-Type: application/json` and serialises for you; `data=` sends form data/raw.
- `verify=True` (default). Never set `verify=False` in production; use a CA bundle (`verify="/path/ca.pem"`) for internal CAs.
- Use `Session` for connection reuse, shared headers, shared retry adapter.
- Never print tokens/passwords. Pull from env variables.

### Pagination (ServiceNow style)

```python
def fetch_all(session, url, params, page_size=100):
    params = {**params, "sysparm_limit": page_size, "sysparm_offset": 0}
    while True:
        r = session.get(url, params=params, timeout=30)
        r.raise_for_status()
        batch = r.json()["result"]
        yield from batch
        if len(batch) < page_size:
            break
        params["sysparm_offset"] += page_size
```

---

## 4. Error handling and retries

### try / except / else / finally

```python
try:
    r = session.get(url, timeout=10)
    r.raise_for_status()
except requests.exceptions.Timeout:
    log.error("Timed out calling %s", url)
    raise
except requests.exceptions.HTTPError as e:
    status = e.response.status_code
    if status == 404:
        return None
    raise
except requests.exceptions.RequestException as e:      # base class: connection errors etc.
    raise IntegrationError(f"Network problem: {e}") from e
else:
    return r.json()                                    # runs only if no exception
finally:
    log.debug("call finished")                         # always runs
```

Principles:
- **Catch specific exceptions**; never a bare `except:`. `except Exception` only at the top level (and log with stack trace).
- Use `raise ... from e` to keep the original cause.
- Use `log.exception("msg")` inside `except` to include the traceback.
- Fail loudly with a **non-zero exit code** so CI/AAP sees failure.

### Custom exceptions

```python
class IntegrationError(Exception):
    """Base for integration problems."""

class AuthError(IntegrationError): ...
class RateLimited(IntegrationError):
    def __init__(self, retry_after: float | None = None):
        super().__init__("Rate limited")
        self.retry_after = retry_after
```

### Retries with exponential backoff (hand-rolled)

```python
import random
import time
import logging
import requests

log = logging.getLogger(__name__)
RETRY_STATUS = {429, 500, 502, 503, 504}


def request_with_retry(session, method, url, *, retries=5, backoff=1.0, **kwargs):
    kwargs.setdefault("timeout", 30)
    for attempt in range(1, retries + 1):
        try:
            r = session.request(method, url, **kwargs)
            if r.status_code not in RETRY_STATUS:
                return r                                      # success or non-retryable (let caller decide)
            wait = float(r.headers.get("Retry-After", backoff * 2 ** (attempt - 1)))
            log.warning("HTTP %s on %s (attempt %d/%d)", r.status_code, url, attempt, retries)
        except (requests.ConnectionError, requests.Timeout) as e:
            wait = backoff * 2 ** (attempt - 1)
            log.warning("%s on %s (attempt %d/%d)", type(e).__name__, url, attempt, retries)
        if attempt == retries:
            break
        time.sleep(wait + random.uniform(0, 0.5))             # jitter avoids thundering herd
    raise IntegrationError(f"{method} {url} failed after {retries} attempts")
```

### Retries via `urllib3` (built into requests)

```python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry = Retry(total=5, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504],
              allowed_methods=["GET", "PUT", "DELETE", "PATCH"],     # POST excluded: not idempotent!
              respect_retry_after_header=True)
session.mount("https://", HTTPAdapter(max_retries=retry))
```

Key interview point: **only retry idempotent operations** (GET/PUT/DELETE, or POST with an idempotency key). Retrying a POST can create duplicate incidents.

### Decorator form (reusable)

```python
import functools

def retry(times=3, delay=1.0, exceptions=(Exception,)):
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*a, **kw):
            for i in range(1, times + 1):
                try:
                    return fn(*a, **kw)
                except exceptions:
                    if i == times:
                        raise
                    time.sleep(delay * 2 ** (i - 1))
        return wrapper
    return deco
```

(Or use the `tenacity` library in real projects.)

### Logging (not `print`)

```python
import logging
logging.basicConfig(level=logging.INFO,
                    format="%(asctime)s %(levelname)s %(name)s: %(message)s")
log = logging.getLogger(__name__)
log.info("Created incident %s", number)            # lazy formatting, no f-string in log calls
```

---

## 5. The project task: create a ServiceNow incident with an OAuth bearer token

```python
#!/usr/bin/env python3
"""Create a ServiceNow incident using OAuth (password grant)."""
from __future__ import annotations

import logging
import os
import sys
import time
from dataclasses import dataclass

import requests

log = logging.getLogger("snow")


class SnowError(Exception):
    pass


@dataclass
class Token:
    access_token: str
    expires_at: float

    @property
    def expired(self) -> bool:
        return time.time() >= self.expires_at - 30        # refresh 30 s early


class SnowClient:
    def __init__(self, instance: str, client_id: str, client_secret: str,
                 username: str, password: str) -> None:
        self.instance = instance.rstrip("/")
        self._creds = dict(client_id=client_id, client_secret=client_secret,
                           username=username, password=password)
        self._token: Token | None = None
        self.session = requests.Session()
        self.session.headers["Accept"] = "application/json"

    def _authenticate(self) -> None:
        r = self.session.post(f"{self.instance}/oauth_token.do",
                              data={"grant_type": "password", **self._creds}, timeout=30)
        if r.status_code != 200:
            raise SnowError(f"OAuth failed: {r.status_code}")      # never log r.text with secrets
        body = r.json()
        self._token = Token(body["access_token"], time.time() + int(body["expires_in"]))

    def _headers(self) -> dict[str, str]:
        if self._token is None or self._token.expired:
            self._authenticate()
        return {"Authorization": f"Bearer {self._token.access_token}"}

    def create_incident(self, short_description: str, **fields: str) -> dict:
        r = self.session.post(f"{self.instance}/api/now/table/incident",
                              headers=self._headers(),
                              json={"short_description": short_description, **fields},
                              timeout=30)
        if r.status_code == 401:                              # token revoked: refresh once
            self._token = None
            r = self.session.post(f"{self.instance}/api/now/table/incident",
                                  headers=self._headers(),
                                  json={"short_description": short_description, **fields},
                                  timeout=30)
        if r.status_code != 201:
            raise SnowError(f"Create incident failed: {r.status_code} {r.text[:300]}")
        return r.json()["result"]


def main() -> int:
    logging.basicConfig(level=logging.INFO)
    try:
        client = SnowClient(os.environ["SN_INSTANCE"], os.environ["SN_CLIENT_ID"],
                            os.environ["SN_CLIENT_SECRET"], os.environ["SN_USER"],
                            os.environ["SN_PASS"])
        inc = client.create_incident("Deployment failed on web01", impact="2", urgency="2")
        log.info("Created %s (%s)", inc["number"], inc["sys_id"])
        return 0
    except KeyError as e:
        log.error("Missing environment variable: %s", e)
    except (SnowError, requests.RequestException) as e:
        log.error("%s", e)
    return 1


if __name__ == "__main__":
    sys.exit(main())
```

---

## 6. `boto3` (AWS)

### Sessions, clients, resources

```python
import boto3

session = boto3.Session(profile_name="dev", region_name="eu-west-1")   # or env/instance role
ec2 = session.client("ec2")                  # low-level, full API coverage, returns dicts
ec2r = session.resource("ec2")               # higher-level objects (legacy for some services; client is preferred)
```

Credentials order: explicit params → env vars → shared config/credentials file → SSO → container/instance role. **Prefer roles/SSO/OIDC; never hardcode keys.**

### List, filter by tag, start/stop (with paginator)

```python
def list_instances(ec2, env: str) -> list[dict]:
    paginator = ec2.get_paginator("describe_instances")
    pages = paginator.paginate(Filters=[
        {"Name": "tag:Env", "Values": [env]},
        {"Name": "instance-state-name", "Values": ["running", "stopped"]},
    ])
    out = []
    for page in pages:
        for res in page["Reservations"]:
            for inst in res["Instances"]:
                tags = {t["Key"]: t["Value"] for t in inst.get("Tags", [])}
                out.append({
                    "id": inst["InstanceId"],
                    "name": tags.get("Name", ""),
                    "state": inst["State"]["Name"],
                    "private_ip": inst.get("PrivateIpAddress"),
                    "type": inst["InstanceType"],
                })
    return out


def stop(ec2, ids: list[str]) -> None:
    ec2.stop_instances(InstanceIds=ids)
    ec2.get_waiter("instance_stopped").wait(InstanceIds=ids)       # blocks until done


def start(ec2, ids: list[str]) -> None:
    ec2.start_instances(InstanceIds=ids)
    ec2.get_waiter("instance_running").wait(InstanceIds=ids)
```

### Error handling

```python
from botocore.exceptions import ClientError, BotoCoreError

try:
    ec2.stop_instances(InstanceIds=["i-0123"])
except ClientError as e:
    code = e.response["Error"]["Code"]                  # e.g. 'InvalidInstanceID.NotFound', 'UnauthorizedOperation'
    msg = e.response["Error"]["Message"]
    if code == "InvalidInstanceID.NotFound":
        log.warning("Already gone")
    else:
        raise
except BotoCoreError as e:                              # network/config errors
    raise
```

Test safely: `DryRun=True` raises `ClientError` with code `DryRunOperation` if you *would* have been allowed.

### Built-in retry config

```python
from botocore.config import Config
cfg = Config(retries={"max_attempts": 10, "mode": "adaptive"}, connect_timeout=5, read_timeout=30)
ec2 = session.client("ec2", config=cfg)
```

### Project task: reconcile AWS vs ServiceNow CMDB and report drift

```python
#!/usr/bin/env python3
"""Compare running EC2 instances with ServiceNow CIs; report drift."""
from __future__ import annotations
import json
import sys
from dataclasses import dataclass, asdict


@dataclass(frozen=True)
class DriftReport:
    missing_in_cmdb: list[str]       # in AWS, not in CMDB
    stale_in_cmdb: list[str]         # in CMDB, not in AWS
    attribute_mismatch: list[dict]   # in both, but data differs

    @property
    def has_drift(self) -> bool:
        return bool(self.missing_in_cmdb or self.stale_in_cmdb or self.attribute_mismatch)


def reconcile(aws: list[dict], cmdb: list[dict]) -> DriftReport:
    """aws items: {id, name, private_ip}; cmdb items: {serial_number, name, ip_address}"""
    aws_by_id = {i["id"]: i for i in aws}
    cmdb_by_id = {c["serial_number"]: c for c in cmdb}

    missing = sorted(aws_by_id.keys() - cmdb_by_id.keys())
    stale = sorted(cmdb_by_id.keys() - aws_by_id.keys())

    mismatches = []
    for iid in aws_by_id.keys() & cmdb_by_id.keys():
        a, c = aws_by_id[iid], cmdb_by_id[iid]
        diffs = {}
        if a["private_ip"] != c["ip_address"]:
            diffs["ip"] = {"aws": a["private_ip"], "cmdb": c["ip_address"]}
        if a["name"] != c["name"]:
            diffs["name"] = {"aws": a["name"], "cmdb": c["name"]}
        if diffs:
            mismatches.append({"id": iid, **diffs})

    return DriftReport(missing, stale, mismatches)


if __name__ == "__main__":
    # wiring: aws = list_instances(ec2, "prod"); cmdb = fetch_all(...) from section 3
    report = reconcile(aws=[], cmdb=[])
    print(json.dumps(asdict(report), indent=2))
    sys.exit(1 if report.has_drift else 0)          # non-zero exit = CI/AAP can alert
```

Design notes worth saying in interviews: pure `reconcile()` function (no I/O) so it is trivially testable; set operations (`-`, `&`) give O(n); exit code signals drift; the IRE-aware write-back is a separate step.

---

## 7. Functions, typing, dataclasses

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Optional, Iterable, Literal, TypedDict, Protocol

def build_name(prefix: str, *parts: str, sep: str = "-", **tags: str) -> str:
    """*parts → tuple of extras; **tags → dict of extras."""
    return sep.join([prefix, *parts])

def first_or_none(items: Iterable[str]) -> str | None:       # 3.10+ union syntax
    return next(iter(items), None)

Env = Literal["dev", "test", "prod"]

class IncidentPayload(TypedDict, total=False):
    short_description: str
    impact: str
    urgency: str

@dataclass(frozen=True)                                       # immutable, hashable
class ServerRequest:
    server_name: str
    env: Env
    requested_by: str
    tags: dict[str, str] = field(default_factory=dict)        # never use a mutable default directly

    def __post_init__(self) -> None:
        if not self.server_name.islower():
            raise ValueError("server_name must be lowercase")
```

Useful modern features: f-strings, `pathlib`, `enum.Enum`, context managers (`with`), generators (`yield`), `functools.lru_cache`, walrus (`:=`). Run `mypy`/`ruff` in CI.

### Context manager example

```python
from contextlib import contextmanager

@contextmanager
def snow_session(base: str, token: str):
    s = requests.Session()
    s.headers["Authorization"] = f"Bearer {token}"
    try:
        yield s
    finally:
        s.close()
```

---

## 8. Files, environment, YAML

```python
import os
from pathlib import Path
import yaml                                         # pip install pyyaml

# environment
instance = os.environ["SN_INSTANCE"]                # KeyError if missing (good: fail fast)
region = os.environ.get("AWS_REGION", "eu-west-1")  # with default

# pathlib
root = Path(__file__).resolve().parent
cfg_path = root / "config" / "settings.yml"
cfg_path.parent.mkdir(parents=True, exist_ok=True)
text = cfg_path.read_text(encoding="utf-8")
for p in root.glob("**/*.yml"): ...

# YAML (ALWAYS safe_load)
with cfg_path.open() as f:
    cfg = yaml.safe_load(f)                         # never yaml.load() on untrusted input

with open("out.yml", "w") as f:
    yaml.safe_dump(cfg, f, sort_keys=False)

# CSV
import csv
with open("servers.csv", newline="") as f:
    for row in csv.DictReader(f):
        print(row["name"])

# subprocess (call ansible-playbook etc.)
import subprocess
res = subprocess.run(["ansible-playbook", "site.yml", "--check"],
                     capture_output=True, text=True, check=False, timeout=600)
if res.returncode != 0:
    log.error(res.stderr)
```

Never use `shell=True` with user input (injection). Pass the command as a **list**.

CLI arguments with `argparse`:

```python
import argparse
p = argparse.ArgumentParser(description="Reconcile AWS and CMDB")
p.add_argument("--env", choices=["dev", "test", "prod"], required=True)
p.add_argument("--dry-run", action="store_true")
args = p.parse_args()
```

---

## 9. Testing with `pytest`

Layout:

```
project/
├── src/recon.py
├── tests/
│   ├── conftest.py
│   └── test_recon.py
└── pytest.ini        # [pytest] \n pythonpath = src
```

### Basic tests (pure function)

```python
# tests/test_recon.py
import pytest
from recon import reconcile


def test_detects_missing_and_stale():
    aws = [{"id": "i-1", "name": "web01", "private_ip": "10.0.0.1"}]
    cmdb = [{"serial_number": "i-2", "name": "old", "ip_address": "10.0.0.9"}]
    report = reconcile(aws, cmdb)
    assert report.missing_in_cmdb == ["i-1"]
    assert report.stale_in_cmdb == ["i-2"]
    assert report.has_drift


def test_no_drift():
    item_aws = {"id": "i-1", "name": "web01", "private_ip": "10.0.0.1"}
    item_cmdb = {"serial_number": "i-1", "name": "web01", "ip_address": "10.0.0.1"}
    assert not reconcile([item_aws], [item_cmdb]).has_drift


@pytest.mark.parametrize("aws_ip,cmdb_ip,expected", [
    ("10.0.0.1", "10.0.0.1", False),
    ("10.0.0.1", "10.0.0.2", True),
])
def test_ip_mismatch(aws_ip, cmdb_ip, expected):
    a = [{"id": "i-1", "name": "n", "private_ip": aws_ip}]
    c = [{"serial_number": "i-1", "name": "n", "ip_address": cmdb_ip}]
    assert reconcile(a, c).has_drift is expected
```

### Fixtures and mocking HTTP

```python
# tests/conftest.py
import pytest

@pytest.fixture
def snow_env(monkeypatch):
    monkeypatch.setenv("SN_INSTANCE", "https://example.service-now.com")
    monkeypatch.setenv("SN_CLIENT_ID", "id")
    monkeypatch.setenv("SN_CLIENT_SECRET", "secret")
    monkeypatch.setenv("SN_USER", "u")
    monkeypatch.setenv("SN_PASS", "p")
```

```python
# tests/test_snow_client.py  (pip install responses)
import responses
import pytest
from snow import SnowClient, SnowError

BASE = "https://example.service-now.com"


@responses.activate
def test_create_incident_ok():
    responses.post(f"{BASE}/oauth_token.do",
                   json={"access_token": "tok", "expires_in": 1800}, status=200)
    responses.post(f"{BASE}/api/now/table/incident",
                   json={"result": {"number": "INC1", "sys_id": "abc"}}, status=201)

    c = SnowClient(BASE, "id", "secret", "u", "p")
    inc = c.create_incident("hello")
    assert inc["number"] == "INC1"
    assert responses.calls[1].request.headers["Authorization"] == "Bearer tok"


@responses.activate
def test_create_incident_failure():
    responses.post(f"{BASE}/oauth_token.do",
                   json={"access_token": "tok", "expires_in": 1800}, status=200)
    responses.post(f"{BASE}/api/now/table/incident", json={"error": {}}, status=403)

    with pytest.raises(SnowError):
        SnowClient(BASE, "id", "secret", "u", "p").create_incident("x")
```

### Mocking boto3

```python
# pip install moto
from moto import mock_aws
import boto3

@mock_aws
def test_list_instances():
    ec2 = boto3.client("ec2", region_name="eu-west-1")
    ec2.run_instances(ImageId="ami-12345678", MinCount=1, MaxCount=1,
                      TagSpecifications=[{"ResourceType": "instance",
                                          "Tags": [{"Key": "Env", "Value": "dev"},
                                                   {"Key": "Name", "Value": "web01"}]}])
    from aws_utils import list_instances
    result = list_instances(ec2, "dev")
    assert result[0]["name"] == "web01"
```

Or use `unittest.mock.patch` / `monkeypatch` for simple cases.

Commands:

```bash
pytest -q                       # run all
pytest -k reconcile -x -vv      # filter by name, stop on first fail, verbose
pytest --cov=src --cov-report=term-missing    # needs pytest-cov
```

CI step (GitHub Actions): `pip install -r requirements.txt && pytest`.

---

## 10. Custom Ansible module and filter (Python)

### Module

```python
#!/usr/bin/python
# plugins/modules/snow_ci_lookup.py   (or library/ next to the playbook)
from __future__ import annotations
from ansible.module_utils.basic import AnsibleModule
from ansible.module_utils.urls import open_url
import json

DOCUMENTATION = r"""
---
module: snow_ci_lookup
short_description: Look up a CI by name in ServiceNow
options:
  instance: {description: Base URL, required: true, type: str}
  token: {description: Bearer token, required: true, type: str, no_log: true}
  name: {description: CI name, required: true, type: str}
"""

EXAMPLES = r"""
- snow_ci_lookup:
    instance: https://dev1.service-now.com
    token: "{{ sn_token }}"
    name: web01
"""

RETURN = r"""
ci:
  description: Matching record or empty dict
  returned: always
  type: dict
"""


def run_module():
    module = AnsibleModule(
        argument_spec=dict(
            instance=dict(type="str", required=True),
            token=dict(type="str", required=True, no_log=True),
            name=dict(type="str", required=True),
        ),
        supports_check_mode=True,                 # read-only, safe in check mode
    )
    p = module.params
    url = f"{p['instance']}/api/now/table/cmdb_ci_server?sysparm_query=name={p['name']}&sysparm_limit=1"
    try:
        resp = open_url(url, headers={"Authorization": f"Bearer {p['token']}",
                                      "Accept": "application/json"}, timeout=30)
        result = json.loads(resp.read()).get("result", [])
    except Exception as e:                        # modules must never traceback: use fail_json
        module.fail_json(msg=f"ServiceNow lookup failed: {e}")

    module.exit_json(changed=False, ci=result[0] if result else {})


def main():
    run_module()


if __name__ == "__main__":
    main()
```

Rules: always `exit_json`/`fail_json`; return `changed` honestly (read-only = `False`); `no_log=True` on secrets; support check mode; compare current vs desired before changing (idempotency); document with `DOCUMENTATION/EXAMPLES/RETURN`; test with `ansible-test sanity` or `pytest` using a mocked `AnsibleModule`.

Run directly for quick tests:

```bash
echo '{"ANSIBLE_MODULE_ARGS": {"instance":"https://x","token":"t","name":"web01"}}' | python snow_ci_lookup.py
```

### Filter plugin

```python
# filter_plugins/aws_filters.py
class FilterModule:
    def filters(self):
        return {"aws_tags_to_dict": self.aws_tags_to_dict}

    @staticmethod
    def aws_tags_to_dict(tags):
        return {t["Key"]: t["Value"] for t in (tags or [])}
```

```yaml
- ansible.builtin.debug:
    msg: "{{ instance.tags_list | aws_tags_to_dict }}"
```

---

## 11. JavaScript for ServiceNow (what you actually need)

ServiceNow server-side scripting is **ES5 by default** (Rhino); ES2021+ is available when you enable it per script (scoped apps on newer releases). Write defensively for ES5.

### Language essentials

```javascript
// types and coercion gotchas
var s = gr.getValue('state');           // string "2"
if (s == 2) {}                          // loose: works. Prefer: parseInt(s, 10) === 2
var el = gr.short_description;          // GlideElement object, NOT a string
var str = gr.short_description + '';    // or .toString() / getValue()

// objects, arrays
var obj = { name: 'web01', env: 'dev' };
obj.name; obj['env'];
var arr = ['a', 'b', 'c'];
arr.push('d'); arr.indexOf('b'); arr.join(',');
arr.forEach(function (x) { gs.info(x); });
var upper = arr.map(function (x) { return x.toUpperCase(); });
var big = arr.filter(function (x) { return x !== 'a'; });

// JSON
var o = JSON.parse('{"a":1}');
var j = JSON.stringify(o);

// safe checks
if (!gs.nil(gr.assigned_to)) {}         // true if empty
if (gr.assigned_to.nil()) {}
```

### Frequently used ServiceNow APIs

| Context | API | Example |
|---|---|---|
| Server | `gs` | `gs.info()`, `gs.error()`, `gs.getProperty()`, `gs.getUserID()`, `gs.nowDateTime()`, `gs.eventQueue()` |
| Server | `GlideRecord` / `GlideAggregate` | See ServiceNow guide section 7 |
| Server | `sn_ws.RESTMessageV2` | Outbound REST |
| Server | `JSON`, `GlideDateTime`, `GlideDuration` | Parsing and dates |
| Client | `g_form` | `getValue`, `setValue`, `setMandatory`, `setVisible`, `showFieldMsg` |
| Client | `g_user` | `userID`, `hasRole('itil')` |
| Client | `GlideAjax` | Call a client-callable Script Include |

### Typical patterns

```javascript
// 1. Safe parse of an outbound response
function parseBody(resp) {
    try {
        return JSON.parse(resp.getBody());
    } catch (e) {
        gs.error('Bad JSON from API: ' + e.message + ' body=' + resp.getBody().substring(0, 200));
        return null;
    }
}

// 2. Retry loop (server side, small number of attempts only)
function callWithRetry(msg, attempts) {
    for (var i = 1; i <= attempts; i++) {
        var resp = msg.execute();
        var code = resp.getStatusCode();
        if (code < 500 && code !== 429) return resp;
        gs.warn('Retry ' + i + ' after HTTP ' + code);
        gs.sleep(1000 * i);                     // blocks a thread: keep short, prefer async flows
    }
    return null;
}

// 3. Dedupe with an object as a set
var seen = {};
var unique = [];
while (gr.next()) {
    var key = gr.getValue('name');
    if (!seen[key]) { seen[key] = true; unique.push(key); }
}

// 4. Build an encoded query safely
var q = 'active=true^priority=1^assignment_group=' + groupSysId;
```

### Python ↔ JavaScript cheat table

| Task | Python | ServiceNow JS (ES5) |
|---|---|---|
| Safe get | `d.get("k", default)` | `obj.k !== undefined ? obj.k : default` |
| Parse JSON | `json.loads(s)` | `JSON.parse(s)` |
| Loop with index | `for i, x in enumerate(xs)` | `xs.forEach(function (x, i) {})` |
| Comprehension | `[f(x) for x in xs if c]` | `xs.filter(c).map(f)` |
| Set | `set(xs)` | object keys `seen[x] = true` |
| String format | `f"{a}-{b}"` | `a + '-' + b` |
| Exceptions | `try/except` | `try/catch` |
| Query a table | `requests.get(.../table/incident)` | `new GlideRecord('incident')` |

---

## 12. Bash minimum

```bash
#!/usr/bin/env bash
set -euo pipefail          # -e exit on error, -u unset var is error, pipefail: a pipe fails if any part fails
IFS=$'\n\t'

# variables and quoting (ALWAYS quote)
name="web 01"
echo "Server: $name"
ls "$name"

# exit codes
if grep -q "ERROR" app.log; then echo "errors found"; fi
cmd || { echo "cmd failed" >&2; exit 1; }
echo $?                                        # last exit code

# pipes: grep / awk / sed / cut / sort / uniq / wc / xargs / jq
grep -E "ERROR|WARN" app.log | awk '{print $1, $5}' | sort | uniq -c | sort -rn | head
sed -i.bak 's/old/new/g' config.txt            # in-place edit (macOS: use -i '')
awk -F, '$3 > 100 {print $1}' data.csv
curl -s "$URL" | jq -r '.result[] | [.number, .state] | @tsv'

# loops, functions, trap
for host in $(cat hosts.txt); do ssh "$host" uptime; done
while read -r line; do echo "$line"; done < hosts.txt

cleanup() { rm -f "$TMP"; }
trap cleanup EXIT
TMP=$(mktemp)

# arguments
usage() { echo "usage: $0 <env>" >&2; exit 2; }
[[ $# -eq 1 ]] || usage
env="$1"
```

Rules: quote variables, use `[[ ]]`, never parse `ls`, use `mktemp`, check dependencies with `command -v jq >/dev/null`, run `shellcheck` on scripts.

---

## 13. Project structure for a script that others maintain

```
snow-aws-recon/
├── pyproject.toml          # or requirements.txt
├── src/
│   └── recon/
│       ├── __init__.py
│       ├── aws.py          # boto3 calls only
│       ├── snow.py         # ServiceNow client only
│       ├── core.py         # pure reconcile logic
│       └── cli.py          # argparse + wiring + exit codes
├── tests/
├── .github/workflows/ci.yml
└── README.md
```

CI (GitHub Actions) minimal gate:

```yaml
name: ci
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python: ["3.10", "3.12"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "${{ matrix.python }}" }
      - run: pip install -r requirements.txt pytest ruff
      - run: ruff check .
      - run: pytest -q
```

Separation of I/O (aws.py, snow.py) from logic (core.py) is what makes the code testable and is the main design point to be able to explain.

---

## 14. Interview quick-fire

1. **How do you call an API robustly?** Session, timeout, `raise_for_status`, retry only idempotent calls with exponential backoff and jitter, honour `Retry-After`, log without secrets, non-zero exit on failure.
2. **`dict.get` vs `[]`?** `get` returns a default instead of raising `KeyError`; use `[]` when absence is a bug and should fail fast.
3. **List vs generator?** A list holds everything in memory; a generator yields lazily (log files, paginated APIs).
4. **How do you avoid hardcoding credentials?** Environment variables from the CI/AAP credential, cloud roles/OIDC, secret managers. Never in code or logs.
5. **How do you test code that calls AWS/ServiceNow?** Keep logic pure and test it directly; mock HTTP with `responses`, AWS with `moto`/`monkeypatch`; parametrize edge cases.
6. **Mutable default argument pitfall?** `def f(x=[])` shares one list across calls; use `None` or `field(default_factory=list)`.
7. **Why `yaml.safe_load`?** `yaml.load` can construct arbitrary Python objects (code execution risk).
8. **What must a custom Ansible module do?** Take an argument spec, support check mode, be idempotent, return `changed` accurately, use `exit_json`/`fail_json`, mask secrets with `no_log`.
9. **GlideRecord value comparison pitfall?** Fields are GlideElement objects; use `getValue()` or `.toString()` to compare, and know loose `==` vs strict `===`.
10. **`set -euo pipefail`?** Exit on error, treat unset variables as errors, fail pipelines if any stage fails.

---

## 15. Revision checklist

- [ ] Rewrite a nested-JSON parsing function using `.get()` and the `dig` helper; test with missing keys
- [ ] Call a public API with `requests` (timeout, `raise_for_status`, pagination)
- [ ] Add a retry/backoff wrapper and prove it with a mocked 503, 503, 200 sequence
- [ ] ServiceNow OAuth client + create-incident script (P4) with environment variables only
- [ ] `boto3`: list instances by tag with a paginator; stop/start with waiters; handle `ClientError`
- [ ] Reconcile script (P7): pure `reconcile()`, JSON report, exit code on drift
- [ ] Add type hints and a dataclass; run `mypy` or `ruff`
- [ ] `pytest`: 5+ tests incl. parametrize, `monkeypatch`, `responses`, and `moto`
- [ ] Custom Ansible module with check mode + a filter plugin; run it from a playbook
- [ ] Re-implement one Python helper in ES5 JavaScript inside a ServiceNow Script Include
- [ ] Write a Bash script with `set -euo pipefail`, a trap, and a `jq` pipeline; run `shellcheck`
- [ ] Put it all in a repo with a GitHub Actions workflow running `ruff` and `pytest`

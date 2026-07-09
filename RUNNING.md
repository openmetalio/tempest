# Tempest

Tempest is the OpenStack integration test suite used to validate OpenStack deployments for operational and trademark compliance.

## 1. Prerequisites

- Python 3.10+
- A live OpenStack deployment with OpenStack services:
  - Minimum: Keystone
  - Core: Keystone, Nova, Glance, Cinder, Neutron, Swift
- Admin credentials for the target cloud
- A Cirros image (or similar lightweight cloud image) uploaded and available in Glance
- At least two flavors available in Nova
- An accessible public or provider network for floating IPs

## 2. Tempest installation

Install Tempest into a virtual environment:

```bash
# Setup the virtual environment
python3 -m venv ~/.local/tempest
source ~/.local/tempest/bin/activate

# Install the latest stable version from PyPi
pip install tempest

# Install from this source tree instead
pip install -e .
```

Verify the install:

```bash
tempest --version
```

## 3. OpenStack Client setup

Before we can configure and run Tempest, we're going to need to collect some information from the target cloud.

Install the OpenStack client:

```sh
# From the same virtualenv as earlier
pip install python-openstackclient
```

Create `~/.config/openstack/clouds.yaml`:

```yaml
clouds:
  my-cloud:
    auth:
      auth_url: https://<keystone-host>:5000/v3
      username: admin
      password: <password>
      project_name: admin
      user_domain_name: Default
      project_domain_name: Default
    identity_api_version: 3
    region_name: <region>
```

Verify it works:

```bash
openstack --os-cloud my-cloud token issue
```

Retrieve the values needed for `tempest.conf`:

```bash
# Images — use a small cloud image such as Cirros
openstack --os-cloud my-cloud image list

# Flavors — pick two; the first should be small (1 vCPU, 512MB RAM or similar)
openstack --os-cloud my-cloud flavor list

# External network for floating IPs
openstack --os-cloud my-cloud network list --external
```

## 4. Tempest configuration

Now we can create the tempest workspace and setup the configuration:

```bash
mkdir -p ~/.config/tempest/defaults ~/.config/tempest/my-cloud
tempest init ~/.config/tempest/my-cloud --config-dir ~/.config/tempest/defaults
```

This creates the workspace directory with a skeleton `tempest.conf` and registers it
under `~/.tempest/workspace.yaml`. To list all registered workspaces:

```bash
tempest workspace list
```

Now we can modify the `tempest.conf` to reflect the configuration values we collected from the target cloud:

```ini
# ~/.config/tempest/my-cloud/etc/tempest.conf

[DEFAULT]
log_file = ~/.config/tempest/my-cloud/tempest_log
use_stderr = true

[oslo_concurrency]
lock_path = ~/.config/tempest/my-cloud/tempest_lock

[service_available]
cinder = true
glance = true
horizon = false
neutron = true
nova = true
swift = true

[validation]
run_validation = true
connect_method = floating
auth_method = keypair

[auth]
admin_username = admin
admin_password = <keystone_admin_password>
admin_project_name = admin
admin_domain_name = Default
use_dynamic_credentials = true

[compute]
endpoint_type = publicURL
flavor_ref = <ID>
flavor_ref_alt = <ID>
image_ref = <UUID>
image_ref_alt = <UUID>

[identity]
auth_version = v3
disable_ssl_certificate_validation = true
region = <region>  # must match the region_id in `openstack catalog list`, or
                    # Tempest silently falls back to a service's first listed
                    # endpoint, which may be internal-only
uri_v3 = https://<IPADDR>:5000/v3
v3_endpoint_type = publicURL

[image]
endpoint_type = publicURL

[network]
endpoint_type = publicURL
public_network_id = <UUID>

[object-storage]
endpoint_type = publicURL

[volume]
endpoint_type = publicURL
```

Then we can do a quick test to make sure that the Tempest config is valid:

```bash
TEMPEST_CONFIG=~/.config/tempest/my-cloud/etc/tempest.conf \
tempest verify-config --update
```

This probes the live API to reconcile extension and feature flags in `tempest.conf`.

## 5. Pre-Test State Snapshot

Before running any tests, capture the current state of the cloud so that the cleanup
tool can safely remove only what Tempest created. `tempest cleanup` uses `--config-file`,
not `--workspace`:

```bash
tempest cleanup \
  --config-file ~/.config/tempest/my-cloud/etc/tempest.conf \
  --init-saved-state
```

This writes `saved_state.json` into the current working directory.

## 6. Running Tests

**Smoke tests** (quick sanity check):

```bash
tempest run --workspace my-cloud --concurrency 2 --smoke
```

**Full test run:**

```bash
tempest run --workspace my-cloud --concurrency 2
```

Adjust `--concurrency` to the number of parallel workers. Each worker requires its own
set of credentials, so with dynamic credentials the limiting factor is usually cloud
capacity, not the credential count.

## 7. Interop trademark validation

The Interop WG publishes required test lists as JSON guidelines files in the interop
repository. Each guideline targets a specific OpenStack trademark (e.g., "OpenStack
Powered Platform"). The test IDs in those files map directly to Tempest test names.

**Step 1 — Get the current guideline file.**

Clone or browse the interop repo to find the active guideline for the trademark you are
pursuing. Guideline files are under `guidelines/` and look like `YYYY.NN.json`.

**Step 2 — Extract the required test list.**

The guideline JSON contains a `"tests"` object. Extract the test IDs into an include
list:

```bash
python3 -c "
import json, sys
with open('guidelines/YYYY.NN.json') as f:
    g = json.load(f)
tests = [t for t, v in g['tests'].items() if v.get('required')]
print('\n'.join(tests))
" > interop-required-tests.txt
```

**Step 3 — List what will run before committing.**

```bash
tempest run --workspace my-cloud --include-list interop-required-tests.txt --list-tests
```

**Step 4 — Run only the required tests.**

```bash
tempest run --workspace my-cloud --include-list interop-required-tests.txt --serial
```

`--serial` avoids any parallel ordering issues; omit it if you want speed and have
enough credential capacity.

## 8. Running from anywhere (no workspace)

If you prefer to run without a registered workspace, point directly at a config file:

```bash
tempest run --config-file ~/.config/tempest/my-cloud/etc/tempest.conf \
  --include-list interop-required-tests.txt
```

## 9. Reading Results

After a run, results are stored in the `.stestr/` directory inside the workspace.
Summarize the last run:

```bash
stestr last --subunit | subunit-stats
```

Or view a detailed, human-readable report:

```bash
stestr last
```

To save a subunit stream for later analysis or submission:

```bash
stestr last --subunit > results.subunit
```

The `subunit-describe-calls` tool produces a summary of each test's API calls:

```bash
tempest subunit-describe-calls --subunit-input results.subunit
```

## 10. Submitting Results for Trademark Approval

The submission process is managed by the OpenInfra Foundation. Contact them directly
to confirm the current procedure, as the interop documentation has not been consistently
updated. Starting points:

- Interop WG mailing list: `openstack-discuss@lists.openstack.org` with `[interop]` in
  the subject
- OpenInfra trademark inquiries: https://openinfra.dev/legal/trademark-policy

Before reaching out, have your results ready:

```bash
stestr last --subunit > results.subunit
```

## 11. Cleanup

After testing, remove all resources Tempest created. Run from the workspace directory
or pass `--config-file` directly:

```bash
tempest cleanup --config-file ~/.config/tempest/my-cloud/etc/tempest.conf
```

To see what would be deleted first (dry run):

```bash
tempest cleanup --config-file ~/.config/tempest/my-cloud/etc/tempest.conf --dry-run
```

Review `dry_run.json` before proceeding. To also delete the Tempest admin project and
users:

```bash
tempest cleanup \
  --config-file ~/.config/tempest/my-cloud/etc/tempest.conf \
  --delete-tempest-conf-objects
```

## 12. Pre-Provisioned Credentials (Alternative to Dynamic)

If admin credentials are unavailable or the cloud policy prevents dynamic user
creation, use pre-provisioned credentials instead.

Generate an accounts file from an environment where admin access is available:

```bash
tempest account-generator \
  --os-username admin \
  --os-password <password> \
  --os-project-name admin \
  --os-domain-name Default \
  --config-file ~/.config/tempest/my-cloud/etc/tempest.conf \
  --concurrency 4 \
  --tag tempest \
  accounts.yaml
```

Place the resulting `accounts.yaml` in the workspace `etc/` directory, then update
`tempest.conf`:

```ini
[auth]
use_dynamic_credentials = false
test_accounts_file = /full/path/to/etc/accounts.yaml
```

The concurrency value should be at least twice the number of parallel workers you plan
to use.

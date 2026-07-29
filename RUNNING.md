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
log_file = /home/<USER>/.config/tempest/my-cloud/logs/tempest.log
use_stderr = true

[oslo_concurrency]
lock_path = /home/<USER>/.config/tempest/my-cloud/tempest_lock

[service_available]
cinder = true
glance = true
horizon = false
neutron = true
nova = true
swift = true

[validation]
auth_method = keypair
connect_method = floating
run_validation = true

image_ssh_user = ubuntu
image_alt_ssh_user = centos

[auth]
admin_username = admin
admin_password = <keystone_admin_password>
admin_project_name = admin
admin_domain_name = Default
use_dynamic_credentials = true

[compute]
endpoint_type = publicURL
flavor_ref = gen2.micro
flavor_ref_alt = gen2.medium
image_ref = <UUID>
image_ref_alt = <UUID>

[identity]
disable_ssl_certificate_validation = true
region = <NAME>
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
volume_size = 4
```

Next we need to disable some tests that are not valid when using Ceph RGW instead of OpenStack Swift:

```conf
# ~/.config/tempest/my-cloud/exclude.txt

# Glance Image sharing is admin-only by our policy and these tests share images
# as a normal project user, which we intentionally forbid.
tempest.api.image.v2.test_images.ListSharedImagesTest.test_list_images_param_member_status
tempest.api.image.v2.test_images_member.ImagesMemberTest
tempest.api.image.v2.test_images_member_negative.ImagesMemberNegativeTest
# RGW returns 404 (not Swift's 401) for unauthenticated write/delete
tempest.api.object_storage.test_container_acl_negative.ObjectACLsNegativeTest.test_write_object_without_using_creds
tempest.api.object_storage.test_container_acl_negative.ObjectACLsNegativeTest.test_delete_object_without_using_creds
# RGW doesn't enforce Swift's arbitrary container-metadata count/length caps
tempest.api.object_storage.test_container_services_negative.ContainerNegativeTest.test_create_container_metadata_exceeds_overall_metadata_count
tempest.api.object_storage.test_container_services_negative.ContainerNegativeTest.test_create_container_metadata_name_exceeds_max_length
tempest.api.object_storage.test_container_services_negative.ContainerNegativeTest.test_create_container_metadata_value_exceeds_max_length
# RGW allows serving a static web page from RGW but we don't enable it because
# it requires knowing the domain(s) during configuration.
tempest.api.object_storage.test_container_staticweb.StaticWebTest.test_web_index
tempest.api.object_storage.test_container_staticweb.StaticWebTest.test_web_listing_css
```

Then we can do a quick test to make sure that the Tempest config is valid, and ensure that it will enable tests based on the extensions we have enabled:

```bash
TEMPEST_CONFIG=~/.config/tempest/my-cloud/etc/tempest.conf \
tempest verify-config --update --replace-ext -o ~/.config/tempest/my-cloud/etc/tempest.conf
```

This probes the live API to reconcile extension and feature flags in `tempest.conf`.

## 5. Pre-Test State Snapshot

Before running any tests, capture the current state of the cloud so that the cleanup
tool can safely remove only what Tempest created. `tempest cleanup` uses `--config-file`,
not `--workspace`:

```bash
tempest cleanup --config-file ~/.config/tempest/my-cloud/etc/tempest.conf \
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
tempest run --workspace my-cloud --concurrency 2 \
  --exclude-list ~/.config/tempest/my-cloud/exclude.txt
```

Adjust `--concurrency` to the number of parallel workers. Each worker requires its own
set of credentials, so with dynamic credentials the limiting factor is usually cloud
capacity, not the credential count.

## 7. Reading Results

After a run, results are stored in the `.stestr/` directory inside the workspace.

```sh
cd ~/.config/temptest/my-cloud
```

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

## 8. Submitting Results for Trademark Approval

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

## 9. Cleanup

After testing, remove all resources Tempest created. Run from the workspace directory
or pass `--config-file` directly:

```bash
tempest cleanup --config-file ~/.config/tempest/my-cloud/etc/tempest.conf
```

To see what would be deleted first (dry run):

```bash
tempest cleanup --dry-run --config-file ~/.config/tempest/my-cloud/etc/tempest.conf
```

Review `dry_run.json` before proceeding. To also delete the Tempest admin project and
users:

```bash
tempest cleanup --config-file ~/.config/tempest/my-cloud/etc/tempest.conf \
  --delete-tempest-conf-objects
```

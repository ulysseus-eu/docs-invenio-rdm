# Upgrading from v11 to v12.0.0b1

## Prerequisites

The steps listed in this article require an existing local installation of InvenioRDM v11, please make sure that this is given!

If unsure, run `invenio-cli install` from inside the instance directory before executing the listed steps.

!!! warning "Backup"

    Always backup your database and files before you try to perform an upgrade.

!!! info "Older versions"

    In case you have an InvenioRDM installation older than v10, you can gradually upgrade using the existing instructions to v10 and afterwards continue from here.
 
## Upgrade Steps

!!! warning "Upgrade your invenio-cli"

    Make sure you have the latest `invenio-cli`, for InvenioRDM v12.0.0b1 the release is v1.2.0

    ```bash
    $ invenio-cli --version
    invenio-cli, version 1.2.0
    ```

!!! info "Virtual environments"
    In case you are not inside a virtual environment, make sure that you prefix each `invenio` command with `pipenv run`.

**Local development**

Changing the Python version in your development environment highly
depends on your setup, and there is no golden rule.
One example would be to use [PyEnv](https://github.com/pyenv/pyenv).

You should delete your virtualenv before running `invenio-cli` or `pipenv` commands below.

!!! warning "Risk of losing data"

    Your virtual env folder contains uploaded files in InvenioRDM, in `var/instance/data`.
    If you need to keep such files, make sure you copy them over to the new virtual env in the same location.

### Upgrade InvenioRDM

Make sure that your virtual env is now running with Python 3.9.

Upgrade the RDM version:

```bash
cd <my-site>
# Upgrade to InvenioRDM v12
invenio-cli packages update 12.0.0b1
pipenv uninstall flask-babelex
delete "from flask_babelex import lazy_gettext as _" from invenio.cfg
# Re-build assets
invenio-cli assets build
```

Optionally, update the file `<my-site>/Pipfile`. Attention: this action might lead to installing unwanted pre-releases of other packages.  

```diff
[packages]
---invenio-app-rdm = {extras = [...], version = "~=11.0.0"}
+++invenio-app-rdm = {extras = [...], version = "==12.0.0b1"}

[pipenv]
allow_prereleases = true
```

### 1. Now clone invenio-app-rdm localy, then modify mogration file like below:

#### Update function update_parent
```python
def update_parent(record):
"""Update parent schema and parent communities for older records."""
if "pids" not in record.parent:
    record.parent["pids"] = {}

    pids = (
        current_rdm_records.records_service.pids.parent_pid_manager.create_all(
            record.parent, pids={}, schemes={"doi"}
        )
    )
    current_rdm_records.records_service.pids.parent_pid_manager.reserve_all(
        record.parent, pids
    )
    record.parent["pids"] = pids

new_parent_schema = "local://records/parent-v3.0.0.json"
record.parent["$schema"] = new_parent_schema
if (
    isinstance(record.parent["access"]["owned_by"], list)
    and len(record.parent["access"]["owned_by"]) > 0
):
    record.parent.access.owned_by = {
        "user": record.parent["access"]["owned_by"][0]["user"]
    }
```

#### Update function update_record
```python
def update_record(record):
# skipping deleted records because can't be committed
if record.is_deleted:
return

record["$schema"] = "local://records/record-v6.0.0.json"

try:
secho(f"Updating record : {record.pid.pid_value}", fg="yellow")

# Initialize media files as disabled if not any
record.setdefault("media_files", {"enabled": False})
if record.media_files.bucket is None:
record.media_files.create_bucket()

update_parent(record)

record.parent.commit()
record.commit()

secho(f"> Updated parent: {record.parent.pid.pid_value}", fg="green")
secho(f"> Updated record: {record.pid.pid_value}\n", fg="green")
return None
except Exception as e:
secho("> Error {}".format(repr(e)), fg="red")
error = "Record {} failed to update".format(record.pid.pid_value)
return error
```

#### Include your module in your instance
```bash
invenio-cli packages install ../invenio-app-rdm
```

### Correct bug from DOI
Inside invenio.cfg, modify:

```bash
DATACITE_ENABLED = True
DATACITE_USERNAME = ""
DATACITE_PASSWORD = ""
DATACITE_PREFIX = "datacite_prefix"
DATACITE_TEST_MODE = True
DATACITE_DATACENTER_SYMBOL = ""
```

### Database migration

Execute the database migration:

```bash
# Execute the database migration
invenio alembic upgrade
```


### Declare usage statistics processing queues

```shell
invenio queues declare
```

Queues dict_keys(['stats-file-download', 'stats-record-view']) have been declared.

### Update indices mappings

```shell
pipenv run invenio index destroy --yes-i-know
```

other terminal : 
```shell
celery -A invenio_app.celery worker --beat --events --loglevel=INFO
```

got :

```shell
[tasks]
. invenio_accounts.tasks.clean_session_table
. invenio_accounts.tasks.delete_ips
. invenio_accounts.tasks.send_security_email
. invenio_app_rdm.tasks.file_integrity_report
. invenio_communities.fixtures.tasks.create_demo_community
. invenio_communities.fixtures.tasks.reindex_featured_entries
. invenio_communities.tasks.clear_cache
. invenio_drafts_resources.services.records.tasks.cleanup_drafts
. invenio_files_rest.tasks.clear_orphaned_files
. invenio_files_rest.tasks.merge_multipartobject
. invenio_files_rest.tasks.migrate_file
. invenio_files_rest.tasks.remove_expired_multipartobjects
. invenio_files_rest.tasks.remove_file_data
. invenio_files_rest.tasks.schedule_checksum_verification
. invenio_files_rest.tasks.verify_checksum
. invenio_github.tasks.disconnect_github
. invenio_github.tasks.process_release
. invenio_github.tasks.refresh_accounts
. invenio_github.tasks.sync_account
. invenio_github.tasks.sync_hooks
. invenio_indexer.tasks.delete_record
. invenio_indexer.tasks.index_record
. invenio_indexer.tasks.process_bulk_queue
. invenio_mail.tasks._send_email_with_attachments
. invenio_mail.tasks.send_email
. invenio_notifications.tasks.broadcast_notification
. invenio_notifications.tasks.dispatch_notification
. invenio_oauthclient.tasks.create_or_update_roles_task
. invenio_rdm_records.fixtures.tasks.create_demo_community
. invenio_rdm_records.fixtures.tasks.create_demo_inclusion_requests
. invenio_rdm_records.fixtures.tasks.create_demo_invitation_requests
. invenio_rdm_records.fixtures.tasks.create_demo_oaiset
. invenio_rdm_records.fixtures.tasks.create_demo_record
. invenio_rdm_records.fixtures.tasks.create_vocabulary_record
. invenio_rdm_records.requests.access.tasks.clean_expired_request_access_tokens
. invenio_rdm_records.services.pids.tasks.register_or_update_pid
. invenio_rdm_records.services.tasks.reindex_stats
. invenio_rdm_records.services.tasks.send_post_published_signal
. invenio_rdm_records.services.tasks.update_expired_embargos
. invenio_records_resources.services.files.tasks.fetch_file
. invenio_records_resources.tasks.extract_file_metadata
. invenio_records_resources.tasks.manage_indexer_queues
. invenio_records_resources.tasks.send_change_notifications
. invenio_requests.tasks.check_expired_requests
. invenio_requests.tasks.request_moderation
. invenio_stats.tasks.aggregate_events
. invenio_stats.tasks.process_events
. invenio_users_resources.services.groups.tasks.reindex_groups
. invenio_users_resources.services.groups.tasks.unindex_groups
. invenio_users_resources.services.users.tasks.execute_moderation_actions
. invenio_users_resources.services.users.tasks.reindex_users
. invenio_users_resources.services.users.tasks.unindex_users
. invenio_vocabularies.services.tasks.process_datastream
. invenio_webhooks.models.process_event
```

then in first terminal
```shell
pipenv run invenio index init
pipenv run invenio rdm-records rebuild-index
```

error : [2024-02-15 15:34:26,817] WARNING in percolator: RequestError(400, 'mapper_parsing_exception', 'No field mapping can be found for the field with name [metadata.resource_type.id]')
[2024-02-15 15:34:26,856] WARNING in percolator: RequestError(400, 'mapper_parsing_exception', 'No field mapping can be found for the field with name [metadata.resource_type.id]')

```shell
pipenv run invenio communities rebuild-index
```

### Data migration

Execute the data migration, note that there is no need to re-index the data:

```bash
pipenv run invenio shell

run migrate_11_0_to_12_0.py
```

Got:
```bash
In [7]: run migrate_11_0_to_12_0.py
Starting data migration...
Updating record : a0y8n-t9n48
> Updated parent: nk1f1-st884
> Updated record: a0y8n-t9n48

Updating record : j9p5f-np862
> Updated parent: gtahm-50a83
> Updated record: j9p5f-np862

Updating record : 45898-zm095
> Updated parent: 5x0wd-gfj03
> Updated record: 45898-zm095

Commiting to DB
Data migration completed, please rebuild the search indices now.
```

```bash
pipenv run invenio rdm rebuild-all-indices
```

got some error : 

```bash
[2024-02-15 13:18:15,278: WARNING/ForkPoolWorker-5] POST http://localhost:9200/instance-dev-stats-record-view/_search [status:404 request:0.004s]
[2024-02-15 13:18:15,279: WARNING/ForkPoolWorker-5] [2024-02-15 13:18:15,278] WARNING in api: NotFoundError(404, 'index_not_found_exception', 'no such index [instance-dev-stats-record-view]', instance-dev-stats-record-view, index_or_alias)
[2024-02-15 13:18:15,278: WARNING/ForkPoolWorker-5] NotFoundError(404, 'index_not_found_exception', 'no such index [instance-dev-stats-record-view]', instance-dev-stats-record-view, index_or_alias)
[2024-02-15 13:18:15,286: WARNING/ForkPoolWorker-5] POST http://localhost:9200/instance-dev-stats-file-download/_search [status:404 request:0.005s]
[2024-02-15 13:18:15,287: WARNING/ForkPoolWorker-5] [2024-02-15 13:18:15,287] WARNING in api: NotFoundError(404, 'index_not_found_exception', 'no such index [instance-dev-stats-file-download]', instance-dev-stats-file-download, index_or_alias)
[2024-02-15 13:18:15,287: WARNING/ForkPoolWorker-5] NotFoundError(404, 'index_not_found_exception', 'no such index [instance-dev-stats-file-download]', instance-dev-stats-file-download, index_or_alias)
```

Change in inveniocfg
```bash
DATACITE_ENABLED = False
DATACITE_USERNAME = ""
DATACITE_PASSWORD = ""
DATACITE_PREFIX = "datacite_prefix"
DATACITE_TEST_MODE = True
DATACITE_DATACENTER_SYMBOL = ""
```

Then 

```bash
invenio-cli run
```

Topology Service
================

This document contains information about the service that runs:

* <https://topology.opensciencegrid.org>
* <https://topology-itb.opensciencegrid.org>
* <https://my.opensciencegrid.org>
* <https://my-itb.opensciencegrid.org>
* <https://myosg.opensciencegrid.org>
* <https://map.opensciencegrid.org>: Generates the topology map used on [OSG Display](https://display.opensciencegrid.org)

The source code for the service is in <https://github.com/opensciencegrid/topology>, in the `src/` subdirectory.
This repository also contains the public part of the data that gets served.


Deployment
----------

Topology is a webapp run with Apache on the host `topology.opensciencegrid.org`.
The ITB instance runs on the host `topology-itb.opensciencegrid.org`.
The hosts are VMs at Nebraska;
for SSH access, contact Derek Weitzel or Brian Bockelman.


### Installation

These instructions assume an EL 7 host with the EPEL repositories available.
The software will be installed into `/opt/topology`.
A second instance for the webhook app will be installed into `/opt/topology-webhook`.
(The ITB instance should be installed into `/opt/topology-itb` and `/opt/topology-itb-webhook` instead.)
The following steps should be done as root.

1.  Install prerequisites:

        :::console
        # yum install python36 gridsite httpd mod_ssl

1.  Clone the repository:

    For the production topology host:

        :::console
        # git clone https://github.com/opensciencegrid/topology /opt/topology
        # git clone https://github.com/opensciencegrid/topology /opt/topology-webhook

    For the topology-itb host:

        :::console
        # git clone https://github.com/opensciencegrid/topology /opt/topology-itb
        # git clone https://github.com/opensciencegrid/topology /opt/topology-itb-webhook

1.  Set up the virtualenv in the clone -- from `/opt/topology` or `/opt/topology-itb`:

        :::console
        # python36 -m venv venv
        # . ./venv/bin/activate
        # pip install -r requirements-apache.txt

1.  Repeat for the webhook instance -- from `/opt/topology-webhook` or `/opt/topology-itb-webhook`.


### File system locations

The following files/directories must exist and have the proper permissions:

| Location                                 | Purpose                                                         | Ownership     | Mode |
| --------                                 | -------                                                         | ---------     | ---- |
| `/opt/topology`                          | Production software install                                     | root:root     | 0755 |
| `/opt/topology-itb`                      | ITB software install                                            | root:root     | 0755 |
| `/opt/topology-webhook`                  | Production webhook software install                             | root:root     | 0755 |
| `/opt/topology-itb-webhook`              | ITB webhook software install                                    | root:root     | 0755 |
| `/etc/opt/topology/config-production.py` | Production config                                               | root:root     | 0644 |
| `/etc/opt/topology/config-itb.py`        | ITB config                                                      | root:root     | 0644 |
| `/etc/opt/topology/bitbucket`            | Private key for contact info repo                               | apache:root   | 0600 |
| `/etc/opt/topology/bitbucket.pub`        | Public key for contact info repo                                | apache:root   | 0644 |
| `/etc/opt/topology/github`               | Private key for pushing automerge commits                       | topomerge:root   | 0600 |
| `/etc/opt/topology/github.pub`           | Public key for pushing automerge commits                        | topomerge:root   | 0644 |
| `/etc/opt/topology/github_webhook_secret`| GitHub webhook secret for validating webhooks                   | topomerge:root   | 0600 |
| `~apache/.ssh`                           | SSH dir for Apache                                              | apache:root   | 0700 |
| `~apache/.ssh/known_hosts`               | Known hosts file for Apache                                     | apache:root   | 0644 |
| `~topomerge`                             | Home dir for `topomerge` Apache user                            | topomerge:root| 0755 |
| `~topomerge/.ssh`                        | SSH dir for `topomerge` Apache user                             | topomerge:root| 0700 |
| `~topomerge/.ssh/known_hosts`            | Known hosts file for `topomerge` Apache user                    | topomerge:root| 0644 |
| `/var/cache/topology`                    | Checkouts of topology and contacts data for production instance | apache:apache | 0755 |
| `/var/cache/topology-itb`                | Checkouts of topology and contacts data for ITB instance        | apache:apache | 0755 |
| `/var/cache/topology-webhook`            | Topology repo and state info for production webhook instance    | topomerge:topomerge | 0755 |
| `/var/cache/topology-itb-webhook`        | Topology repo and state info for ITB webhook instance           | topomerge:topomerge | 0755 |

`~apache/.ssh/known_hosts` must contain an entry for `bitbucket.org`;
use `ssh-keyscan bitbucket.org` to get the appropriate entry.

`~topomerge/.ssh/known_hosts` must contain an entry for `github.com`;
use `ssh-keyscan github.com` to get the appropriate entry.


### Software configuration

Configuration for the main app is under `/etc/opt/topology/`, in `config-production.py` and `config-itb.py`.
The webhook app configuration is in `config-production-webhook.py` and `config-itb-webhook.py`.
The files are in Python format and override default settings in `src/webapp/default_config.py` in the topology repo.

HTTPD configuration is in `/etc/httpd`; we use the modules `mod_ssl`, `mod_gridsite`, and `mod_wsgi`.
The first two are installed via yum;
the .so file for mod_wsgi is located in `/opt/topology/venv/lib/python3.6/site-packages/mod_wsgi/server/`
or `/opt/topology-itb/venv/lib/python3.6/site-packages/mod_wsgi/server/` for the ITB instance.

Each of the hostnames are VHosts in the apache configuration.  Some special notes:

* <https://map.opensciencegrid.org> runs in the same wsgi process as the production topology, but the URL is limited to only the map code.  Further, it does not use mod_gridsite so that users are not asked to present a client certificate.
* VHosts are configured:

        :::console
        ServerName topology.opensciencegrid.org
        ServerAlias my.opensciencegrid.org myosg.opensciencegrid.org


### Data configuration

Configuration is in `/etc/opt/topology/config-production.py` and `config-itb.py`;
and `config-production-webhook.py` and `config-itb-webhook.py`.

| Variable               | Purpose                                                                                                               |
|------------------------|-----------------------------------------------------------------------------------------------------------------------|
| `TOPOLOGY_DATA_DIR`    | The directory containing a clone of the `topology` repository for data use                                            |
| `TOPOLOGY_DATA_REPO`   | The remote tracking repository of `TOPOLOGY_DATA_DIR`                                                                 |
| `TOPOLOGY_DATA_BRANCH` | The remote tracking branch of `TOPOLOGY_DATA_DIR`                                                                     |
| `WEBHOOK_DATA_DIR`     | The directory containing a mirror-clone of the `topology` repository for webhook use                                  |
| `WEBHOOK_DATA_REPO`    | The remote tracking repository of `WEBHOOK_DATA_DIR`                                                                  |
| `WEBHOOK_DATA_BRANCH`  | The remote tracking branch of `WEBHOOK_DATA_DIR`                                                                      |
| `WEBHOOK_STATE_DIR`    | Directory containing webhook state information between pull request and status hooks                                  |
| `WEBHOOK_SECRET_KEY`   | Secret key configured on GitHub for webhook delivery                                                                  |
| `CONTACT_DATA_DIR`     | The directory containing a clone of the `contact` repository for data use                                             |
| `CONTACT_DATA_REPO`    | The remote tracking repository of `CONTACT_DATA_DIR`</br>(default: `"git@bitbucket.org:opensciencegrid/contact.git"`) |
| `CONTACT_DATA_BRANCH`  | The remote tracking branch of `CONTACT_DATA_BRANCH`</br>(default: `"master"`)                                         |
| `CACHE_LIFETIME`       | Frequency of automatic data updates in seconds</br>(default: `900`)                                                   |
| `GIT_SSH_KEY`          | Location of ssh public key file for git access.<br/>
                           `/etc/opt/topology/bitbucket.pub` for the main app, and `/etc/opt/topology/github.pub` for the webhook app            |

Puppet ensures that the production `contact` and `topology` clones are up to date with their configured remote tracking
repo and branch.
Puppet does not manage the ITB data directories so they need to be updated by hand during testing.


### GitHub Configuration for Webhook App

1.   Go to the [webhook settings](https://github.com/opensciencegrid/topology/settings/hooks) page on GitHub.
     There are four webhooks to set up; `pull_request` and `status` for both the topology and topology-itb hosts.

     | Payload URL                                                   | Content type     | Events to trigger webhook |
     |---------------------------------------------------------------|------------------|---------------------------|
     | https://topology.opensciencegrid.org/webhook/status           | application/json | Statuses                  |
     | https://topology.opensciencegrid.org/webhook/pull_request     | application/json | Pull requests             |
     | https://topology-itb.opensciencegrid.org/webhook/status       | application/json | Statuses                  |
     | https://topology-itb.opensciencegrid.org/webhook/pull_request | application/json | Pull requests             |

     For each webhook, "Secret" should be a random 40 digit hex string, which should match the contents of the file
     `/etc/opt/topology/github_webhook_secret` (the path configured in `WEBHOOK_SECRET_KEY`).

1.   The OSG's dedicated GitHub user for automating pushes is currently [osg-bot](https://github.com/osg-bot).

     1.  This user needs to have write access to the [topology](https://github.com/opensciencegrid/topology) repo on GitHub.

     1.  The ssh public key in `/etc/opt/topology/github.pub` should be registered with the `osg-bot` GitHub user.

         This can be done by logging into GitHub as `osg-bot`, and adding the new ssh key under the
         [settings](https://github.com/settings/keys) page.

### Required System Packages

Currently the webhook app uses the `mailx` command to send email.
If not already installed, install it with:

        :::console
        # yum install mailx


Testing changes on the ITB instance
-----------------------------------

All changes should be tested on the ITB instance before deploying to production.
If you can, test them on your local machine first.
These instructions assume that the code has not been merged to master.

1.  Update the ITB software installation at `/opt/topology-itb` and note the current branch:

        :::console
        # cd /opt/topology-itb
        # git fetch --all
        # git status

1.  Check out the branch you are testing.
    If the target remote is not configured, [add it](https://help.github.com/articles/adding-a-remote/):

        :::console
        # git checkout -b <BRANCH> <REMOTE>/<BRANCH NAME>

1.  Verify that you are using the intended data associated with the code you are testing:

    1. If the data format has changed in an incompatible way, modify `/etc/opt/topology/config-itb.py`:

        1. Backup the ITB configuration file:

                :::console
                # cd /etc/opt/topology
                # cp -p config-itb.py{,.bak}

        1. Change the `TOPOLOGY_DATA_DIR` and/or `CONTACT_DATA_DIR` lines to point to a new directories so the previous
           data does not get overwritten with incompatible data.

    1. If you need to use a different branch for the data, switch to it:

        1. Check the branch of `TOPOLOGY_DATA_DIR` from  `/etc/opt/topology/config-itb.py`

                :::console
                # cd <TOPOLOGY_DATA_DIR>
                # git fetch --all
                # git status

        1. Note the previous branch, you will need this later
        1. If the target remote is not configured, [add it](https://help.github.com/articles/adding-a-remote/)
        1. Check out the target branch:

                :::console
                # git checkout -b <BRANCH NAME> <REMOTE>/<BRANCH NAME>

    1. Pull any upstream changes to ensure that your branch is up to date:

            :::console
            # git pull

1.  For updates to the webhook app, follow the above instructions for the ITB webhook instance under
`/opt/topology-itb-webhook` and its corresponding config file, `/etc/opt/topology/config-itb-webhook.py`.

1.  Restart `httpd`:

        :::console
        # systemctl restart httpd

1.  Test the web interface at <https://topology-itb.opensciencegrid.org>.

    Errors and output are in `/var/log/httpd/error_log`.


### Reverting changes

1.  Switch `/opt/topology-itb` to the previous branch:

        :::console
        # cd /opt/topology-itb
        # git checkout <BRANCH>

1.  For updates to the webhook app, switch `/opt/topology-itb-webhook` to the previous master:

        :::console
        # cd /opt/topology-itb-webhook
        # git checkout <BRANCH>

1.  If you made config changes to `/etc/opt/topology/config-itb.py` or `config-itb-webhook.py`, restore the backup.

1.  If you checked out a different branch for data, revert it back to the old branch.

1.  Restart `httpd`:

        :::console
        # systemctl restart httpd

1.  Test the web interface at <https://topology-itb.opensciencegrid.org>.



Updating the production instance
--------------------------------

Updating the production instance is similar to updating ITB instance.

1.  Update master on the Git clone at `/opt/topology`:

        :::console
        # cd /opt/topology
        # git pull origin master

1.  For updates to the webhook app, update master on the Git clone at `/opt/topology-webhook`:

        # cd /opt/topology-webhook
        # git pull origin master

1.  Make config changes to `/etc/opt/topology/config-production.py` and/or `config-production-webhook.py` if necessary.

1.  Restart `httpd`:

        :::console
        # systemctl restart httpd

1.  Test the web interface at <https://topology.opensciencegrid.org>.

    Errors and output are in `/var/log/httpd/error_log`.


### Reverting changes

1.  Switch `/opt/topology` to the previous master:

        :::console
        # cd /opt/topology
        ### (use `git reflog` to find the previous commit that was used)
        # git reset --hard <COMMIT>

1.  For updates to the webhook app, switch `/opt/topology-webhook` to the previous master:

        # cd /opt/topology-webhook
        ### (use `git reflog` to find the previous commit that was used)
        # git reset --hard <COMMIT>

1.  If you made config changes to `/etc/opt/topology/config-production.py` or `config-production-webhook.py`, revert them.

1.  Restart `httpd`:

        :::console
        # systemctl restart httpd

1.  Test the web interface at <https://topology.opensciencegrid.org>.



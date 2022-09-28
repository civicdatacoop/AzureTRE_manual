# Manual for maintenance and deployment of Azure TRE
This extends the official manual available here: https://microsoft.github.io/AzureTRE/

## General notes
**Note:** Be aware that data traffic from and to the machine exceeds 20GB.

**Note:** Sometimes steps of Makefile may fail without any good reason, in that case, wait 15 min and run it again (it’s usually because state on Azure is pending).

**Note:** `all` caption of `Makefile` is overridden.

## Deployment of Azure TRE using WSL (or Ubuntu):

Before anything, go to bash (or WSL on Windows) and do the following:

1. Install the following system packages: `sudo apt install gcc g++ make jq git`
2. Install Azure CLI (if needed): `curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash`
3. Log into Azure: `az login`
4. Install Terraforms, see: https://learn.hashicorp.com/tutorials/terraform/install-cli
5. Activate Docker on WSL (relevant only on Windows): see: https://docs.docker.com/desktop/windows/wsl/
6. Install the latest version of Node.JS and yarn: https://computingforgeeks.com/install-latest-node-js-and-npm-on-ubuntu-debian/ 
7. Install the latest version of yarn (default repo in Ubuntu does not contain the correct version), see: https://phoenixnap.com/kb/how-to-install-yarn-ubuntu
8. Install Porter (https://porter.sh/install/) in version v0.38.12 (or compatible, but not >=v1 which does not work). Also, export `PATH` var with porter's path whenever system is rebooted (or after new installation). To add porter to `PATH` env variable, use:
```shell
export PORTER_HOME=~/.porter
export PATH=$PORTER_HOME:$PATH
porter --version
```
9. Install the yq: see https://mikefarah.gitbook.io/yq/v/v3.x/
10. Follow the manual to do the rest (but read all notes & warnings below in advance).

**Note:** To find the proper acronym of the region, see: https://azuretracks.com/2021/04/current-azure-region-names-reference/

**Critical warning:** do not use a hyphen (-) when naming resources (or anything else); use lower-case only (particularly for the ID of TRE) and strings below 12 characters.

**Warning:** running the scripts for the first time takes around 2 hours (not 30 minutes as promised).

**Critical warning:** run `make auth` before `make all`

**Note:** you may need to update `templates/workspaces/base/.env` file (created by renaming or copying .env.sample template) – this is not mentioned in the documentation. Use the file generated by make auth to find the info. This is pertinent only if you wish to run end-to-end tests.

## After deployment
1. Install certbot (`sudo apt install certbot`)
2. Check if `/opt/certbot/bin/certbot` is the path to certbot command, if not, change the paths in files:
    1. /devops/scripts/check_dependencies.sh (line 72)
    2. /templates/core/terraform/scripts/letsencrypt.sh (line 94)
    3. /templates/shared_services/certs/scripts/letsencrypt.sh (line 100)
    4. Add `--manual-public-ip-logging-ok` to calls (in last two files) – but first, check if your version of certbot supports this flag
3. Run `make letsencrypt`
4. Check it all using `curl https://URL/api/health` (that URL can be found in IMPORTANT NOTES output of make letsencrypt command, it is something like tremcdvid.uksouth.cloudapp.azure.com)

**Warning:** Be aware that you have just 20 attempts for registering on letsencrypt per week (better to check twice before you run anything)

## Shared services
Follow the manual: https://microsoft.github.io/AzureTRE/tre-admins/setup-instructions/configuring-shared-services/

**Note:** Before that, check if you have properly configured files that use certbot (as described in the previous step)

**Warning:** In the case that your app logs the following error:
`The redirect URI 'WHATEVER' specified in the request does not match the redirect URIs configured for the application 'WHATEVER'. Make sure the redirect URI sent in the request matches one added to your application in the Azure portal. Navigate to https://aka.ms/redirectUriMismatchError to learn more about how to fix this.`
Go to portal.azure.com > Azure Active Directory > App registrations > (find the matching app by ID from the error message) > Authentication > (Add mechanically URL from error message)

**Critical warning:** when clicking 'Authorization' in Swagger, leave the `client_secret` field empty.

**Critical warning:** you need to assign permissions (namely TREUser and TREAdmin) to a user you want to use for managing TRE (typically yourself). Do this before authorization (it takes around 10 minutes before permissions completely propagate).

**Note:** think twice what you really need to deploy (probably you do not need, for example, Gitea).

## Base Workspace
1. Follow the manual (step number 6).
2. To register workspace, run in Swagger using route `/api/workspaces` method POST:
```json
{
  "templateName": "tre-workspace-base",
  "properties": {
    "display_name": "manual-from-swagger",
    "description": "workspace for team X",
    "client_id":"auto_create",
    "address_space_size": "medium"
  }
}
```
(that `auto_create` argument is an undocumented but critical).

**Warning:** the process very often fails (without any meaningful error message), if that happens, wait 10 minutes and repeat it (it can take up to 5 times).

## Installing workspace service and user resource
Follow the manual, but in the first step, you need
to build the container first, therefore, use.
```shell
make bundle-build DIR=./templates/workspace_services/guacamole
make bundle-publish DIR=./templates/workspace_services/guacamole BUNDLE_TYPE=workspace_service
make bundle-register DIR=./templates/workspace_services/guacamole BUNDLE_TYPE=workspace_service
```
The first line (building step) needs to be added; that is the difference.

Also, then you need to generate JSON Payload
manually, using a specialised script (that is
not mentioned in the manual either).

1. Go to the folder with a workspace service, e. g.:
   1. `cd ./templates/workspace_services/guacamole`
2. Run the command:
   1. `../../../devops/scripts/register_bundle_with_api.sh -r ACR_NAME -i -t workspace`
   2. but first, in this command, replace the `ACR_NAME` with a proper value (you can find it in the `/devops/.env` file)
3. Now, you should see the JSON Payload.

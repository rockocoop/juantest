Job Template and Workflow Template automation for Ansible Tower
===============================================================

This project concerns automating job template and workflow template creation on Tower.
It provides a way to export job/workflow templates as yaml files and a way to deploy new job/workflow templates based on yaml files in the same format.
The primary usecase is to allow migrating job/workflow templates between environments in an easier fashion.
The job/workflow template definitions are designed to be stored in a git repository, which also serves as a good backup in case accidental changes are made in Tower.


Credentials
-----------

A technical user is required to be used for interacting with the Tower API and SFS. The permissions of this TU requires the following permissions to be requested from Ansible Tower team:

1. "Workflow Admin" permissions inside the uuc org (this is required if workflow templates need to be imported)

2. Assigned to the same AD group as the rest of the users from the specific team

3. If some Job Templates that need to be updated using an import and the Job Templates don't have the team (ex. 'team-linux') assigned as "Admin", then "Job Template Admin" permissions are required in the uuc org

A new credential of type "Ansible Tower" needs to be created on Tower containing the username and password to this TU and the location of Tower (ex. <https://atpfe-test.internal.unicreditgroup.eu> for test)


How to use
----------

1. Make a new repository on Kyndryl GitHub

2. Clone this repository and update the remote to point your new one

3. Make a new branch (ex. `prod`, `noprod`)

4. Switch to the new branch and remove the contents of the `job_templates/` directory (if there are any)

5. Commit your changes and push this repository to your newly created remote on Kyndryl GitHub

6. Create a new project on Tower pointing to your repository

7. Create 2 job templates for the 2 playbooks (`create_job_templates.yml` and `export_job_templates.yml`) as described bellow

To update your repository you need to merge changes from the `main` branch of this repository.

Exporting
---------

The playbook `export_job_templates.yml` handles exporting **job templates and workflow templates**, storing them as yaml files and uploading them to SFS.

The job template for this playbook should be configured like the following:

- Name, Description: use a descriptive name, like any other job templates
- Job Type: Run
- Inventory: You can set ANY inventory here, all tasks run on `localhost` which is implicitly part of all inventories. No tasks will run on remote machines.
- Project: The project you created pointing to your version of this repo
- Playbook: `export_job_templates.yml`
- Credentials: Type: Ansible Tower, the credential you created for your TU (no others are needed)
- Extra Vars: `upload_url: 'https://{{ sfs_host }}/files'` (to make sure SFS upload works correctly)

All other parameters can be left as default, "Limit" is not necessary since all tasks run on localhost.

**Survey**

This playbook takes a couple variables as input.
It's a good idea to set up a survey for this.

*If this looks like a lot of work you can import the survey spec through the api from the [`doc/export_survey_spec.json`](doc/export_survey_spec.json) file.*
*[How to do this](doc/set_survey_spec_using_api.md)*

Add 3 new prompts as follows:

1. Workflow template names:

    - Prompt: `Names of Workflow Templates to export`
    - Description: `one name on each line`
    - Answer Var Name: `wf_names_text`
    - Answer Type: `Textarea`
    - Min Length: 0
    - Max Length: Set this high enough so you won't run into this character limit (ex. 65535)
    - Default Answer: leave empty
    - Required: **NO**

2. Job template names:

    - Prompt: `Names of Job Templates to export`
    - Description: `one name on each line`
    - Answer Var Name: `jt_names_text`
    - Answer Type: `Textarea`
    - Min Length: 0
    - Max Length: Set this high enough so you won't run into this character limit (ex. 65535)
    - Default Answer: leave empty
    - Required: **NO**

3. Export job templates contained in workflow templates:

    - Prompt: `Also export Job Templates that are Nodes of exported Workflow Templates`
    - Description: `If this is "True" Job Templates "inside" Workflow Templates specified above are also exported`
    - Answer Var Name: `export_wf_child_jts`
    - Answer Type: `Multiple Choice (single select)`
    - Multiple choice options:
        + `True`
        + `Fasle`

    - Default Answer: `True`
    - Required: **YES**

If you do not want to use a survey you can provide the job template names through "Extra Vars" using the variable `jt_names` **which takes a list**.
The same is true for workflow templates with the variable `wf_names` which also expects a list of workflow template names.

**Results**

The results are uploaded to SFS in the default location of `uuc/{{ tower_project_name }}/{{ timestamp }}.zip`.
The zip file contains two directories `job_templates/` and `workflow_templates/`.
Each contains the yaml definition files for job or workflow templates in files named `{{ job_template_name }}.yml` or `{{ workflow_template_name }}.yml`.


Deploying
---------

The playbook `create_job_tempaltes.yml` handles creating **job templates and workflow templates** defined in your repository.

Workflow templates are sourced from the `workflow_templates/` directory while job templates from the `job_templates/` directory.
All files ending in `.yml` present in these directories will be imported when the playbook runs.

The playbook can be used both to create new templates and restore existing ones to a "known good" configuration.
When a job/workflow template with an identical name is found it will attempt to modify it to match the configuration stored in the yaml files.

The job template for this playbook should be configured like the following:

- Name, Description: use a descriptive name, like any other job templates
- Job Type: Run
- Inventory: You can set ANY inventory here, all tasks run on `localhost` which is implicitly part of all inventories. No tasks will run on remote machines.
- Project: The project you created pointing to your version of this repo
- Playbook: `create_job_templates.yml`
- Credentials: Type: Ansible Tower, the credential you created for your TU (no others are needed)

All other parameters can be left as default, "Limit" is not necessary since all tasks run on localhost, no "Extra Vars" are needed.

**Using exported yaml files**

Yaml files generated by the export playbook can simply be placed in the `job_templates/` or `workflow_templates` directories, and they will be deployed by this playbook on the next run.
**This only works if all resources** (projects, inventories, credentials) **are named exactly the same on all environments**.
Otherwise the creation of job/workflow templates will only work correctly on the environment the export was run on.

To correctly import a whole workflow template the job templates used within it have to also be imported at the same time or they have to already exist in the environment.

**Dealing with diffferences in environments**

If there are differences between environments in the names of projects, inventories or credentials that are used in the job templates you wish to deploy you should use a separate branch for every environment (`prod`, `noprod`...).
By editing the yaml files provided by the export and keeping different versions of them on these branches you can make the deploy playbook work correctly on all environments.

If you use Linux/Mac or you have `sed` available on Windows/WSL you can use it to quickly replace `_noprod` endings in project/inventory/credential/job template names with `_prod` on all your job template files like this:

```
sed -i -e 's/_noprod/_prod/g' job_templates/*.yml
```

You can then commit those changes to the prod branch.

How to set the survey spec using the Tower API
==============================================

This is a description of how to set the survey spec for the Export job template through the Tower API.

This might save you a bit of time and some typing when setting up this job template.

Navigate to `{tower host}/api/v2/job_templates/`, log in with your credentials you use for Tower if you're not already.
*Example link for TEST Tower: <https://atpfe-test.internal.unicreditgroup.eu/api/v2/job_templates/>*

Search for the your job template by it's **exact name** like the following `{tower host}/api/v2/job_templates/?name={name of your export job templae}`.
There should be one result note down the number after `"id:"`.

Navigate to `{tower host}/api/v2/job_templates/{id of your job template}/survey_spec/`, copy the contents of [`doc/export_survey_spec.json`](export_survey_spec.json) paste it to the `CONTENT` field at the bottom of the page and click `POST`.

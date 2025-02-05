License Scanning is often considered part of Software Composition Analysis (SCA). SCA can contain aspects of inspecting the items your code uses. Open source software licenses define how you can use, modify and distribute the open source software. Thus, when selecting an open source package to merge to your code it is imperative to understand the types of licenses and the user restrictions the package falls under, which helps you mitigate any compliance issues. 

This config template can be included in your `.gitlab-ci.yml` to get the scanning job for free (similar to how the gitlab container scanning thing works).

## Setup Instructions
At the very top of your .gitlab-ci.yml either add or expand the `include:` section so it looks similar to this:  
```yaml
include:
  - remote: https://raw.githubusercontent.com/ambient-innovation/gitlab-trivy-license-checks/main/license-checks.yaml
  # There might be more includes here, usually starting with template like the following:
  # - template: 'Workflows/Branch-Pipelines.gitlab-ci.yml'
```

You will also need to have at least one stage called test in your top-level stages config for the default configuration:  
```yaml
stages:
  - prebuild
  - build
  - test
  - posttest
  - deploy
```  
**The `test` stage has to come after the docker image has already been built and pushed to the registry or the scanner will not work.**

Last but not least you need a job within that test stage going by the name `license_scanning`. A minimal config looks like this:  
```yaml
license_scanning:
  variables:
    IMAGE: $IMAGE_TAG_BACKEND
```

The example shown here will overwrite the `license_scanning` job from the template and tell it to

a) scan an image as specified in the `IMAGE_TAG_BACKEND` variable,\
b) perform a simple license scan\
c) only report errors with a level of HIGH,CRITICAL or UNKNOWN. 

You can also specify the `FILENAME` of the result-output as you like. 

**Note:** If you wish to run the `license_scanning` job in another stage than "`test`" (as it does by default) simply copy the above code to your .gitlab-ci.yml file and add the keyword `stage` with your custom stage name.

Example for minimal stage-overwrite setup:

```yaml
license_scanning:
  stage: my-custom-stage
```

## Show reports in the Merge-Request UI
To get a singular job to report on all the scanning errors in one place, add a check scan results job that combines the other job results.  

Here's an example of how that job could look like:  
```yaml
check security scan results:
  stage: posttest
  image: ${CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX}/alpine:latest
  dependencies:
    - license_scanning
    # List all your container scanning jobs here, one per line, i.e.:
    # - container_scanning_frontend
    # List all your license scanning jobs here, one per line, i.e.:
    # - license_scanning_frontend
  tags:
    - low-load
  before_script:
    - apk update && apk add jq coreutils grep
  script:
    - echo "Step 1 - Merge all codeclimate reports from scanning jobs"
    - jq -s 'add' gl-codeclimate-*.json > gl-codeclimate.json
    - echo "Step 2 - Check if there were any vulnerabilities and exit with a status code equal to the number of vulnerabilities"
    - jq '.[].type' .\gl-codeclimate.json | grep "issue" | exit $(wc -l)
  # Enables https://docs.gitlab.com/ee/user/application_security/container_scanning/ (Container Scanning report is available on GitLab EE Ultimate or GitLab.com Gold)
  artifacts:
    paths:
      - gl-codeclimate.json
    reports:
      codequality: gl-codeclimate.json
```

If all is good, you'll see a new green bar above your test results.  
If any vulnerabilities were found, you'll see a new yellow bar above the test results in your Merge-Request:  
![Code Quality Seal](images/codequality-seal.jpg)  
You can then expand that section and see all the results:  
![Critical Code Quality Issues](images/codequality-critical.jpg)

You can also just check the failed scanning jobs for a plaintext error report. This can also include additional details not visible in the GitLab-UI.  
Currently trivy has no way to exclude operating system packages from the plaintext report, so this output will contain false-positive reports for those.  
Just scroll past the first table and only check the one that says Node.js, Python or whatever the programming language used for your project might be.

## Interpreting the output
Each job will print a table with all licenses found. At the time of writing this, trivy will always print the licenses of OS packages as well, but you should not be concerned about any of them.  

The merge-request widget / the json artifacts already filter out OS packages, but if you need to look at the tabular output, you can safely ignore the OS Packages and only look at the report for programming language packages (i.e.: Python packages, Node.js packages etc.)

Packages with a Severity of unknown either didn't provide a License at all or didn't match any of the licenses known to the software.  
If the license field says UNKNOWN, you should verify the license by hand at the package repository. (i.e. NPM, PyPi, GitHub, ...)  
If the license field has a license name in it, it's probably an alternative spelling that isn't yet recognized by trivy. Please open an issue or PR on our GitHub to add the new spelling to the trivy.yaml file in this repository.

Packages with a Severity rating of CRITICAL most likely require you to disclose the sourcecode of your application to every user/visitor. Check with the Privacy Circle if in doubt.  
Packages with a Severity rating of HIGH usually have a viral license like GPL. Such packages are most likely fine if used in the backend, but might require you to disclose the sourcecode to every user if used in the frontend. Check with the Privacy Circle if in doubt.

## Scanning multiple images/directories (i.e. frontend and backend)  
To scan multiple images/directories, you can simply copy the job above, add another key `extends: license_scanning` and change the variable values for the other container.

Here's an example:
```yaml
license_scanning_frontend:
  extends:
    - license_scanning
  variables:
    IMAGE: $IMAGE_TAG_FRONTEND
```

## Unknown licenses detected / licenses mismatched
Trivy compares the license names it finds to a static list of names contained in it's binary distribution. It's very likely that not all your dependencies will match against this list due to typos, different spellings or dual-licensing. In these cases you will have to create your own custom mapping in a config file called trivy.yaml.

To get you started, this repository ships with it's own trivy.yaml where we already matched a few common misspellings of license names into their corresponding categories. Unless specified otherwise, this scanner job will download the trivy.yaml and use that. We'd like to encourage you to submit new license mappings as PRs to this repository.

We will however not adjust the severity of individual licenses in this repository. If your project allows for strong-copyleft-licenses to be used or requires that you can't disclose library authors to your users for example, you will have to edit the trivy.yaml in your own repository.
You can download our main-copy and store it somewhere in your project source to modify it. Then point the license scanner at your personal config file using the TRIVY_YAML environment variable.

Here's an example:
```yaml
license_scanning:
  variables:
    TRIVY_YAML: './frontend/custom-trivy.yaml'
```

Some packages in the python world report the full license text as their license. Until that is fixed upstream these packages will be reported having unknown licenses and show up as an error.  
As of October 2023, we know at least `pipenv` and `traitlets` do this.  
Because pipenv is only needed during setup, you can easily remove that requirement from your pipfile without any disadvantage.


## Advanced Settings  
The container scanning job exposes a few more variables by which you can adjust the scanning if needed. The default settings are the recommendation of the TE-Circle, though.  

### Change minimum severity reported
By adding a new variable called `SEVERITY` to your job, you can change which severity items should be reported. The default is to report UNKNOWN, HIGH and CRITICAL licenses. The remaining options are: `LOW`, `MEDIUM`  
Trivy requires a full list of severities to report. To report all severities from MEDIUM and higher for example, you need to specify a comma-separated list like so: `SEVERITY: "MEDIUM,HIGH,CRITICAL,UNKNOWN"`

We recommend only scanning for licenses with a CRITICAL or UNKNOWN level in the backend as that is typically not considered to be distributed and licenses such as the GPL are perfectly fine to use without having to disclose the project sources to users. In frontend settings where users download your (minified) JavaScript, you should set the report level to include HIGH as well. Licenses such as the GPL would otherwise require to disclose the sourcecode to any user who requests it.

### Setting exit code for scanner

Since Trivy is not able to exclude findings for OS-level packages, we will have some matches in basically every case. 
A pipeline, that always warns, will in reality never warn. That's why the default here is set to "0", meaning that it 
will be "green" in GitLab. If you want to change this, just set the variable `EXIT_CODE_ON_FINDINGS`.

Here's an example:
```yaml
license_scanning:
  variables:
    EXIT_CODE_ON_FINDINGS: 1
```

### Other settings
By default trivy performs one run in full-silence mode writing the results to the gitlab codeclimate report file and then another one showing the results in a plaintext table. If the scan is taking very long, you can also show a progress bar during the scan by setting the `TRIVY_NO_PROGRESS` variable to `"false"`.  
To make sure you're doing a fresh run and instruct trivy to download a fresh vulnerability database, you can turn off/move the cache directory via the `TRIVY_CACHE_DIR` variable. The default value for this variable is a directory called `.trivycache`

You can add more variables corresponding to the CLI switches as [documented on the trivy homepage](https://aquasecurity.github.io/trivy/v0.48/docs/references/configuration/cli/trivy/)  
NOTE: This link points to the reference as of v0.48 - December 2023, make sure to check the latest version for changes in newer versions.

This config ships with a default trivy.yaml that contains some additional license mappings for common misspellings of licenses. If you'd rather include your own mapping table, download the trivy.yaml file from this repository and store it in your git repository. You can then edit it and configure the Scanner to use your own file instead by providing the path to your trivy.yaml file in the TRIVY_YAML environment variable.

If you find missing license mappings, we would much appreciate it if you'd just add a PR to this repository with your changes or open an issue for assistance.

# Chef Azure DevOps CI/CD Pipelines templates and examples

_Note: steps use the `v3` of the [Azure DevOps Chef Extension](https://github.com/chef-partners/azuredevops-chef-extension)(currently in limited preview) but can be replaced with standard script step approach._

## Policyfile Workflow

[Azure DevOps Pipelines YAML templates](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops) for common steps and stages, as well as examples of usage in complete pipelines in the `example_pipelines` directory for:

* **Cookbook CI (with a Policyfile for dependency resolution only, instead of Berksfile)**
  * Prep:
    * Git tag for version in `metadata.rb` does not exist yet
    * Cookbook `metadata.rb`:
      * Lists at least one supported platform
      * Specifies a Chef Version
      * Any dependencies are pinned using `~>` pessimistic (or `=` equality) [version constraints](https://docs.chef.io/cookbook_versioning/#constraints)
  * Lint:
    * Cookstyle
    * ChefSpec
  * Integration Testing:
    * kitchen-azure
    * kitchen-vsphere
  * Release:
    * Git Tagging with version in `metadata.rb`
* **Policyfile CI/CD**
  * Including promotion using [Azure DevOps Environments](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/environments?view=azure-devops) mapped to Chef Infra Server Policy Groups 
    * Uses the new [YAML CD features in Azure DevOps Pipelines](https://devblogs.microsoft.com/devops/announcing-general-availability-of-azure-pipelines-yaml-cd/). 
    * Example of [Adding Approvals to Azure DevOps YAML Pipelines](https://medium.com/faun/adding-approvals-to-azure-devops-yaml-pipeline-21f41578677b).
    * [Azure DevOps Environments Approval checks](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops&tabs=check-pass). 
  * Prep:
    * Update Policyfile Lock JSON (`chef install` and `chef update`)
    * Publish updated Policyfile Lock JSON as pipeline artifact
  * Integration Testing:
    * kitchen-azure
    * kitchen-vsphere
  * Publish:
    * Export Policyfile archive (`chef export -a`)
    * Publish updated Policyfile Lock JSON as build artifact
  * Release/Deploy:
    * Use `deployment` jobs with `environments` mapped to policy groups and push Policyfile archive to Policy group (`chef push-archive`)

### Example Policyfiles

```ruby
# A name that describes what the system you're building with Chef does.
name 'test-policy'

# Where to find external cookbooks:
default_source :supermarket

# run_list: chef-client will run these recipes in the order specified.
run_list 'test::default', 'chef-client::default', 'audit::default'

# Specify a custom source for a single cookbook:
cookbook 'test', git: 'https://AZUREREPOHERE/_git/test', tag: 'v0.1.8'

default['development']['test']['message'] = 'test message for development'
default['production']['test']['message'] = 'test message for production'
```

The Policyfile CI/CD pipeline also works with Policyfiles that use `include_policy` from another repo (by automatically commiting the Policyfile Lock JSON so it's available for other policies to include) e.g.

```ruby
# This is the file server policy for all Chef managed file servers.
name 'another-policy'

# Where to find cookbooks:
default_source :supermarket

# include base policy
include_policy 'test-policy', git: 'https://AZUREREPOHERE/_git/test-policy', path: 'test-policy.lock.json'

# Chef-client will run these recipes in the order specified.
run_list 'openssh::default'
```

## Legacy Workflow

`example_pipelines` directory also includes legacy workflow pipelines for:
* **Chef Repo CI/CD (environment, roles, data bags)**
  * Lint:
    * JSON Lint
    * Legacy Workflow safety checks including:
      * Every cookbook listed in environment files is pinned with equality `=` pin
      * Every cookbook version listed in environment files exists on the Chef Infra Server
      * Every environment file pins every uploaded cookbook
  * Upload:
    * Upload Roles, Environments and Data Bags
      
* **Cookbook CI/CD (with Berksfile)**
  * Prep:
    * Git tag for version in `metadata.rb` does not exist yet
    * Cookbook or cookbook version has not been uploaded to the Chef Infra Server organisation yet
    * Cookbook `metadata.rb`:
      * Lists at least one supported platform
      * Specifies a Chef Version
      * Any dependencies are pinned using `~>` pessimistic (or `=` equality) [version constraints](https://docs.chef.io/cookbook_versioning/#constraints)
    * Publish Berksfile Lock as pipeline artifact
  * Lint:
    * Cookstyle
    * ChefSpec
  * Integration Testing:
    * kitchen-azure
    * kitchen-vsphere
  * Release:
    * Berks Upload
    * Git Tagging with version in `metadata.rb`

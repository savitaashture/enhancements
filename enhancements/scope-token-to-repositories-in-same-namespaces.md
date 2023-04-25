---
title: Extend GitHub token scope to a list of provided repositories within and outside namespaces
authors:
  - "@savita"
creation-date: 2023-04-17
status: implementable
---

# Extend GitHub token scope to a list of provided repositories within and outside namespaces

## Summary

By default, the GitHub token that Pipelines as Code generates is scoped only to the repository where the payload comes from.

However, in some cases, the developer team might want the token to allow control over additional repositories. For example, there might be a CI repository where the `.tekton/pr.yaml` file and source payload might be located, however the build process defined in `pr.yaml` mightg fetch tasks from a separate private CD repository.

You can extend the scope of the GitHub token in two ways:

* _Repository level configuration_: extend the token a list of repositories that exist in the same namespace as the original repository.

* _Global configuration_: extend the token a list of repositories in different namespaces.

## Prerequisite

In the `pipelines-as-code` configmap, set the `secret-github-app-token-scoped` key to `false`. This setting enables the scoping of the GitHub token to private and public repositories listed under the Global and Repository level configuration.

### Scoping the GitHub token using Global configuration

You can use global configuration to set a list of repositories in any namespaces.

To set the global configuration, in the `pipelines-as-code` configmap, set the `secret-github-app-scope-extra-repos` key, as in the following example:

  ```
  apiVersion: v1
  data:
    secret-github-app-scope-extra-repos: "owner2/project2, owner3/project3"
  kind: ConfigMap
  metadata:
    name: pipelines-as-code
    namespace: pipelines-as-code
  ```

### Scoping GH token to a list of Repos provided by Repository level configuration

You can use the `Repository` custom resource to scope the generated GitHub token to a list of repositories. The repositories can be public or private, but must reside in the same namespace as the repository with which the `Repository` resource is associated.

Set the `repo_list_to_scope_token` spec configuration within the `Repository` custom resource, as in the following example:

  ```
  apiVersion: "pipelinesascode.tekton.dev/v1alpha1"
  kind: Repository
  metadata:
    name: test
    namespace: test-repo
  spec:
    url: "https://github.com/linda/project"
    repo_list_to_scope_token:
    - "owner/project"
    - "owner1/project1"
  ```

In this example, the `Repository` custom resource is associated with the `linda/project` repository in the `test-repo` namespace. The scope of the generated GitHub key is extended to the `owner/project` and `owner1/project1` repositories, as well as the `linda/project` repository. These repositories must exist under the `test-repo` namespace.

**Note:**

If any of the repositories does not exist in the namespace, the generation of the GitHub token fails with an error message as in the following example:

     ```
     repo owner1/project1 does not exist in namespace test-repo
     ```

### Scenarios for providing global and repository level configurations

1. When you provide both a `secret-github-app-scope-extra-repos` key in the `pipelines-as-code` configmap and a `repo_list_to_scope_token` spec configuration in the `Repository` custom resource, the token is scoped to all the repositories from both configurations, as in the following example:

    * `pipelines-as-code` configmap:

        ```
        apiVersion: v1
        data:
          secret-github-app-scope-extra-repos: "owner2/project2, owner3/project3"
        kind: ConfigMap
        metadata:
          name: pipelines-as-code
          namespace: pipelines-as-code
        ```

    * `Repository` custom resource

        ```
        apiVersion: "pipelinesascode.tekton.dev/v1alpha1"
        kind: Repository
        metadata:
          name: test
          namespace: test-repo
        spec:
          url: "https://github.com/linda/project"
          repo_list_to_scope_token:
          - "owner/project"
          - "owner1/project1"
        ```

    The GitHub token is scoped to the following repositories: `owner/project`, `owner1/project1`, `owner2/project2`, `owner3/project3`, `linda/project`.

2. If you set only the global configuration in the `secret-github-app-scope-extra-repos` key in the `pipelines-as-code` configmap, the GitHub token is scoped to all the listed repositories, as well as the original reporitory from which the payload files come.

3. If you set only the `repo_list_to_scope_token` spec in the `Repository` custom resource, the GitHub token is scoped to all the listed repositories, as well as the original reporitory from which the payload files come. All the repositories must exist in the same namespace where the `Repository` custom resource is created.

4. If you did not install the GitHub app for any repositories that you list in the global or repository level configuration, creation of the GitHub token fails with the following error message:

```
failed to scope token to repositories in namespace article-pipelines with error : could not refresh installation id 36523992's token: received non 2xx response status \"422 Unprocessable Entity\" when fetching https://api.github.com/app/installations/36523992/access_tokens: Post \"https://api.github.com/repos/savitaashture/article/check-runs\
```

5. If the scoping of the GitHub token to the repositories set in global or repository level configuration fails for any reason, the CI process does not run. This includes cases where the same repository is listed in the global or repository level configuration, and the scoping fails for the repository level configuration because the repository is not in the same namespace as the `Repository` custom resource.

In the following example, the `owner5/project5` repository is listed in both the global configuration and in  tyhe repository level configuration:

    ```
    apiVersion: v1
    data:
      secret-github-app-scope-extra-repos: "owner5/project5"
    kind: ConfigMap
    metadata:
      name: pipelines-as-code
      namespace: pipelines-as-code
    ```

    ```
    apiVersion: "pipelinesascode.tekton.dev/v1alpha1"
    kind: Repository
    metadata:
      name: test
      namespace: test-repo
    spec:
      url: "https://github.com/linda/project"
      repo_list_to_scope_token:
      - "owner5/project5"
    ```

In this example, if the `owner5/project5` repository is not under the `test-repo` namespace, creation of the GitHub token fails with the following error message:

    ```
    repo owner5/project5 does not exist in namespace test-repo
    ```

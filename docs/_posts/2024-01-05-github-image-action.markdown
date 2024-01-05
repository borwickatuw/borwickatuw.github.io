---
layout: default
title: "Github action to purge container repository images"
---

We use podman images for development, stored in the GitHub container registry. Our process adds new images to the registry, but we need a way to "clean up" the registry. Here's the github action, stored in `.github/workflows/main.yml` in the default repository branch:

    name: Delete old container images

    on:
      schedule:
        - cron: "0 0 * * SUN" # Sundays
      workflow_dispatch:

    jobs:
      clean-ghcr:
        name: Delete old unused container images
        runs-on: ubuntu-latest
        steps:
          - name: Delete untagged containers older than 1 month
            uses: snok/container-retention-policy@v2
            with:
              image-names: FIXME*
              cut-off: A month ago UTC
              account-type: org
              org-name: FIXME
              keep-at-least: 1
              skip-tags: develop, main, production, v*
              token: ${{ secrets.PAT }}

This job relies on [container-retention-policy](https://github.com/snok/container-retention-policy). There are a few notable parts:

  * Listing `workflow_dispatch:` in the `on:` stanza lets you run the job manually
  * `image-names` accepts wilcards, which works with our naming convention
  * for `${{secrets.PAT}}`, this goes into the secrets for the repository to find whatever you called the secret. To get a secret, you need to generate it from your account and give it permission to view and delete registry data. Once you have it, you populate the secret from the repository > settings > security/secrets and variables > repository secrets.

  * We were using `untagged-only: true`, but our job was tagging containers with their git commit hash. Specifically, it tags using this Python code:
  
        def get_podman_tags_for_repo(repo_path):
            prev_dir = os.getcwd()
            os.chdir(repo_path)
            tags = [
                os.popen('git rev-parse --short HEAD').read(),
                ]

            version_output = os.popen(
                "git describe --tags --exact-match --match 'v*.*.*' 2>/dev/null").read()
            if version_output:
                version_parts = version_output.split('.')
                tags.extend(
                    (
                        version_output,
                        f'{version_parts[0]}.{version_parts[1]}',
                        version_parts[0])
                    )

            os.chdir(prev_dir)
            return tags


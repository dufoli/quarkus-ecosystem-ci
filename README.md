# quarkus-ecosystem-ci
Repository used for orchestrating cross-CI builds of extensions part of Quarkus ecosystem/universe.

# What it's all about

It's in everyone's interest to know if participants in the [Quarkus Platform](https://github.com/quarkusio/quarkus-platform) build properly 
against the latest Quarkus, since Quarkus moves very fast.
At the same time, due to resource constraints it is not possible for all platform members to run their tests in the Quarkus organization.
In that vein Quarkus has come up with a solution that allows both Quarkus members and Quarkus Platform participants to know each day whether their extension
works with the latest Quarkus or not, while sharing the CI load amongst platform participants.
Furthermore, the solution aims to be as frictionless as possible for Quarkus Platform participants while also not requiring said participants to trust 
Quarkus with their github tokens. 

This repository contains the metadata that needs to be setup for a platform participant to have their CI automatically triggered and post the result on a Quarkus issue.

# What do I need to do as a Quarkus Platform participant  

After your extension has been added to the Quarkus Platform, you will additionally need to do the following:

* Open an issue on the Quarkus repo, where updates will be posted (the CI job will close or open the issue depending on the CI outcome). 
An example of such an issue is can be found [here](https://github.com/quarkusio/quarkus/issues/8562). 

Note that in one of the following steps you will need to configure a Github token for **this** user in **your** extension repository. 
The user can be a regular user (for example the author of the extension) or a bot.   

* Open a Pull Request to this repository where you add a directory for your extension and inside it place a file named `info.yaml`.
The needs to contain the following information:

```yaml
url: https://github.com/someorg/my-extension # Github repo of your extension 
issues:
  repo: quarkusio/quarkus # this should be left as is when CI updates are going to be reported on the Quarkus repository
  latestCommit: 123456 # this is the number of the issue you created above 
```     

An example of the above configuration can be found [here](https://github.com/quarkusio/quarkus-ecosystem-ci/blob/419a6c18312ac26ab0213ae1bf0ee6d38a550f4e/qpid/info.yaml).

* Add a secret named `ECOSYSTEM_CI_TOKEN` in your repository that contains the token of the user that opened the issue mentioned in the first step. This token will be used
in order to open and close the issue from the CI action.

* Add the following files to configure the Quarkus CI in Github Actions for your repository:

`.github/workflows/quarkus-master.yaml` (this file contains the actual Github Actions declaration)

```yaml
name: "Quarkus ecosystem CI"
on:
  watch:
    types: [started]

# For this CI to work, ECOSYSTEM_CI_TOKEN needs to contain a GitHub with rights to close the Quarkus issue that the user/bot has opened,
 # while 'ECOSYSTEM_CI_REPO_PATH' needs to be set to the corresponding path in the 'quarkusio/quarkus-ecosystem-ci' repository

env:
  ECOSYSTEM_CI_REPO: quarkusio/quarkus-ecosystem-ci
  ECOSYSTEM_CI_REPO_FILE: context.yaml
  JAVA_VERSION: 11

  #########################
  # Repo specific setting #
  #########################

  ECOSYSTEM_CI_REPO_PATH: FIXME # TODO: this needs to be set to the directory name added in 'quarkusio/quarkus-ecosystem-ci' 

jobs:
  build:
    name: "Build against Quarkus from master"
    runs-on: ubuntu-latest
    if: github.actor == 'quarkusbot'

    steps:
      - name: Install yq
        run: sudo add-apt-repository ppa:rmescandon/yq && sudo apt update && sudo apt install yq -y

      - name: Set up Java
        uses: actions/setup-java@v1
        with:
          java-version: ${{ env.JAVA_VERSION }}

      - name: Checkout repo
        uses: actions/checkout@v2
        with:
          path: current-repo
          ref: master

      - name: Checkout Ecosystem
        uses: actions/checkout@v2
        with:
          repository: ${{ env.ECOSYSTEM_CI_REPO }}
          ref: master
          path: ecosystem-ci

      - name: Setup and Run Tests
        run: |
          cd ecosystem-ci/${ECOSYSTEM_CI_REPO_PATH}
          ISSUE_NUM=$(yq r ${ECOSYSTEM_CI_REPO_FILE} issues.latestCommit)
          ISSUE_REPO=$(yq r ${ECOSYSTEM_CI_REPO_FILE} issues.repo)
          QUARKUS_VERSION=$(yq r ${ECOSYSTEM_CI_REPO_FILE} quarkus.version)
          cd - > /dev/null

          # perform actual test run
          cd current-repo
          if QUARKUS_VERSION=${QUARKUS_VERSION} .github/test ; then
            echo "Tests succeded"
            TEST_STATUS="success"
          else
            echo "Tests failed"
            TEST_STATUS="failure"
          fi

          echo "Attempting to report results"

          sudo apt-get update -o Dir::Etc::sourcelist="sources.list" \
            -o Dir::Etc::sourceparts="-" -o APT::Get::List-Cleanup="0"
          sudo apt-get install -y gnupg2 gnupg-agent
          echo "Installing SDKMAN"
          curl -s "https://get.sdkman.io" | bash
          source ~/.sdkman/bin/sdkman-init.sh
          sdk install jbang 0.21.0

          jbang .github/report.java "${{ secrets.ECOSYSTEM_CI_TOKEN }}" "${TEST_STATUS}" "${ISSUE_REPO}" "${ISSUE_NUM}" "${GITHUB_REPOSITORY}"

          echo "Report completed"

          if [[ ${TEST_STATUS} != "success" ]]; then
            exit 1
          fi
```         

`.github/mvn-settings.xml` (needed to enable Quarkus snapshots)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings>
	<profiles>
		<profile>
			<id>snapshots</id>
			<activation>
				<activeByDefault>true</activeByDefault>
			</activation>
			<repositories>
				<repository>
					<id>snapshots-repo</id>
					<url>https://oss.sonatype.org/content/repositories/snapshots</url>
					<releases>
						<enabled>false</enabled>
					</releases>
					<snapshots>
						<enabled>true</enabled>
					</snapshots>
				</repository>
			</repositories>
			<pluginRepositories>
				<pluginRepository>
					<id>snapshots-repo</id>
					<url>https://oss.sonatype.org/content/repositories/snapshots</url>
					<releases>
						<enabled>false</enabled>
					</releases>
					<snapshots>
						<enabled>true</enabled>
					</snapshots>
				</pluginRepository>
			</pluginRepositories>
		</profile>
	</profiles>
</settings>
```

`.github/report.java` (a [jbang](https://github.com/maxandersen/jbang) script that updates the issue status mentioned in step 1)

```java
//usr/bin/env jbang "$0" "$@" ; exit $?

//DEPS org.kohsuke:github-api:1.101

import org.kohsuke.github.*;
import java.io.File;
import java.io.IOException;
import java.util.concurrent.TimeUnit;

class Report {

	public static void main(String[] args) throws IOException {
		final String token = args[0];
		final String status = args[1];
		final String issueRepo = args[2];
		final Integer issueNumber = Integer.valueOf(args[3]);
		final String thisRepo = args[4];

		final boolean succeed = "success".equalsIgnoreCase(status);
		if ("cancelled".equalsIgnoreCase(status)) {
			System.out.println("Job status is `cancelled` - exiting");
			System.exit(0);
		}

		System.out.println(String.format("The CI build had status %s.", status));

		final GitHub github = new GitHubBuilder().withOAuthToken(token).build();
		final GHRepository repository = github.getRepository(issueRepo);

		final GHIssue issue = repository.getIssue(issueNumber);
		if (issue == null) {
			System.out.println(String.format("Unable to find the issue %s in project %s", issueNumber, issueRepo));
			System.exit(-1);
		} else {
			System.out.println(String.format("Report issue found: %s - %s", issue.getTitle(), issue.getHtmlUrl().toString()));
			System.out.println(String.format("The issue is currently %s", issue.getState().toString()));
		}

		if (succeed) {
			if (issue != null  && isOpen(issue)) {
				// close issue with a comment
				final GHIssueComment comment = issue.comment(String.format("Build fixed:\n* Link to latest CI run: https://github.com/%s/actions", thisRepo));
				issue.close();
				System.out.println(String.format("Comment added on issue %s - %s, the issue has also been closed", issue.getHtmlUrl().toString(), comment.getHtmlUrl().toString()));
			} else {
				System.out.println("Nothing to do - the build passed and the issue is already closed");
			}
		} else  {
			if (isOpen(issue)) {
				final GHIssueComment comment = issue.comment(String.format("The build is still failing:\n* Link to latest CI run: https://github.com/%s/actions", thisRepo));
				System.out.println(String.format("Comment added on issue %s - %s", issue.getHtmlUrl().toString(), comment.getHtmlUrl().toString()));
			} else {
				issue.reopen();
				final GHIssueComment comment = issue.comment(String.format("Unfortunately, the build failed:\n* Link to latest CI run: https://github.com/%s/actions", thisRepo));
				System.out.println(String.format("Comment added on issue %s - %s, the issue has been re-opened", issue.getHtmlUrl().toString(), comment.getHtmlUrl().toString()));
			}
		}

	}

	private static boolean isOpen(GHIssue issue) {
		return (issue.getState() == GHIssueState.OPEN);
	}
}
```

Finally, add `.github/test` script where you need to add the operations necessary to launch the tests your extensions contains.

```shell script
#!/usr/bin/env bash
set -e

# for example, this could be enough
mvn --settings .github/mvn-settings.xml clean install -Dquarkus.version=${QUARKUS_VERSION} -Dnative -Dquarkus.native.container-build=true
```

An example project containing all these files (and which has been tested with the whole process) can be found [here](https://github.com/geoand/quarkus-qpid-jms/tree/dc4933338a4f1b8588e6d069575a7769a5b22608/.github)

## How come this works?

The "trick" (more like a hack actually) is that Quarkus Platform participant's Github Actions are triggered when the Quarkus Ecosystem CI stars the extension repository.
Furthermore, before starring the repository, some context information is written to this repository which is then meant to be read in the triggered Github Action.
This way this Quarkus Github Action does not need to hold any secrets for the participants.  

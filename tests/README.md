# Opendevstack Quickstarter tests

All tests for quickstarters follow the same schema:

1. a single test file, named `jenkinsfile_test.go` - which provisions, and runs a generated quickstarter
1. a directory `golden` housing golden records for the jenkins stages, and sonarqube to verify the actual jenkins responses against
1. a quickstarter test always provisions the component into an ODS namespace called `unitt` - which is created by the tests in [ods-core](https://github.com/opendevstack/ods-core/tree/master/tests). So those must be run first, thru `make test` in `ods-core/tests`

Lets look at a single test in detail - in this case the one for [spring boot](be-java-springboot/jenkinsfile_test.go)

1. Create the bitbucket repository (and bitbucket project if needed)
```
	// cleanup and create bb resources for this test
	utils.CleanupAndCreateBitbucketProjectAndRepo(
		quickstarterName, componentId)
```
Here, the key is the `cd_user` configured in `ods-configuration/ods-core.env` must have rights to create and manage a bitbucket project

2. Start the provisioning of a quickstarter thru the webhook proxy

```
	// run provision job for quickstarter in project's cd jenkins
	stages, err := utils.RunJenkinsFile(
		"ods-quickstarters",
		values["ODS_BITBUCKET_PROJECT"],
		values["ODS_GIT_REF"],
		coreUtils.PROJECT_NAME,
		fmt.Sprintf("%s/Jenkinsfile", quickstarterName),
		coreUtils.PROJECT_NAME_CD,
		coreUtils.EnvPair{
			Name:  "COMPONENT_ID",
			Value: componentId,
		},
		coreUtils.EnvPair{
			Name:  "GIT_URL_HTTP",
			Value: fmt.Sprintf("%s/%s/%s.git", values["REPO_BASE"], coreUtils.PROJECT_NAME, componentId),
		},
	)
```

3. Verify expected stages from the `golden` provisioning record against the response `stages`

4. Trigger a jenkins build run instance (for the provisioned component)

```
	// run master build of provisioned quickstarter in project's cd jenkins
	stages, err = utils.RunJenkinsFile(
		componentId,
		coreUtils.PROJECT_NAME,
		"master",
		coreUtils.PROJECT_NAME,
		"Jenkinsfile",
		coreUtils.PROJECT_NAME_CD,
		coreUtils.EnvPair{
			Name:  "COMPONENT_ID",
			Value: componentId,
		},
	)
```

5. Verify expected stages from the `golden` run / build record against the response `stages`

5. Verify the created OCP artifacts, and if pods / deployments are available

```
	resourcesInTest := coreUtils.Resources{
		Namespace:         coreUtils.PROJECT_NAME_DEV,
		ImageTags:         []coreUtils.ImageTag{{Name: componentId, Tag: "latest"}},
		BuildConfigs:      []string{componentId},
		DeploymentConfigs: []string{componentId},
		Services:          []string{componentId},
		ImageStreams:      []string{componentId},
	}
```

All necessary utils, except for [scripts](scripts) are housed in [ods-core/tests](https://github.com/opendevstack/ods-core/tree/master/tests/utils) 
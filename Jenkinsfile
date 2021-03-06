#! groovy
library 'pipeline-library'
def nodeVersion = '8.9.0'

timestamps() {
	node('(osx || linux) && git && npm-publish && curl') {
		def packageVersion = ''
		def isPR = false

		stage('Checkout') {
			// checkout scm
			// Hack for JENKINS-37658 - see https://support.cloudbees.com/hc/en-us/articles/226122247-How-to-Customize-Checkout-for-Pipeline-Multibranch
			checkout([
				$class: 'GitSCM',
				branches: scm.branches,
				extensions: scm.extensions + [[$class: 'CleanBeforeCheckout'], [$class: 'CloneOption', honorRefspec: true, noTags: true, reference: '', shallow: true, depth: 30, timeout: 30]],
				userRemoteConfigs: scm.userRemoteConfigs
			])

			isPR = env.BRANCH_NAME.startsWith('PR-')
			packageVersion = jsonParse(readFile('package.json'))['version']
			currentBuild.displayName = "#${packageVersion}-${currentBuild.number}"
		}

		nodejs(nodeJSInstallationName: "node ${nodeVersion}") {
			ansiColor('xterm') {
				timeout(55) {
					stage('Build') {
						// Install yarn if not installed
						if (sh(returnStatus: true, script: 'which yarn') != 0) {
							// TODO Install using the curl script via chef before-hand?
							// sh 'curl -o- -L https://yarnpkg.com/install.sh | bash'
							sh 'npm install -g yarn'
						}
						sh 'yarn install'
						if (sh(returnStatus: true, script: 'which ti') != 0) {
							// Install titanium
							sh 'yarn global add titanium'
						}
						if (sh(returnStatus: true, script: 'ti config sdk.selected') != 0) {
							// Install titanium SDK and select it
							sh 'ti sdk install -d'
						}
						try {
							withEnv(["PATH+ALLOY=${pwd()}/bin"]) {
								sh 'yarn test'
							}
						} finally {
							junit 'TEST-*.xml'
						}
						fingerprint 'package.json'
						// Don't tag PRs
						if (!isPR) {
							pushGitTag(name: packageVersion, message: "See ${env.BUILD_URL} for more information.", force: true)
						}
					} // stage
				} // timeout

				stage('Security') {
					// Clean up and install only production dependencies
					sh 'rm -rf node_modules/'
					sh 'yarn install --production'

					// Scan for NSP and RetireJS warnings
					sh 'yarn global add nsp'
					sh 'nsp check --output summary --warn-only'

					sh 'yarn global add retire'
					sh 'retire --exitwith 0'

					step([$class: 'WarningsPublisher', canComputeNew: false, canResolveRelativePaths: false, consoleParsers: [[parserName: 'Node Security Project Vulnerabilities'], [parserName: 'RetireJS']], defaultEncoding: '', excludePattern: '', healthy: '', includePattern: '', messagesPattern: '', unHealthy: ''])
				} // stage

				stage('Publish') {
					if (!isPR) {
						sh 'npm publish'
						// Trigger appc-cli-wrapper job
						build job: 'appc-cli-wrapper', wait: false
					}
				}

				stage('JIRA') {
					if (!isPR) {
						def versionName = "Alloy ${packageVersion}"
						def projectKey = 'ALOY'
						def issueKeys = jiraIssueSelector(issueSelector: [$class: 'DefaultIssueSelector'])

						// Comment on the affected tickets with build info
						step([
							$class: 'hudson.plugins.jira.JiraIssueUpdater',
							issueSelector: [$class: 'hudson.plugins.jira.selector.DefaultIssueSelector'],
							scm: scm
						])

						// Create the version we need if it doesn't exist...
						step([
							$class: 'hudson.plugins.jira.JiraVersionCreatorBuilder',
							jiraVersion: versionName,
							jiraProjectKey: projectKey
						])

						// Should append the new version to the ticket's fixVersion field
						def fixVersion = [name: versionName]
						for (i = 0; i < issueKeys.size(); i++) {
							def result = jiraGetIssue(idOrKey: issueKeys[i])
							def fixVersions = result.data.fields.fixVersions << fixVersion
							def testIssue = [fields: [fixVersions: fixVersions]]
							jiraEditIssue(idOrKey: issueKeys[i], issue: testIssue)
						}

						// Should release the version
						step([$class: 'JiraReleaseVersionUpdaterBuilder', jiraProjectKey: projectKey, jiraRelease: versionName])
					}
				}
			} // ansiColor
		} //nodejs
	} // node
} // timestamps

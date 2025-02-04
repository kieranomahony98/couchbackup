#!groovy
// Copyright © 2017, 2023 IBM Corp. All rights reserved.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
// http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

def getEnvForSuite(suiteName, version) {

  def envVars = []

  // Add test suite specific environment variables
  switch(suiteName) {
    case 'test':
      envVars.add("COUCHBACKUP_MOCK_SERVER_PORT=${7700 + version.toInteger()}")
      break
    case 'toxytests/toxy':
      envVars.add("TEST_TIMEOUT_MULTIPLIER=50")
      envVars.add("COUCHBACKUP_MOCK_SERVER_PORT=${7800 + version.toInteger()}")
      break
      case 'test-iam':
        envVars.add("CLOUDANT_IAM_TOKEN_URL=${SDKS_TEST_IAM_URL}")
        envVars.add("COUCHBACKUP_MOCK_SERVER_PORT=${7900 + version.toInteger()}")
        break
    default:
      error("Unknown test suite environment ${suiteName}")
  }

  return envVars
}



// NB these registry URLs must have trailing slashes

// url of registry for public uploads
def getRegistryPublic() {
    return "https://registry.npmjs.org/"
}

// url of registry for artifactory down
def getRegistryArtifactoryDown() {
    return "${Artifactory.server('taas-artifactory').getUrl()}/api/npm/cloudant-sdks-npm-virtual/"
}

def noScheme(str) {
    return str.substring(str.indexOf(':') + 1)
}

def withNpmEnv(registry, closure) {
  withEnv(['NPMRC_REGISTRY=' + noScheme(registry),
           'NPM_CONFIG_REGISTRY=' + registry,
           'NPM_CONFIG_USERCONFIG=.npmrc-jenkins']) {
    closure()
  }
}

def nodeYaml(version) {
    return """\
      |    - name: node${version}
      |      image: ${globals.ARTIFACTORY_DOCKER_REPO_VIRTUAL}/node:${version}
      |      command: ['sh', '-c', 'sleep 99d']
      |      imagePullPolicy: Always
      |      resources:
      |        requests:
      |          memory: "2Gi"
      |          cpu: "650m"
      |        limits:
      |          memory: "4Gi"
      |          cpu: "4"
      |      securityContext:
      |        runAsUser: 1000""".stripIndent()
}

def agentYaml() {
  return """\
    |apiVersion: v1
    |kind: Pod
    |metadata:
    |  name: couchbackup
    |spec:
    |  imagePullSecrets:
    |    - name: artifactory
    |  containers:
    |    - name: jnlp
    |      image: ${globals.ARTIFACTORY_DOCKER_REPO_VIRTUAL}/sdks-full-agent
    |      imagePullPolicy: Always
    |      resources:
    |        requests:
    |          memory: "2Gi"
    |          cpu: "650m"
    |        limits:
    |          memory: "4Gi"
    |          cpu: "4"
    ${nodeYaml(16)}
    ${nodeYaml(20)}
    |restartPolicy: Never""".stripMargin('|')
}



def runTest(version, filter=null, testSuite='test') {
  if (filter == null) {
    if (env.TEST_FILTER == null) {
      // The set of default tests includes unit and integration tests, but
      // not ones tagged #slower, #slowest.
      filter = '-i -g \'#slowe\''
    } else {
      filter = env.TEST_FILTER
    }
  }
  def testReportPath = "${testSuite}-${version}-results.xml"
  // Run tests using creds
  withEnv(getEnvForSuite("${testSuite}", version)) {
    withCredentials([usernamePassword(credentialsId: 'testServerLegacy', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASSWORD'),
                      usernamePassword(credentialsId: 'artifactory', usernameVariable: 'ARTIFACTORY_USER', passwordVariable: 'ARTIFACTORY_PW'),
                      string(credentialsId: 'testServerIamApiKey', variable: "${(testSuite == 'test-iam') ? 'COUCHBACKUP_TEST_IAM_API_KEY' : 'IAM_API_KEY'}")]) {
      try {
        // For the IAM tests we want to run the normal 'test' suite, but we
        // want to keep the report named 'test-iam'
        def testRun = (testSuite != 'test-iam') ? testSuite : 'test'
        def dbPassword = java.net.URLEncoder.encode(DB_PASSWORD, "UTF-8")
        
        // Actions:
        //  3. Install mocha-jenkins-reporter so that we can get junit style output
        //  4. Fetch database compare tool for CI tests
        //  5. Run tests using filter
        withCredentials([usernamePassword(usernameVariable: 'NPMRC_USER', passwordVariable: 'NPMRC_TOKEN', credentialsId: 'artifactory')]) {
          withEnv(['NPMRC_EMAIL=' + env.NPMRC_USER]) {
            withNpmEnv(registryArtifactoryDown) {
              sh """
                set +x
                export COUCH_BACKEND_URL="https://\${DB_USER}:${dbPassword}@\${SDKS_TEST_SERVER_HOST}"
                export COUCH_URL="${(testSuite == 'toxytests/toxy') ? 'http://localhost:3000' : ((testSuite == 'test-iam') ? '${SDKS_TEST_SERVER_URL}' : '${COUCH_BACKEND_URL}')}"
                set -x
                ./node_modules/mocha/bin/mocha.js --reporter mocha-jenkins-reporter --reporter-options junit_report_path=${testReportPath},junit_report_stack=true,junit_report_name=${testSuite} ${filter} ${testRun}
              """
            }
          }
        }
      } finally {
        junit "**/${testReportPath}"
      }
    }
  }
}

pipeline {
  agent {
    kubernetes {
      yaml "${agentYaml()}"
    }
  }
  stages {
    stage('Build') {
      steps {
        withCredentials([usernamePassword(usernameVariable: 'NPMRC_USER', passwordVariable: 'NPMRC_TOKEN', credentialsId: 'artifactory')]) {
          withEnv(['NPMRC_EMAIL=' + env.NPMRC_USER]) {
            withNpmEnv(registryArtifactoryDown) {
              sh 'npm ci'
              sh 'npm install mocha-jenkins-reporter --no-save'
            }
          }
        }
      }
    }
    stage('QA') {
      parallel {
        // Stages that run on LTS version from full agent default container
        stage('Lint') {
          steps {
            sh 'npm run lint'
          }
        }
        stage('Node LTS') {
          steps {
            script{
              runTest('18')
            }
          }
        }
        stage('IAM Node LTS') {
          steps {
            script{
              runTest('18', '-i -g \'#unit|#slowe\'', 'test-iam')
            }
          }
        }
        stage('Network Node LTS') {
          when {
            beforeAgent true
            environment name: 'RUN_TOXY_TESTS', value: 'true'
          }
          steps {
            script{
              runTest('18', '', 'toxytests/toxy')
            }
          }
        }
        // Stages that run for other version node containers in the pod
        stage('Node 16x') {
          steps {
            container('node16') {
              script{
                runTest('16')
              }
            }
          }
        }
        stage('Node 20x') {
          when {
            beforeAgent true
            // build this stage for branches working on resolving Node 20 problems
            // otherwise skip it for now
            branch '583-*'
          }
          steps {
            container('node20') {
              script{
                runTest('20')
              }
            }
          }
        }
      }
    }
    stage('SonarQube analysis') {
      when {
        beforeAgent true
        allOf {
          expression { env.BRANCH_NAME }
          not {
            expression { env.BRANCH_NAME.startsWith('dependabot/') }
          }
        }
      }
      steps {
        script {
          def scannerHome = tool 'SonarQubeScanner';
          withSonarQubeEnv(installationName: 'SonarQubeServer') {
            sh "${scannerHome}/bin/sonar-scanner -X -Dsonar.qualitygate.wait=true -Dsonar.projectKey=couchbackup -Dsonar.branch.name=${env.BRANCH_NAME}"
          }
        }
      }
    }
    // Publish the primary branch
    stage('Publish') {
      when {
        beforeAgent true
        branch 'main'
      }
      steps {
        script {
          def v = com.ibm.cloudant.integrations.VersionHelper.readVersion(this, 'package.json')
          String version = v.version
          boolean isReleaseVersion = v.isReleaseVersion

          // Upload using the NPM creds
          withCredentials([string(credentialsId: 'npm-mail', variable: 'NPMRC_EMAIL'),
                          usernamePassword(credentialsId: 'npm-creds', passwordVariable: 'NPMRC_TOKEN', usernameVariable: 'NPMRC_USER')]) {
            // Actions:
            // 1. add the build ID to any snapshot version for uniqueness
            // 2. publish the build to NPM adding a snapshot tag if pre-release
            sh "${isReleaseVersion ? '' : ('npm version --no-git-tag-version ' + version + '.' + env.BUILD_ID)}"
            withNpmEnv(registryPublic) {
              sh "npm publish ${isReleaseVersion ? '' : '--tag snapshot'}"
            }
          }
          // Run the gitTagAndPublish which tags/publishes to github for release builds
          gitTagAndPublish {
              versionFile='package.json'
              releaseApiUrl='https://api.github.com/repos/IBM/couchbackup/releases'
          }
        }
      }
    }
  }
}

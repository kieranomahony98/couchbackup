#!groovy
// Copyright © 2017, 2019 IBM Corp. All rights reserved.
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

def getEnvForSuite(suiteName) {
  // Base environment variables
  def envVars = [
    "COUCH_BACKEND_URL=https://" + DB_USER + ":" + DB_PASSWORD+ "@" + SDKS_TEST_SERVER_HOST,
    "DBCOMPARE_NAME=DatabaseCompare",
    "DBCOMPARE_VERSION=1.0.1",
    "NVM_DIR=${env.HOME}/.nvm"
  ]

  // Add test suite specific environment variables
  switch(suiteName) {
    case 'test':
      envVars.add("COUCH_URL=https://" + DB_USER + ":" + DB_PASSWORD + "@" + SDKS_TEST_SERVER_HOST)
      break
    case 'toxytests/toxy':
      envVars.add("COUCH_URL=http://localhost:3000") // proxy
      envVars.add("TEST_TIMEOUT_MULTIPLIER=50")
      break
      case 'test-iam':
        envVars.add("COUCH_URL=${SDKS_TEST_SERVER_URL}")
        envVars.add("COUCHBACKUP_TEST_IAM_API_KEY=" + IAM_API_KEY)
        envVars.add("CLOUDANT_IAM_TOKEN_URL=${SDKS_TEST_IAM_URL}")
        break
    default:
      error("Unknown test suite environment ${suiteName}")
  }

  return envVars
}

def setupNodeAndTest(version, filter='', testSuite='test') {
  node {
    // Install NVM
    sh 'wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.2/install.sh | bash'
    // Unstash the built content
    unstash name: 'built'

    withEnv(["NVM_DIR=${env.HOME}/.nvm"]) {
      if (testSuite == 'lint') {
        sh """
          [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
          nvm install ${version}
          nvm use ${version}
          npm run lint
        """
      } else {
        // Run tests using creds
        withCredentials([
                usernamePassword(credentialsId: 'testServerLegacy',
                        usernameVariable: 'DB_USER',
                        passwordVariable: 'DB_PASSWORD'),
                usernamePassword(credentialsId: 'artifactory',
                        usernameVariable: 'ARTIFACTORY_USER',
                        passwordVariable: 'ARTIFACTORY_PW'),
                string(credentialsId: 'testServerIamApiKey',
                        variable: 'IAM_API_KEY'),
                string(credentialsId: 'dbCompareName',
                        variable: 'DBCOMPARE_NAME'),
                string(credentialsId: 'dbCompareVersion',
                        variable: 'DBCOMPARE_VERSION')]) {
          withEnv(getEnvForSuite("${testSuite}")) {
            try {
              // For the IAM tests we want to run the normal 'test' suite, but we
              // want to keep the report named 'test-iam'
              def testRun = (testSuite != 'test-iam') ? testSuite : 'test'

              // Actions:
              //  1. Load NVM
              //  2. Install/use required Node.js version
              //  3. Install mocha-jenkins-reporter so that we can get junit style output
              //  4. Fetch database compare tool for CI tests
              //  5. Run tests using filter
              FILENAME = '$DBCOMPARE_NAME-$DBCOMPARE_VERSION'
              sh '''
                [ -s "${NVM_DIR}/nvm.sh" ] && . "${NVM_DIR}/nvm.sh"
                nvm install ${version}
                nvm use ${version}
                npm install mocha-jenkins-reporter --save-dev
                curl -O -u ${ARTIFACTORY_USER}:${ARTIFACTORY_PW} https://na.artifactory.swg-devops.com/artifactory/cloudant-sdks-maven-local/com/ibm/cloudant/${DBCOMPARE_NAME}/${DBCOMPARE_VERSION}/${FILENAME}.zip
                unzip ${FILENAME}.zip
                ./node_modules/mocha/bin/mocha --reporter mocha-jenkins-reporter --reporter-options junit_report_path=./test/test-results.xml,junit_report_stack=true,junit_report_name=${testSuite} ${filter} ${testRun}
              '''
            } finally {
              junit '**/*test-results.xml'
            }
          }
        }
      }
    }
  }
}

stage('Build') {
  // Checkout, build
  node {
    checkout scm
    sh 'npm ci'
    stash name: 'built', useDefaultExcludes: false
  }
}

stage('QA') {
  // Allow a supplied a test filter, but provide a reasonable default.
  String filter;
  if (env.TEST_FILTER == null) {
    // The set of default tests includes unit and integration tests, but
    // not ones tagged #slower, #slowest.
    filter = '-i -g \'#slowe\''
  } else {
    filter = env.TEST_FILTER
  }

  def axes = [
    Node12x:{ setupNodeAndTest('12', filter) }, // 12.x Maintenance LTS
    Node14x:{ setupNodeAndTest('14', filter) }, // 14.x Active LTS
    Node:{ setupNodeAndTest('16', filter) }, // Current
    // Test IAM on the current Node.js version. Filter out unit tests and the
    // slowest integration tests.
    Iam: { setupNodeAndTest('16', '-i -g \'#unit|#slowe\'', 'test-iam') },
    Lint: { setupNodeAndTest('12', '', 'lint') }
  ]
  // Add unreliable network tests if specified
  if (env.RUN_TOXY_TESTS && env.RUN_TOXY_TESTS.toBoolean()) {
    axes.Network = { setupNodeAndTest('node', '', 'toxytests/toxy') }
  }
  // Run the required axes in parallel
  parallel(axes)
}

// Publish the master branch
stage('Publish') {
  if (env.BRANCH_NAME == "master") {
    node {
      unstash 'built'

      def v = com.ibm.cloudant.integrations.VersionHelper.readVersion(this, 'package.json')
      String version = v.version
      boolean isReleaseVersion = v.isReleaseVersion

      // Upload using the NPM creds
      withCredentials([string(credentialsId: 'npm-mail', variable: 'NPM_EMAIL'),
                       usernamePassword(credentialsId: 'npm-creds', passwordVariable: 'NPM_TOKEN', usernameVariable: 'NPM_USER')]) {
        // Actions:
        // 1. create .npmrc file for publishing
        // 2. add the build ID to any snapshot version for uniqueness
        // 3. publish the build to NPM adding a snapshot tag if pre-release
        sh "echo '//registry.npmjs.org/:_authToken=\\\${NPM_TOKEN}' > .npmrc"
        sh '''
          ${isReleaseVersion ? '' : ('npm version --no-git-tag-version ' + version + '.' + BUILD_ID)}
          npm publish ${isReleaseVersion ? '' : '--tag snapshot'}
        '''
      }
    }
  }

  // Run the gitTagAndPublish which tags/publishes to github for release builds
  gitTagAndPublish {
      versionFile='package.json'
      releaseApiUrl='https://api.github.com/repos/cloudant/couchbackup/releases'
  }
}

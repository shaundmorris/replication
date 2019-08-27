//"Jenkins Pipeline is a suite of plugins which supports implementing and integrating continuous delivery pipelines into Jenkins. Pipeline provides an extensible set of tools for modeling delivery pipelines "as code" via the Pipeline DSL."
//More information can be found on the Jenkins Documentation page https://jenkins.io/doc/
pipeline {
    agent {
        node {
            label 'linux-small'
            customWorkspace "/jenkins/workspace/${env.JOB_NAME}/${env.BUILD_NUMBER}"
        }
    }
    parameters {
        booleanParam(name: 'RELEASE', defaultValue: false, description: 'Perform Release?')
        string(name: 'RELEASE_VERSION', defaultValue: 'NA', description: 'The version to release. An NA value will release the current version')
        string(name: 'RELEASE_TAG', defaultValue: 'NA', description: 'The release tag for this version. An NA value will result in replication-RELEASE_VERSION')
        string(name: 'NEXT_VERSION', defaultValue: 'NA', description: 'The next development version. An NA value will increment the patch version')
    }
    options {
        buildDiscarder(logRotator(numToKeepStr: '25'))
        disableConcurrentBuilds()
        timestamps()
    }
    triggers {
        /*
          Restrict nightly builds to master branch, all others will be built on change only.
          Note: The BRANCH_NAME will only work with a multi-branch job using the github-branch-source
        */
        cron(env.BRANCH_NAME == "master" ? "H H(21-23) * * *" : "")
    }
    environment {
        LINUX_MVN_RANDOM = '-Djava.security.egd=file:/dev/./urandom'
        COVERAGE_EXCLUSIONS = '**/test/**/*,**/itests/**/*,**/*Test*,**/sdk/**/*,**/*.js,**/node_modules/**/*,**/jaxb/**/*,**/wsdl/**/*,**/nces/sws/**/*,**/*.adoc,**/*.txt,**/*.xml'
        DISABLE_DOWNLOAD_PROGRESS_OPTS = '-Dorg.slf4j.simpleLogger.log.org.apache.maven.cli.transfer.Slf4jMavenTransferListener=warn '
    }
    stages {
        // The incremental build will be triggered only for PRs. It will build the differences between the PR and the target branch
        stage('Incremental Build') {
            when {
                allOf {
                    expression { env.CHANGE_ID != null }
                    expression { env.CHANGE_TARGET != null }
                }
            }
            parallel {
                stage('Linux') {
                    steps {
                        // TODO: Maven downgraded to work around a linux build issue. Falling back to system java to work around a linux build issue. re-investigate upgrading later
                        withMaven(maven: 'Maven 3.5.3', jdk: 'jdk8-latest', globalMavenSettingsConfig: 'default-global-settings', mavenSettingsConfig: 'codice-maven-settings', mavenOpts: '${LINUX_MVN_RANDOM}') {
                            sh 'mvn install -B -DskipStatic=true -DskipTests=true $DISABLE_DOWNLOAD_PROGRESS_OPTS'
                            sh 'mvn install -B -Dgib.enabled=true -Dgib.referenceBranch=/refs/remotes/origin/$CHANGE_TARGET $DISABLE_DOWNLOAD_PROGRESS_OPTS'
                        }
                    }
                }
                //Commenting this out for now as the windows build is currently failing due to external environment issues and we may no longer need windows builds going forward
                //stage ('Windows') {
                //    agent { label 'server-2016-small' }
                //    steps {
                //        withMaven(maven: 'Maven 3.5.3', jdk: 'jdk8-latest', globalMavenSettingsConfig: 'default-global-settings', mavenSettingsConfig: 'codice-maven-settings') {
                //             bat 'mvn install -B -DskipStatic=true -DskipTests=true  %DISABLE_DOWNLOAD_PROGRESS_OPTS%'
                //             bat 'mvn install -B -Dgib.enabled=true -Dgib.referenceBranch=/refs/remotes/origin/%CHANGE_TARGET% %DISABLE_DOWNLOAD_PROGRESS_OPTS%'
                //        }
                //    }
                //}
            }
        }
        stage('Full Build') {
            when { expression { env.CHANGE_ID == null } }
            parallel {
                stage('Linux') {
                    steps {
                        // TODO: Maven downgraded to work around a linux build issue. Falling back to system java to work around a linux build issue. re-investigate upgrading later
                        withMaven(maven: 'Maven 3.5.3', jdk: 'jdk8-latest', globalMavenSettingsConfig: 'default-global-settings', mavenSettingsConfig: 'codice-maven-settings', mavenOpts: '${LINUX_MVN_RANDOM}') {
                            script {
                                if (params.RELEASE == true) {
                                    sh "mvn -B -Dtag=${env.RELEASE_TAG} -DreleaseVersion=${env.RELEASE_VERSION} -DdevelopmentVersion=${env.NEXT_VERSION} release:prepare"
                                    env.RELEASE_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                                } else {
                                    sh 'mvn clean install -B $DISABLE_DOWNLOAD_PROGRESS_OPTS'
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('Quality Analysis') {
            parallel {
                // Sonar stage only runs against master
                //Need to check SonarQube Token
//                stage ('SonarCloud') {
//                    steps {
//                        //Run SonarCloud on all branches as it will take care of removing analysis after 30 days
//                        //Need to check the token
//                        withMaven(maven: 'M35', jdk: 'jdk8-latest', globalMavenSettingsConfig: 'default-global-settings', mavenSettingsConfig: 'codice-maven-settings', mavenOpts: '${LINUX_MVN_RANDOM}') {
//                            withCredentials([string(credentialsId: 'SonarQubeGithubToken', variable: 'SONARQUBE_GITHUB_TOKEN'), string(credentialsId: 'cxbot-sonarcloud', variable: 'SONAR_TOKEN')]) {
//                                script {
//                                    sh 'mvn -q -B -Dcheckstyle.skip=true org.jacoco:jacoco-maven-plugin:prepare-agent install sonar:sonar -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=$SONAR_TOKEN  -Dsonar.organization=shaundmorris-github -Dsonar.projectKey=replication:replication -Dsonar.exclusions=${COVERAGE_EXCLUSIONS} $DISABLE_DOWNLOAD_PROGRESS_OPTS'
//
//                                }
//                            }
//                        }
//                    }
//                }
                stage('SonarCloud') {
                    steps {
                        withMaven(maven: 'M35', jdk: 'jdk8-latest', globalMavenSettingsConfig: 'default-global-settings', mavenSettingsConfig: 'codice-maven-settings', mavenOpts: '${LARGE_MVN_OPTS} ${LINUX_MVN_RANDOM}') {
                            withCredentials([string(credentialsId: 'SonarQubeGithubToken', variable: 'SONARQUBE_GITHUB_TOKEN'), string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                                script {
                                    // If this build is not a pull request, run sonar scan. otherwise run incremental scan
                                    if (env.CHANGE_ID == null) {
                                        sh 'mvn -q -B -Dcheckstyle.skip=true org.jacoco:jacoco-maven-plugin:prepare-agent install sonar:sonar -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=$SONAR_TOKEN  -Dsonar.organization=shaundmorris-github -Dsonar.projectKey=replication:replication -Dsonar.exclusions=${COVERAGE_EXCLUSIONS} -pl !$DOCS,!$ITESTS $DISABLE_DOWNLOAD_PROGRESS_OPTS'
                                    } else {
                                        sh 'mvn -q -B -Dcheckstyle.skip=true org.jacoco:jacoco-maven-plugin:prepare-agent install sonar:sonar -Dsonar.github.pullRequest=${CHANGE_ID} -Dsonar.github.oauth=${SONARQUBE_GITHUB_TOKEN}  -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=$SONAR_TOKEN -Dsonar.organization=shaundmorris-github -Dsonar.projectKey=replication:replication -Dsonar.pullrequest.provider=github -Dsonar.exclusions=${COVERAGE_EXCLUSIONS} -pl !$DOCS,!$ITESTS -Dgib.enabled=true -Dgib.referenceBranch=/refs/remotes/origin/$CHANGE_TARGET $DISABLE_DOWNLOAD_PROGRESS_OPTS'
                                    }
                                }
                            }
                        }
                    }
                }
            }
        } 
    }
}
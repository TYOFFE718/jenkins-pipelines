properties(
    [buildDiscarder(
      logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '30', numToKeepStr: '10')
    ),
    [$class: 'RebuildSettings', autoRebuild: false, rebuildDisabled: false],
    parameters([
        choiceParam(name: "TYPE", choices: "snapshot\nreleases", description: "Deployment type"),
        choiceParam(name: "ENVIRONMENT", choices: "development\nproduction\nlocal", description: "AWS Region within which to deploy Infrastructure"),
        string(name: "PIPELINE_BRANCH", defaultValue: "master", description: "")
    ]),

    pipelineTriggers([])
  ])

switch(params.ENVIRONMENT) {
    case "dev":
        ACCOUNT = "dev"
        break
    case "prod":
        ACCOUNT = "pre-prod"
        break
    default:
        ACCOUNT = "prod"
        break
}

// if report name contains HTTP unfriendly character 
// it will be replaced by its corresponding encoded value where % will be replaced with _, 
//i.e. space - _20
reportName = "Documentation and Test report"

def sendSlackMessage(color, message) {
    message = "${message}: 'Running spec tests for <${env.RUN_CHANGES_DISPLAY_URL}|[Recent Changes]> ${env.JOB_NAME} #${env.BUILD_NUMBER}' <${env.BUILD_URL}|(Build Details)>"
    if (message.contains("SUCCESS")) {
        message = "SUCCESS: Spec testing completed <https://jenkins-prod.aws.audiomack.com/job/testing/job/spec/Documentation_20and_20Test_20report/|View Report>"
    }
    if (message.contains("FAILURE")) {
        message = "FAILURE: Build failed ${env.JOB_NAME} #${env.BUILD_NUMBER}' <${env.BUILD_URL}|(Build Details)>"
    }
    slackSend color: color,
              message: message
}

try {
    node {
        timestamps {
            ansiColor('xterm') {
                dir("${env.WORKSPACE}") {
                    deleteDir()
                }
                try {
                    stage ('Checkout source') {
                        dir('pipeline') {
                            git([url: "https://github.com/tyoffeawscloud/spec.git",
                                 branch: params.PIPELINE_BRANCH ?: 'master',
                                 credentialsId: 'aws_github'])
                        }
                    }
                } catch (e) {
                    error "Cannot checkout repo..."
                }

                dir("${env.WORKSPACE}/pipeline/docs") {
                    stage("run build sh") {
                        
                        ["development", "production"].each {
                        envFileContent = "API_CONSUMER_KEY=phpunit\nAPI_CONSUMER_SECRET=xxxxxxxxx\nAPI_TOKEN=xxxxxxxxx\nAPI_TOKEN_SECRET=xxxxxxx\nAPI_ENDPOINT=${params.ENVIRONMENT == "development" ? "https://dev.tyoffecloud.com" : "PRODUCTION"}"
                        writeFile file: "${env.WORKSPACE}/pipeline/docs/.env.${it}", text: envFileContent
                        }

                        sendSlackMessage('#FFFF00', "STARTED")
                        output = sh script: "sh ./build.sh ${params.ENVIRONMENT} build", returnStdout: true
                    }
                    if (output.contains("exited with code 1")) {
                       error "Build has failed..." 
                    }

                    stage("Publish report") {
                        publishHTML([allowMissing: false,
                                     alwaysLinkToLastBuild: false,
                                     keepAll: false,
                                     reportDir: 'output',
                                     reportFiles: 'output.html,report.html',
                                     reportName: reportName,
                                    reportTitles: 'Documentation, Test report'])
                    }
                }
            }
        }
    }
} catch (e) {
    currentBuild.result = "FAILURE"
    error "Build has failed..."
} finally {
    if (currentBuild.result == "FAILURE") {
        sendSlackMessage('#FF0000', "FAILURE")
    } else {
        sendSlackMessage('#00FF00', "SUCCESS")
    }

import groovy.transform.Field
import groovy.json.JsonSlurperClassic
import net.sf.json.JSONArray
import net.sf.json.JSONObject

@Field def buildRootDir = "DevOpsCode"
@Field def shoonyaPipelineUser = "aakashawscredntials"
@Field def cfnUpdateTasks = [:]
@Field def allCfnUpdateSuccessful = true
@Field def slackMessageChannel = "#cloudfgh"
@Field def g_tesseractChange = false

// waf params
@Field def wafEndpointType = "ALB"

def sendSlackMessage(titleText, messageText, messageColor, channelName) {
    print("Slack Message")
    // JSONArray attachments = new JSONArray();
    // JSONObject attachment = new JSONObject();

    // attachment.put('title', titleText.toString());
    // attachment.put('text', messageText.toString());
    // attachment.put('color', messageColor);

    // attachments.add(attachment);
    // slackSend(botUser: true, channel: channelName, attachments: attachments.toString())
}

def downloadFileFromGit(gitUrl, branchName, filePath) {
    withCredentials([[$class: 'UsernamePasswordMultiBinding',
        credentialsId: 'githubcredentials',
        usernameVariable: 'GIT_USERNAME',
        passwordVariable: 'GIT_PASSWORD']]) {

        // Get the waf.yaml from devops repo
        // sh "git archive --remote=${gitUrl} --format=tar ${branchName} ${filePath} | tar xf -"
        sh "svn export https://github.com/Akayrathee/EventBridge/trunk AakashCode --force"
        sh "ls"
        sh "pwd"
    }
}


def updateCloudFormationStacksParallel(stackName, stackRegion, cfnParams) {
    cfnUpdateTasks["${stackName}"] = {
        node {
  
            stage("${stackName}") {
                script {
                    try {
                        sendSlackMessage(
                            "WAF Config Sync in Progress...",
                            " ► Stack: ${stackName} \n ► Region: ${stackRegion}\n",
                            'good',
                            slackMessageChannel
                        )

                        def gitUrl = "git@github.com:Akayrathee/EventBridge.git"
                        def branchName = "master"
                        def filePath = "lambda.yaml"
                        print(filePath)
                        downloadFileFromGit(gitUrl, branchName, filePath)

                        withAWS(credentials: 'aakashawscredntials', region: stackRegion){
                            def outputs = cfnUpdate(
                                stack:"${stackName}",
                                file:"AakashCode/lambda.yaml",
                                params:cfnParams,
                                timeoutInMinutes:180,
                                pollInterval:10000
                            )
                            print(outputs)
                        }
                    }
                    catch(error) {
                        allCfnUpdateSuccessful = false
                        // Alert to slack about failure
                        echo("Updation of the stack ${stackName} failed. Error = " + error.toString())
                        sendSlackMessage(
                            "WAF Sync Failure",
                            " ► Stack: ${stackName} \n ► Region: ${stackRegion}\n ► buildUrl: ${env.BUILD_URL}\n",
                            'danger',
                            slackMessageChannel
                        )
                    }
                }
            }
        }
    }
}

pipeline {
    agent any 

    stages {

        stage('Check for Code Change'){
            steps{
             timestamps
                {
                script
                    {
                    def lineCountFileChanges = sh ( script: "git diff HEAD^ HEAD | wc -l",returnStdout: true).trim() as Integer
                    println("Total number of line changes : ${lineCountFileChanges}")
                    if (lineCountFileChanges > 0 ){
                        g_tesseractChange = true
                    }

                    }
                }
            }
        }

        stage("EB configuration check") {
            steps {
                timestamps {
                    script {
                        dir(buildRootDir) {                
                            cfnUpdateTasks = [:]
                            allCfnUpdateSuccessful = true
                            def datas = readYaml file: '../config.yaml'
                            // def REGIONS = ["us-east-1", "us-east-2"]

                            for(data in datas.EventBridgeConfig){
                                echo ""
                                echo "Updating EventBridge in -${data.region} ..."
                                def cfnParams = new JsonSlurperClassic().parseText("{}")
                                cfnParams["LambdaFunctionLayerARN"] = data.LambdaFunctionLayer

                                echo "Cloudformation params: ${cfnParams}"
                                
                                updateCloudFormationStacksParallel("WAF-eventbridge-${data.region}", data.region, cfnParams)
                            }
                            echo "Updating stacks in parallel..."
                            echo "${cfnUpdateTasks}"
                            parallel cfnUpdateTasks
                            echo "Parallel tasks complete."

                            if (!allCfnUpdateSuccessful) {
                                error("One or more stackUpdation failed; Please check the log for more information.")
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            sendSlackMessage(
                "WafDeploymentPipeline Failure",
                " Sync Failed\n ► buildUrl: ${env.BUILD_URL}\n",
                'danger',
                slackMessageChannel
            )
            echo "Failure"
        }
        cleanup {
            echo "Cleaning Workspace.."
            cleanWs()
            echo "Exiting Script"
        }
    }
}
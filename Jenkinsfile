
#!/usr/bin/env groovy

// workspace is the current personal working directory as in Unix/Linux    
def WORKSPACE = pwd()
// defined environments as target in the Azure cloud
def environments = ["development", "staging", "production"]
// the project name
def project = 'jenkins-scripting-java'
// the GitHub API URL to retrieve all the branch names to support the multi-branch logic
def branchApi = new URL("https://api.github.com/repos/L00162879/${project}/branches")
// retrieves the brances
def branches = new groovy.json.JsonSlurper().parse(branchApi.newReader())

// this function sends build result notifications to Slack
def slackNotification(boolean success) {
    def color = success ? "#00FF00" : "#FF0000"
    def status = success ? "SUCCESSFUL" : "FAILED"
    def message = status + ": Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
    slackSend(color: color, channel: "#jenkins-ca", message: message)
}

// this function returns the target email recipients
def getEmailRecipients() {
    return 'preevenkat@gmail.com'
}

// node block - somehow mapped to a node machine acting as a jenkins node
node ("jenkins-scripting-java") {     

  // variables
  environment { 
      GIT_CREDENTIALS = ''
      BRANCH_NAME = 'main'
      JOB_NAME = ''
      BUILD_URL = ''
      BUILD_NUMBER = ''
  }

  // preconfigured tools in Jenkins
  tools { 
        maven 'Maven 3.6.3' 
        jdk 'jdk11' 
    }
  
  // cron expression to poll GitHub every 30 minutes
  triggers { pollSCM('H/30 * * * *') }

  // try-catch block - exception handling
  try {
               
        stage ('Checkout') {

            // for...each loop
            branches.each {

              // gets branch name from current iterator item and assigns current to env variable
              env.BRANCH_NAME = it.name

              // adjusts the job name
              def jobName = "${project}-${branchName}".replaceAll('/','-')
              
              //effective job with checkout steps
              job(jobName) {

                steps {
                  git branch: env.BRANCH_NAME,
                  credentialsId: env.GIT_CREDENTIALS,
                  url: 'L00162879@github.com/L00162879/jenkins-scripting-java.git'                
                }

              }
             }

        }

        stage ('Compile') {
            parallel 'Compilation': {
                git 'https://github.com/preevenkat/jenkins-scripting-java.git'
                sh 'java -version'
                sh 'mvn compile'
            }, 'Static Analysis': {
                stage("Analysis") {
                    sh 'mvn checkstyle:checkstyle'                    
                    step([$class: 'CheckStylePublisher',
                      canRunOnFailed: true,
                      defaultEncoding: '',
                      healthy: '100',
                      pattern: '**/target/checkstyle-result.xml',
                      unHealthy: '90',
                      useStableBuildAsReference: true
                    ])
                }
            },
            failFast: true
        } 
        
        // Build the Java project with Maven
        stage('Build') {

            options {
                timeout(time: 50, unit: "MINUTES")
            }

            steps {    
                // retries the build steps 3 times  
                retry(3) { 
                git 'https://github.com/preevenkat/jenkins-scripting-java.git'           
                sh 'java -version'
                sh 'mvn -B -DskipTests clean package'
                }
            }          
        }

        // Test the Java project with Maven
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    // aggregate the JUnit test reports
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        // Check if the JAR file is presented (generated) and deploys it with Maven
        stage('Deploy') {
            steps {
                echo "For this lab, it first checks if the Java JAR artifact has been created. Then delivers it."
                sh 'ls $WORKSPACE/target/jenkins-scripting-java-1.0.jar'
                sh 'mvn -B -DskipTests clean deploy'
            }
        }

        // stage to confirm if it should be deployed to Azure
        stage('Deploy to Azure - Confirmation') {
          timeout(time: 60, unit: 'MINUTES') {
            input "Deploy to Azure?"
          }
        }
        // deploy to Azure
        stage("Deploy to Azure targets") {
            def deployments = [:]
            environments.each { e ->
              deployments[e] = {
                stage(e) {
                  steps {
                    withCredentials([azureServicePrincipal('AzureServicePrincipalID')]) {
                      sh 'az login --service-principal -u $.AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID'
                      sh 'az account set -s $AZURE_SUBSCRIPTION_ID'
                      sh './jenkins/deploy-to-aks.sh jenkins-scripting-java v1.0'
                    }
                  }
                }
              }
            }
          parallel deployments
        }

        // post actions
        post {

        always {
            archiveArtifacts artifacts: '$WORKSPACE/target/*.jar', fingerprint: true
        }

        success {
            archiveArtifacts '*.tgz, *.zip, docs/build/html/**'
            emailext(
            subject: "${env.JOB_NAME} [${env.BUILD_NUMBER}] Development Promoted to Master",
            body: """<p>'${env.JOB_NAME} [${env.BUILD_NUMBER}]' Development Promoted to Master":</p>
            <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""",
            to: getEmailRecipients()
          )
        }

        aborted {
            error "Aborted, exiting now"
        }

        failure {
            emailext(
            subject: "${env.JOB_NAME} [${env.BUILD_NUMBER}] Failed!",
            body: """<p>'${env.JOB_NAME} [${env.BUILD_NUMBER}]' Failed!":</p>
            <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""",
            to: getEmailRecipients()
         )
            error "Failed, exiting now"
        }
      }

      } catch (error) {     
        // global exception handler for our pipeline   

        if ("${error}".startsWith('org.jenkinsci.plugins.workflow.steps.FlowInterruptedException')) {
          echo "Build was aborted manually"
        }

        // external notification - sends a failure message to Slack
        slackNotification(false)

        throw error
      }

}

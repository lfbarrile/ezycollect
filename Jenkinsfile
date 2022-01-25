#!groovy

import net.sf.json.JSONArray;
import net.sf.json.JSONObject;

pipeline {
    agent any
    tools {
        maven 'M2'
        jdk 'Jdk1.8u191'
    }

    environment {
        ECR_URL = 'https://758526784474.dkr.ecr.us-east-1.amazonaws.com/'
        S3_BUCKET_TEMPLATE = "nginx-example-template"
        PATH_DEPLOY = "nginx-example"
        ARCHITECTURE = "Serverless"

        // For teste purpose only (need to be set always to TRUE! )
        RUN_TEST = false
        RUN_PRE_BUILD = true
        RUN_SITE = false
        RUN_SONAR = false
        RUN_POST_BUILD = true
        RUN_COMPILE = true
        RUN_CHECKS = true
        RUN_BUILD_CI = false
    }

    options {
        //skipDefaultCheckout()
        // Only keep the 50 most recent builds
        buildDiscarder(logRotator(numToKeepStr: '50'))
        timeout(time: 20, unit: 'MINUTES')
        // disableConcurrentBuilds()
    }

    stages {
        stage('Check commit message') {
            steps {
                script {
                    current_commit_message = sh(script: '''
                git rev-list --format=%B --max-count=1 HEAD |head -2 |tail -1
              ''', returnStdout: true).trim()

                    if (current_commit_message == 'Prepare for next Release') {
                        currentBuild.result = 'ABORTED'
                        error('Parando build por ser um commit de CI.')
                    }
                }

            }
        }

		stage('Check Branch Name') {
            steps {
                script {
                    if (BRANCH_NAME.startsWith("master") || BRANCH_NAME.startsWith("feature") || BRANCH_NAME.startsWith("develop") || BRANCH_NAME.startsWith("release") || BRANCH_NAME.startsWith("hotfix")) {
                        echo "***** Let's go to the Build *****"

                    } else {
                        currentBuild.result = 'ABORTED'
                        error('Parando o build por não estar de acordo com a nomenclatura de Branch.')
                    }
                }
            }
        } 

        stage('Notify') {
            steps {
                echo sh(returnStdout: true, script: 'env')
                notifyBuild('STARTED')
            }
        }

        stage('Compile') {
            when {
                environment name: 'RUN_COMPILE', value: 'true'
            }
            steps {
                sh 'npm install --save'
            }
        }

        stage('Pre-Build CheckList') {
            when {
                environment name: 'RUN_CHECKS', value: 'true'
            }
            steps {
                parallel(
                        "Commit Behind": {
                            checkCommitBehind()
                        }
                )
            }
        }

        stage('Pre-Build') {
            when {
                environment name: 'RUN_PRE_BUILD', value: 'true'
            }
            steps {
                script {
                   if (BRANCH_NAME.startsWith("master")) {
                        echo "***** PERFORMING STEPS ON MASTER *****"
                        env['environment'] = "prd"
                        env['jobId'] = "xxxxxxxxxx"
                        env['AUTO_DEPLOY'] = false
                        env['AUTO_DEPLOY_DR'] = true
                        updateVersion(true)
                    }
                    else if (BRANCH_NAME.startsWith("develop")) {
                        echo "***** PERFORMING STEPS ON DEVELOP BRANCH *****"
                        //env['environment'] = "dev"
                        //env['jobId'] = "06ba7d2a-6ee8-4c6b-ae9f-4c1b7714c850"
                        env['environment'] = "hml"
                        env['jobId'] = "xxxxxxxxxxx"
                        env['AUTO_DEPLOY'] = true
                        updateVersion(false)
                    }
                    else if (BRANCH_NAME.startsWith("release")) {
                        echo "***** PERFORMING STEPS ON RELEASE BRANCH *****"
                        env['environment'] = "hml"
                        env['jobId'] = "xxxxxxxxxxx"
                        env['AUTO_DEPLOY'] = false
                        updateVersion(false)
                    }
                    else if (BRANCH_NAME.startsWith("feature")) {
                        echo "***** PERFORMING STEPS ON FEATURE BRANCH *****"
                        env['environment'] = "dev"
                        env['jobId'] = "xxxxxxxxxxx"
                        env['AUTO_DEPLOY'] = false
                        updateVersion(false)
                    }
                    else if (BRANCH_NAME.startsWith("hotfix")) {
                        echo "***** PERFORMING STEPS ON HOTFIX BRANCH *****"
                        env['environment'] = "prd"
                        env['jobId'] = "xxxxxxxxxxxx"
                        env['AUTO_DEPLOY'] = false
                        updateVersion(false)
                    }
                    else {
                        echo "***** BRANCHES MUST START WITH RELEASE OR DEVELOP *****"
                        echo "***** STOPPED BUILD *****"
                        currentBuild.result = 'FAILURE'
                    }

                    sh(script: '''[[ $( aws ecr describe-repositories --repository-names ${PATH_DEPLOY} 2>/dev/null ) ]] && echo 'Repository exists' || { echo 'Creating repository '$( aws ecr create-repository --repository-name ${PATH_DEPLOY} --output text | awk '{ print $5 }' ) ; }''', returnStdout: true).trim()
                    sh(script: '''[[ $( aws ecr get-repository-policy --repository-name ${PATH_DEPLOY} 2>/dev/null ) ]] && echo 'Repository policy exists' || { echo 'Setting permissions for repository' && aws ecr set-repository-policy --repository-name ${PATH_DEPLOY} --policy-text file://~/ecr-policy.json 2>/dev/null ; }''', returnStdout: true).trim()

                    env['url_docker'] = "758526784474.dkr.ecr.us-east-1.amazonaws.com/nginx-example:${newVersion}"
                }

                sh 'echo "***** FINISHED PRE-BUILD STEP *****"'
            }
        }

        stage('Build docker image'){
            steps {
                script {
                    docker.withRegistry("${env.ECR_URL}",'ecr:us-east-1:ecr-private-registry'){
                        def app = docker.build("nginx-example")
                        app.push(env['newVersion'])
                        app.push('latest')
                    }
                }
            }
        }

        stage('Publish Cloudformation Template and parameter') {
            steps{
                script {
                    echo "upload template to s3://${env.S3_BUCKET_TEMPLATE}/nginx-example/${newVersion}/templates/"
                    sh "aws s3 cp cloudformation/template/ s3://${S3_BUCKET_TEMPLATE}/nginx-example/${newVersion}/templates/ --recursive"
                    echo "upload parameter files to s3://${env.S3_BUCKET_TEMPLATE}/nginx-example/${newVersion}/parameters/"
                    sh "aws s3 sync cloudformation/parameters/ s3://${env.S3_BUCKET_TEMPLATE}/nginx-example/${newVersion}/parameters/"
                    env['fileOutput'] = "cloudformation.yml"
                }
            }
        }

        stage('deployment'){
            when {
                environment name: 'AUTO_DEPLOY', value: 'true'
            }
            steps {
                deploy(env['environment'], env['jobId'])
            }
        }
        
        stage('Post-Build Actions') {
            when {
                environment name: 'RUN_POST_BUILD', value: 'true'
            }
            steps {
                parallel(
                        "Get Sonar Status publish": {
                            sh 'echo "Not implemented YET"'
                            // CHECK http://stackoverflow.com/questions/42909439/using-waitforqualitygate-in-a-jenkins-declarative-pipeline
                        }
                )
            }
        }
    }
    post {
        success {
            notifyBuild('SUCCESSFUL')
        }
        failure {
            notifyBuild('FAILED')
        }
        always {
            deleteDir() //F**k Dawn you jenkins !
        }
    }
}

def notifyBuild(String buildStatus = 'STARTED') {
    // build status of null means successful
    buildStatus = buildStatus ?: 'SUCCESSFUL'

    // Default values
    String colorCode = '#FF0000'
    String subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
    String summary = "${subject} \n (${env.BUILD_URL})  "
    String details = """<p>${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
    <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>"""


    JSONArray attachments = new JSONArray();
    JSONObject attachment = new JSONObject();

    // Override default values based on build status
    if (buildStatus == 'STARTED') {
        colorCode = '#FFFF00'
        attachment.put('text','Build Starting?')
        attachment.put('thumb_url','http://iconarchive.com/show/vegas-icons-by-musett/coins-7000-icon.html')
    } else if (buildStatus == 'SUCCESSFUL') {
        colorCode = '#00FF00'
        attachment.put('text','Build realizado com sucesso.')
        attachment.put('thumb_url','http://iconarchive.com/show/vegas-icons-by-musett/coins-7000-icon.html')

        JSONArray fields = new JSONArray();
        JSONObject field = new JSONObject();

        field.put('title', 'Template S3');
        field.put('value', env['fileOutput']);
        fields.add(field);

        field = new JSONObject();

        field.put('title', 'Version');
        field.put('value', env['newVersion']);
        fields.add(field);

        field.put('title', 'Path');
        field.put('value', 'nginx-example');
        fields.add(field);

        attachment.put('fields',fields);

    } else {
        attachment.put('text','money é good e nois num have')
        attachment.put('thumb_url','http://iconarchive.com/show/rounded-square-icons-by-hopstarter/Button-Delete-icon.html')
        colorCode = '#FF0000'
    }

    String buildUrl = "${env.BUILD_URL}";
    attachment.put('title', subject);
    attachment.put('callback_id', buildUrl);
    attachment.put('title_link', buildUrl);
    attachment.put('fallback', subject);
    attachment.put('color', colorCode);

    attachments.add(attachment);

    // Send notifications
    echo attachments.toString();
    slackSend(attachments: attachments.toString())

}

def checkCommitBehind() {
    sh 'echo "Verificando se branch necessita de merge com a master."'
    script {
        sh(script: '''set +x; set +e;
                      git fetch;
                      commitsBehind=$(git rev-list --left-right --count origin/master... |awk '{print $1}');
                      if [ ${commitsBehind} -ne 0 ]
                      then
                        echo "Esta branch está ${commitsBehind} commits atrás da master!"
                        exit 1
                      else
                        echo "Esta branch não tem commits atrás da master."
                      fi''')
    }

}

def bump_git_tag() {
    echo "Bumping Git CI Tag"

    script {
        sh "git fetch --tags"
        env['bumpci_tag'] = sh(script: '''
        current_tag=$(git tag -n9 -l |grep bumpci |awk '{print $1}' |sort -V |tail -1)
        if [[ $current_tag == '' ]]
        then
          current_tag=0.0.1
        fi
        echo "${bumpci_tag}"
      ''', returnStdout: true)

        //sh "git tag -a ${bumpci_tag} -m bumpci && git push origin refs/tags/${bumpci_tag}"
    }
}

def updateVersion(boolean isMaster) {
    sh "git fetch --tags"
        env['docker_version'] = sh(script: '''
            current_tag=`git tag -n9 -l |grep docker_version |awk '{print $1}' |sort -V |tail -1`
            echo ${current_tag}
            ''', returnStdout: true).trim()

    if (env['docker_version'] == ''){
        env['docker_version'] = "1.0.0"
    }

    def oldVersion = "${env.docker_version}".tokenize('.')
    major = oldVersion[0].toInteger()
    minor = oldVersion[1].toInteger()
    patch = oldVersion[2].toInteger()

    if (isMaster) {
        major += 1
        minor = 0
        patch = 0
    } else {
        patch += 1
    }
    env['newVersion'] = major + '.' + minor + '.' + patch
    //env['newVersion'] = '1.0.7'

    bump_version_tag()
}

def version_code_tag() {
    echo "getting Git version Tag"
    script {
        sh "git fetch --tags"
        env['bumpci_tag'] = sh(script: '''
            current_tag=$(git tag -n9 -l |grep version |awk '{print $1}' |sort -V |tail -1)
            if [[ $current_tag == '' ]]
            then
              echo 1.0.0 |tr -d '\n'
            else
              echo "${current_tag} + 1" |/bin/bc |/bin/tr -d '\n'"
            fi
            ''', returnStdout: true).trim()
    }
  }

def bump_version_tag() {
  echo "Bumping version CI Tag"
  script {
      sh "git tag -a ${newVersion} -m docker_version && git push origin refs/tags/${newVersion}"
  }
}

def deploy(environment, job_id){
    script {
        echo "Iniciando deploy no ambiente de ${environment}"
        env['profile'] = environment
        print_versions()
        env['current_commit_message'] = current_commit_message
        step([$class: "RundeckNotifier",
            includeRundeckLogs: true,
            jobId: "${job_id}",
            nodeFilters: "",
            options: """
                        Arquitetura=${ARCHITECTURE}
                        template=${fileOutput}
                        version=${newVersion}
                        path=${PATH_DEPLOY}
                        ChangeLog=${current_commit_message}
                    """,
            rundeckInstance: "xxxx.ezycollect.com",
            shouldFailTheBuild: true,
            shouldWaitForRundeckJob: true,
            tags: "",
            tailLog: true])
    }
    valid_stats()
}

def valid_stats() {
    script {
        def stats = sh(script: '''
                        aws cloudformation describe-stacks --stack-name ${PATH_DEPLOY} --profile ${profile} | jq '.Stacks[0].StackStatus' | sed -e 's/\"//g'
                             ''', returnStdout: true).trim()
        if ( stats == 'UPDATE_COMPLETE' || stats == 'CREATE_COMPLETE' || stats == 'OK') {
             echo "Success"
        }
        else {
            currentBuild.result = 'FAILURE'
            error("Problema ao realizar o deploy")
        }
    }
}

def print_versions(){
    script{
        def oldVersion = sh(script: '''
                            aws cloudformation describe-stacks --stack-name ${PATH_DEPLOY} --profile ${profile} --output text | grep Versao | cut -f3
                        ''',returnStdout: true).trim()

        //To ELK and Rollback
        echo "OLD_VERSION=${oldVersion}"
        echo "NEW_VERSION=${newVersion}"
    }
}
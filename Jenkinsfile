def sonarHome = ""
def imagereference = ""
def GITTEA_HOST = "gitea-app-cicd.smartplay-np.lcsd.hksarg"
def GITTEA_OPS_HOST = "gitea-ops-cicd.smartplay-np.lcsd.hksarg"
def GITTEA_OPS_ALICLOUD_HOST = "gitea-gitea"
def DOCKER_HOST_ONPRIM = "docker-cicd.smartplay-np.lcsd.hksarg:8443"
def DOCKER_GROUP_NAME = "cicd"
def GITTEA_PORT = "2222"
def GITTEA_OPS_PORT = "2222"
def GITTEA_OPS_ALICLOUD_PORT = "2222"

pipeline {
    agent {
        node {
            label 'maven'
        }
    }
    stages {
        stage('Setup parameters') {
            steps {
                script {
                    properties([
                        parameters([
                            // choice(
                            //     choices: ['ONE', 'TWO'], 
                            //     name: 'PARAMETER_01'
                            // ),
                            // booleanParam(
                            //     defaultValue: true, 
                            //     description: '', 
                            //     name: 'BOOLEAN'
                            // ),
                            // text(
                            //     defaultValue: '''
                            //     this is a multi-line 
                            //     string parameter example
                            //     ''', 
                            //      name: 'MULTI-LINE-STRING'
                            // ),
                            string(
                                defaultValue: 'com.redhat.demo',
                                name: 'PACKAGE_ARTIFACT_GROUP',
                                trim: true
                            ),
                            string(
                                defaultValue: 'springboot-helloworld',
                                name: 'PACKAGE_ARTIFACT_NAME',
                                trim: true
                            ),
                            string(
                                defaultValue: 'demo',
                                name: 'PACKAGE_BUILD_NAME',
                                trim: true
                            ),
                            string(
                                defaultValue: 'demo:latest',
                                name: 'PACKAGE_BUILD_OUTPUT',
                                trim: true
                            ),
                            string(
                                defaultValue: 'demo',
                                name: 'GITEA_OPS_ORGANIZATION',
                                trim: true
                            ),
                            string(
                                defaultValue: 'image-registry.openshift-image-registry.svc:5000/cicd-common/demo:latest',
                                name: 'KUSTOMIZE_IMAGE_NAME',
                                trim: true
                            ),
                            string(
                                defaultValue: 'image-registry.openshift-image-registry.svc:5000/cicd-common/demo:latest',
                                name: 'KUSTOMIZE_PROJECT_NAME',
                                trim: true
                            ),
                            string(
                                defaultValue: 'springboot-helloworld.git',
                                name: 'GITEA_OPS_PROJECT',
                                trim: true
                            ),
                            string(
                                defaultValue: 'demo',
                                name: 'GITEA_OPS_ALICLOUD_ORGANIZATION',
                                trim: true
                            ),
                            string(
                                defaultValue: 'springboot-helloworld.git',
                                name: 'GITEA_OPS_ALICLOUD_PROJECT',
                                trim: true
                            ),
                            string(
                                defaultValue: 'demo',
                                name: 'GITEA_APP_ORGANIZATION',
                                trim: true
                            ),
                            string(
                                defaultValue: 'springboot-helloworld.git',
                                name: 'GITEA_APP_PROJECT',
                                trim: true
                            )
                        ])
                    ])
                }
            }
        }
        stage('checkout') {
            steps {
                checkout(
                    [
                        $class: 'GitSCM', branches: [[name: '*/main']],
                        userRemoteConfigs:[[credentialsId: 'GITEA-APPS-DEPLOY',
                        url:"ssh://git@${GITTEA_HOST}:${GITTEA_PORT}/${params.GITEA_APP_ORGANIZATION}/${params.GITEA_APP_PROJECT}"]]
                    ])
            }
        }
        stage('test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('compile') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }
        stage('code scanning') {
            
            steps {
                script {
                    sonarHome = tool 'sonarqube-maven'
                }
                withSonarQubeEnv(installationName: 'sonarqube-maven', envOnly: true) {
                    println "${env.SONAR_HOST_URL}"
                    sh 'env'
                    sh "echo ${sonarHome}"
                    withCredentials([
                        file(credentialsId: 'CICD-CA-JKS', variable: 'CICDCAJKS' ),
                        string(credentialsId: 'CICD-CA-JKS-KEY', variable: 'CICDCAJKSKEY' )
                    ]) {
                        sh "export SONAR_SCANNER_OPTS=\"-Djava.net.debug=all -Djavax.net.ssl.trustStore=${sonarHome}/cicd-ca.jks -Djavax.net.ssl.keyStore=${sonarHome}/cicd-ca.jks -Djavax.net.ssl.keyStorePassword=changeit -Djavax.net.ssl.trustStorePassword=changeit -Dmaven.wagon.http.ssl.insecure=true\""
                        sh "${sonarHome}/bin/sonar-scanner -Dsonar.projectKey=${params.PACKAGE_ARTIFACT_GROUP}.${PACKAGE_ARTIFACT_NAME} -Dsonar.java.binaries=target"
                    }
                }
            }
        }
        stage('deploy to nexus') {
            steps {
                sh 'mvn deploy -DskipTests -Pdeploy-to-nexus'
            }
        }
        stage('build in openshift and push to nexus') {
            steps {
                script{
                    def pomFile = 'pom.xml'
                    def pom = readMavenPom file: pomFile
                    print "${pom.version}"
                    sh "mvn dependency:copy -Dartifact=${params.PACKAGE_ARTIFACT_GROUP}:${params.PACKAGE_ARTIFACT_NAME}:${pom.version} -DoutputDirectory=/tmp/"
                }
                dir('binary') {
                    sh """
                        mvn dependency:copy -Dartifact=${params.PACKAGE_ARTIFACT_GROUP}:${params.PACKAGE_ARTIFACT_NAME}:${pom.version} -DoutputDirectory=./
                    """
                    script {
                        openshift.withCluster('smartplay-np'){
                            openshift.withProject('cicddemo-dev'){
                                def result = openshift.raw("start-build --follow ${PACKAGE_BUILD_NAME} --from-file=./${params.PACKAGE_ARTIFACT_NAME}-${pom.version}.jar")
                                echo "${result.out}"
                                sh "exit ${result.status}"
                            }
                        }
                    }
                }
            }
        }
        stage('Scan new build in ACS') {
            steps {
                withCredentials([string(credentialsId: 'ACS_TOKEN', variable: 'ROX_API_TOKEN'), string(credentialsId: 'ACS_URL', variable: 'ROX_CENTRAL_ADDRESS')]){
                    sh """
                      roxctl --insecure-skip-tls-verify -e ${ROX_CENTRAL_ADDRESS} image check --image ${DOCKER_HOST_ONPRIM}/${DOCKER_GROUP_NAME}/${PACKAGE_BUILD_OUTPUT}
                    """
                }
            }
        }
        stage('deploy in openshift') {
            steps {
                script {
                    
                    openshift.withCluster('smartplay-np'){
                        openshift.withProject('cicddemo-dev'){
                            def result1 = openshift.tag("--source=docker","${DOCKER_HOST_ONPRIM}/${DOCKER_GROUP_NAME}/${params.PACKAGE_BUILD_OUTPUT}", "cicd-common/${params.PACKAGE_ARTIFACT_NAME}:latest")
                            echo "${result1.out}"
                            def result2 = openshift.tag("cicd-common/${params.PACKAGE_ARTIFACT_NAME}:latest","cicd-common/${params.PACKAGE_ARTIFACT_NAME}:dev")
                            echo "${result2.out}"
                            
                        }
                        openshift.withProject('cicd-common'){
                            def istag = openshift.selector("istag/${params.PACKAGE_ARTIFACT_NAME}:dev").object()
                            imagereference  =istag.image.dockerImageReference
                            echo "IMAGEREF ${imagereference}"
                        }
                        dir('git-ops') {
                            withCredentials([sshUserPrivateKey(credentialsId: 'GITEA-APPS-DEPLOY', keyFileVariable: 'keyfile')]) {
                                sh """
                                  rm -rf springboot-helloworld
                                  export GIT_SSH_COMMAND="ssh -i ${keyfile} -o IdentitiesOnly=yes -o StrictHostKeyChecking=no" 
                                  git clone 'ssh://git@${GITTEA_OPS_HOST}:${GITTEA_OPS_PORT}/${params.GITEA_OPS_ORGANIZATION}/${params.GITEA_OPS_PROJECT}'
                                  cd springboot-helloworld/environments/dev
                                  git config user.email "cicd@smartplay-np.lcsd.hksarg"
                                  git config user.name "cicd"
                                  git status
                                  kustomize edit set image ${params.KUSTOMIZE_IMAGE_NAME}=${imagereference}
                                  git add ./kustomization.yaml
                                  git commit -m "CI Image Update"
                                  git tag -a v0.0.1 -m "CI Image Update" --force
                                  git push origin main
                                  git push --tags --force
                                """
                            }
                            
                        }
                        openshift.withProject('cicd-tools'){
                            def resultLogin = openshift.raw('rsh argocd-application-controller-0  argocd --config /tmp/config  login  --insecure --core')
                            echo "${resultLogin.out}"
                            
                            try {
                                def resultSyn = openshift.raw("rsh argocd-application-controller-0  argocd --config /tmp/config app sync \"cicd-tools/${params.KUSTOMIZE_PROJECT_NAME}\"")
                                echo "${resultSyn.out}"
                            } catch (Exception e){
                                echo "Exception "+e.toString() +" expectred"
                            }
                            sh 'sleep 3'
                            def resultWait = openshift.raw('rsh argocd-application-controller-0  argocd --config /tmp/config app wait "cicd-tools/springboot-demo-dev"')
                            echo "${resultWait.out}"
                        }
                    }
                }
            }
        }
        

        stage('deploy in alicloud') {
            agent {
                node {
                    label 'alicloud-nonprod-jumphost'
                }
            }
            steps {
                script {
                    withCredentials([file(credentialsId: 'DOCKER-CICD-AUTH', variable: 'DOCKERCICDAUTH'), 
                                 file(credentialsId: 'ALICLOUD-NP-REGISTRY-AUTH', variable: 'ALICLOUDNPREGISTRYAUTH')]) {
                        sh """
                        podman pull --authfile ${DOCKERCICDAUTH} --tls-verify=false  ${imagereference}
                        podman tag ${imagereference} smartplay-nonp-registry-vpc.cn-hongkong.cr.aliyuncs.com/cicd-tools/springboot-helloworld:${currentBuild.number}
                        podman push --authfile ${ALICLOUDNPREGISTRYAUTH} --tls-verify=false  smartplay-nonp-registry-vpc.cn-hongkong.cr.aliyuncs.com/cicd-tools/springboot-helloworld:${currentBuild.number}
                        podman rmi smartplay-nonp-registry-vpc.cn-hongkong.cr.aliyuncs.com/cicd-tools/springboot-helloworld:${currentBuild.number}
                        podman rmi ${imagereference}
                        """
                                 }
                    openshift.withCluster('alicloud-nonprod'){
                        openshift.withProject('cicd-tools'){
                            withCredentials([sshUserPrivateKey(credentialsId: 'GITEA-APPS-DEPLOY', keyFileVariable: 'keyfile')]) {
                                writeFile file: "gitcommand.txt", text: """
                                    rm -rf /tmp/git-ops
                                    mkdir -p /tmp/git-ops
                                    cd /tmp/git-ops
                                    export GIT_SSH_COMMAND="ssh -i /tmp/keyfile.txt -o IdentitiesOnly=yes -o StrictHostKeyChecking=no" 
                                    git clone 'ssh://git@${GITTEA_OPS_ALICLOUD_HOST}:${GITTEA_OPS_ALICLOUD_PORT}/${params.GITEA_OPS_ALICLOUD_ORGANIZATION}/${params.GITEA_OPS_ALICLOUD_PROJECT}'
                                    cd springboot-helloworld/environments/dev
                                    git config user.email "cicd@smartplay-np.lcsd.hksarg"
                                    git config user.name "cicd"
                                    git status
                                    kustomize edit set image image-registry.openshift-image-registry.svc:5000/cicd-common/demo:latest=smartplay-nonp-registry-vpc.cn-hongkong.cr.aliyuncs.com/cicd-tools/springboot-helloworld:${currentBuild.number}
                                    git add ./kustomization.yaml
                                    git commit -m "CI Image Update"
                                    git tag -a v0.0.1 -m "CI Image Update" --force
                                    git push origin main
                                    git push --tags --force
                                    rm -f /tmp/keyfile.txt
                                    rm -f /tmp/gitcommand.txt
                                """
                                def pod = openshift.selector('pod',['app.kubernetes.io/name':'argocd-server']).object()
                                echo pod.metadata.name
                                def resultCp = openshift.raw("cp ${keyfile} ${pod.metadata.name}:/tmp/keyfile.txt")
                                echo "${resultCp.out}"
                                def resultCp2 = openshift.raw("cp gitcommand.txt ${pod.metadata.name}:/tmp/gitcommand.txt")
                                echo "${resultCp2.out}"
                                def resultExec = openshift.raw("exec ${pod.metadata.name} -- bash /tmp/gitcommand.txt")
                                echo "${resultExec.out}"
                                def resultLogin = openshift.raw("rsh ${pod.metadata.name}  argocd --config /tmp/config  login  --insecure --core")
                                echo "${resultLogin.out}"
                                
                                try {
                                    def resultSyn = openshift.raw("rsh ${pod.metadata.name}  argocd --config /tmp/config app sync \"cicd-tools/springboot-demo-dev\"")
                                    echo "${resultSyn.out}"
                                } catch (Exception e){
                                    echo "Exception "+e.toString() +" expectred"
                                }
                                sh 'sleep 3'
                                def resultWait = openshift.raw("rsh ${pod.metadata.name}   argocd --config /tmp/config app wait \"cicd-tools/springboot-demo-dev\"")
                                echo "${resultWait.out}"
                            }
                        }  
                    }
                }
            }
        }
    }
}
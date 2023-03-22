def sonarHome = ""
def imagereference = ""
def GITTEA_HOST = "gitea-app-cicd.smartplay-np.lcsd.hksarg"
def GITTEA_OPS_HOST = "gitea-ops-cicd.smartplay-np.lcsd.hksarg"
def GITTEA_OPS_ALICLOUD_HOST = "gitea-gitea"
def DOCKER_HOST_ONPRIM = "docker-cicd.smartplay-np.lcsd.hksarg:8443"
def DOCKER_HOST_ALICLOUD = "smartplay-nonp-registry-vpc.cn-hongkong.cr.aliyuncs.com"
def DOCKER_GROUP_NAME = "cicd"
def DOCKER_GROUP_ALICLOUD_NAME = "cicd-tools"
def GITTEA_PORT = "2222"
def GITTEA_OPS_PORT = "2222"
def GITTEA_OPS_ALICLOUD_PORT = "2222"
def pom 

pipeline {
    agent {
        node {
            label 'maven'
        }
    }
    stages {
        stage('Check Parameters') {
            steps {
                script {
                    if (!params.containsKey("GITEA_ORGANIZATION") 
                        || !params.containsKey("GITEA_PROJECT")
                        || !params.containsKey("BUILD_AND_DEPLOY_TO_CONTAINER")
                        ) {
                            //   GITEA_ORGANIZATION -> e.g. demo
                            //   GITEA_PROJECT -> e.g. springboot-helloworld
                            //   BUILD_AND_DEPLOY_TO_CONTAINER -> true
                            properties([
                                parameters([
                                    string(
                                        name: 'GITEA_ORGANIZATION',
                                        trim: true
                                    ),
                                    string(
                                        name: 'GITEA_PROJECT',
                                        trim: true
                                    ),
                                    booleanParam(
                                        name: 'BUILD_AND_DEPLOY_TO_CONTAINER',
                                        defaultValue: true
                                    )
                                ])
                            ])
                        }
                    if (params.GITEA_ORGANIZATION.isEmpty() 
                    || params.GITEA_PROJECT.isEmpty()) {
                        error("Missing Mandatory Paramters")
                    }
                    
                }
            }
        }
        stage('checkout') {
            steps {
                checkout(
                    [
                        $class: 'GitSCM', branches: [[name: '*/main']],
                        userRemoteConfigs:[[credentialsId: 'GITEA-APPS-DEPLOY',
                        url:"ssh://git@${GITTEA_HOST}:${GITTEA_PORT}/${params.GITEA_ORGANIZATION}/${params.GITEA_PROJECT}"]]
                    ])
                script {
                    def pomFile = 'pom.xml'
                    pom = readMavenPom file: pomFile
                    print "${pom.version}"
                    print "${pom.groupId}"
                    print "${pom.artifactId}"
                }
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
                        sh "${sonarHome}/bin/sonar-scanner -Dsonar.projectKey=${pom.groupId}.${pom.artifactId} -Dsonar.java.binaries=target"
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
            when {
                expression {
                    return params.BUILD_AND_DEPLOY_TO_CONTAINER
                }                
            }
            steps {
                script{
                    print "${pom.version}"
                    print "${pom.groupId}"
                    print "${pom.artifactId}"
                    sh "mvn dependency:copy -Dartifact=${pom.groupId}:${pom.artifactId}:${pom.version} -DoutputDirectory=/tmp/"

                    dir('binary') {
                        sh """
                            mvn dependency:copy -Dartifact=${pom.groupId}:${pom.artifactId}:${pom.version} -DoutputDirectory=./
                        """
                        openshift.withCluster('smartplay-np'){
                            openshift.withProject('cicddemo-dev'){
                                def result = openshift.raw("start-build --follow ${pom.artifactId} --from-file=./${pom.artifactId}-${pom.version}.jar")
                                echo "${result.out}"
                                sh "exit ${result.status}"
                            }
                        }
                    }
                }
            }
        }
        stage('Scan new build in ACS') {
            when {
                expression {
                    return params.BUILD_AND_DEPLOY_TO_CONTAINER
                }                
            }
            steps {
                withCredentials([string(credentialsId: 'ACS_TOKEN', variable: 'ROX_API_TOKEN'), string(credentialsId: 'ACS_URL', variable: 'ROX_CENTRAL_ADDRESS')]){
                    sh """
                      roxctl --insecure-skip-tls-verify -e ${ROX_CENTRAL_ADDRESS} image check --image ${DOCKER_HOST_ONPRIM}/${DOCKER_GROUP_NAME}/${pom.artifactId}
                    """
                }
            }
        }
        stage('deploy in openshift') {
            when {
                expression {
                    return params.BUILD_AND_DEPLOY_TO_CONTAINER
                }                
            }
            steps {
                script {
                    
                    openshift.withCluster('smartplay-np'){
                        openshift.withProject('cicddemo-dev'){
                            def result1 = openshift.tag("--source=docker","${DOCKER_HOST_ONPRIM}/${DOCKER_GROUP_NAME}/${pom.artifactId}", "cicd-common/${pom.artifactId}:latest")
                            echo "${result1.out}"
                            def result2 = openshift.tag("cicd-common/${pom.artifactId}:latest","cicd-common/${pom.artifactId}:dev")
                            echo "${result2.out}"
                            
                        }
                        openshift.withProject('cicd-common'){
                            def istag = openshift.selector("istag/${pom.artifactId}:dev").object()
                            imagereference  =istag.image.dockerImageReference
                            echo "IMAGEREF ${imagereference}"
                        }
                        dir('git-ops') {
                            withCredentials([sshUserPrivateKey(credentialsId: 'GITEA-APPS-DEPLOY', keyFileVariable: 'keyfile')]) {
                                sh """
                                  rm -rf springboot-helloworld
                                  export GIT_SSH_COMMAND="ssh -i ${keyfile} -o IdentitiesOnly=yes -o StrictHostKeyChecking=no" 
                                  git clone 'ssh://git@${GITTEA_OPS_HOST}:${GITTEA_OPS_PORT}/${params.GITEA_ORGANIZATION}/${params.GITEA_PROJECT}'
                                  cd springboot-helloworld/environments/dev
                                  git config user.email "cicd@smartplay-np.lcsd.hksarg"
                                  git config user.name "cicd"
                                  git status
                                  kustomize edit set image image-registry.openshift-image-registry.svc:5000/cicd-common/${pom.artifactId}:latest=${imagereference}
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
                                def resultSyn = openshift.raw("rsh argocd-application-controller-0  argocd --config /tmp/config app sync \"cicd-tools/${pom.artifactId}-dev\"")
                                echo "${resultSyn.out}"
                            } catch (Exception e){
                                echo "Exception "+e.toString() +" expectred"
                            }
                            sh 'sleep 3'
                            def resultWait = openshift.raw("rsh argocd-application-controller-0  argocd --config /tmp/config app wait \"cicd-tools/${pom.artifactId}-dev\"")
                            echo "${resultWait.out}"
                        }
                    }
                }
            }
        }
        

        stage('deploy in alicloud') {
            when {
                expression {
                    return params.BUILD_AND_DEPLOY_TO_CONTAINER
                }                
            }
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
                        podman tag ${imagereference} ${DOCKER_HOST_ALICLOUD}/${DOCKER_GROUP_ALICLOUD_NAME}/${pom.artifactId}:${currentBuild.number}
                        podman push --authfile ${ALICLOUDNPREGISTRYAUTH} --tls-verify=false ${DOCKER_HOST_ALICLOUD}/${DOCKER_GROUP_ALICLOUD_NAME}/${pom.artifactId}:${currentBuild.number}
                        podman rmi ${DOCKER_HOST_ALICLOUD}/${DOCKER_GROUP_ALICLOUD_NAME}/${pom.artifactId}:${currentBuild.number}
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
                                    git clone 'ssh://git@${GITTEA_OPS_ALICLOUD_HOST}:${GITTEA_OPS_ALICLOUD_PORT}/${params.GITEA_ORGANIZATION}/${params.GITEA_PROJECT}'
                                    cd springboot-helloworld/environments/dev
                                    git config user.email "cicd@smartplay-np.lcsd.hksarg"
                                    git config user.name "cicd"
                                    git status
                                    kustomize edit set image image-registry.openshift-image-registry.svc:5000/cicd-common/${pom.artifactId}:latest=${DOCKER_HOST_ALICLOUD}/${DOCKER_GROUP_ALICLOUD_NAME}/${pom.artifactId}:${currentBuild.number}
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
                                    def resultSyn = openshift.raw("rsh ${pod.metadata.name}  argocd --config /tmp/config app sync \"cicd-tools/${pom.artifactId}-dev\"")
                                    echo "${resultSyn.out}"
                                } catch (Exception e){
                                    echo "Exception "+e.toString() +" expectred"
                                }
                                sh 'sleep 3'
                                def resultWait = openshift.raw("rsh ${pod.metadata.name}   argocd --config /tmp/config app wait \"cicd-tools/${pom.artifactId}-dev\"")
                                echo "${resultWait.out}"
                            }
                        }  
                    }
                }
            }
        }
    }
}
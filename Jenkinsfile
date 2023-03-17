def sonarHome =""
def imagereference=""
pipeline {
    agent {
        node {
            label 'maven'
        }
    }
    stages {
        stage('checkout') {
            steps {
                sh 'mvn -v'
                sh 'env'
                checkout([$class: 'GitSCM', branches: [[name: '*/main']], userRemoteConfigs:[[credentialsId: 'GITEA-APPS-DEPLOY', url:'ssh://git@gitea-app-cicd.smartplay-np.lcsd.hksarg:2222/demo/springboot-helloworld.git']]  ])
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
                        sh "${sonarHome}/bin/sonar-scanner -Dsonar.projectKey=${env.JOB_NAME} -Dsonar.java.binaries=target"
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
                    sh 'mvn dependency:copy -Dartifact=com.redhat.demo:springboot-helloworld:0.0.1 -DoutputDirectory=/tmp/'
                }
                dir('binary') {
                    sh '''
                        mvn dependency:copy -Dartifact=com.redhat.demo:springboot-helloworld:0.0.1 -DoutputDirectory=./
                        #oc login --insecure-skip-tls-verify -u cicd -pCicd#lcsd2022 https://api.nonp-cluster.smartplay-np.lcsd.hksarg:6443
                        #oc -n cicddemo-dev start-build demo --from-file=./springboot-helloworld-0.0.1.jar
                    '''
                    script {
                        openshift.withCluster('smartplay-np'){
                            openshift.withProject('cicddemo-dev'){
                                def result = openshift.raw('start-build --follow demo --from-file=./springboot-helloworld-0.0.1.jar')
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
                    sh '''
                      roxctl --insecure-skip-tls-verify -e ${ROX_CENTRAL_ADDRESS} image check --image docker-cicd.smartplay-np.lcsd.hksarg:8443/cicd/demo:latest
                    '''
                }
            }
        }
        stage('deploy in openshift') {
            steps {
                script {
                    
                    openshift.withCluster('smartplay-np'){
                        openshift.withProject('cicddemo-dev'){
                            def result1 = openshift.tag('--source=docker','docker-cicd.smartplay-np.lcsd.hksarg:8443/cicd/demo:latest', 'cicd-common/springboot-dmeo:latest')
                            echo "${result1.out}"
                            def result2 = openshift.tag('cicd-common/springboot-dmeo:latest','cicd-common/springboot-dmeo:dev')
                            echo "${result2.out}"
                            
                        }
                        openshift.withProject('cicd-common'){
                            def istag = openshift.selector('istag/springboot-dmeo:dev').object()
                            imagereference  =istag.image.dockerImageReference
                            echo "IMAGEREF ${imagereference}"
                        }
                        dir('git-ops') {
                            withCredentials([sshUserPrivateKey(credentialsId: 'GITEA-APPS-DEPLOY', keyFileVariable: 'keyfile')]) {
                                sh """
                                  rm -rf springboot-helloworld
                                  export GIT_SSH_COMMAND="ssh -i ${keyfile} -o IdentitiesOnly=yes -o StrictHostKeyChecking=no" 
                                  git clone 'ssh://git@gitea-ops-cicd.smartplay-np.lcsd.hksarg:2222/demo/springboot-helloworld.git'
                                  cd springboot-helloworld/environments/dev
                                  git config user.email "cicd@smartplay-np.lcsd.hksarg"
                                  git config user.name "cicd"
                                  git status
                                  kustomize edit set image image-registry.openshift-image-registry.svc:5000/cicd-common/demo:latest=${imagereference}
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
                                def resultSyn = openshift.raw('rsh argocd-application-controller-0  argocd --config /tmp/config app sync "cicd-tools/springboot-demo-dev"')
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
                                    git clone 'ssh://git@gitea-gitea:2222/demo/springboot-helloworld.git'
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
                                def resultCp2 = openshift.raw("cp gitcommand.txt ${pod.metadata.name}:/tmp/gitcommand.txt")
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
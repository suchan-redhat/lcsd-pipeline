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
def DEV_SUFFIX = "dev"
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
                        || !params.containsKey("NAMESPACE_PREFIX")                        
                        || !params.containsKey("BUILD_AND_DEPLOY_TO_CONTAINER")
                        || !params.containsKey("RELEASE_VERSION")
                        ) {
                            //   GITEA_ORGANIZATION -> e.g. demo
                            //   GITEA_PROJECT -> e.g. springboot-helloworld
                            //   BUILD_AND_DEPLOY_TO_CONTAINER -> true
                            properties([
                                parameters([
                                    string(
                                        name: 'GITEA_ORGANIZATION',
                                        description: 'Organization Name, should be smartplay',
                                        trim: true
                                    ),
                                    string(
                                        name: 'RELEASE_VERSION',
                                        description: 'Release in vX.Y.Z semver',
                                        trim: true
                                    ),
                                    string(
                                        name: 'NAMESPACE_PREFIX',
                                        description: 'namespace prefix (e.g.: cicddemo)',
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
                    || params.NAMESPACE_PREFIX.isEmpty()
                    || params.RELEASE_VERSION.isEmpty()
                    ) {
                        error("Missing Mandatory Paramters")
                    }
                    
                }
            }
        }
        stage('checkout') {
            steps {
                checkout(
                    [
                        $class: 'GitSCM', branches: [[name: "${params.RELEASE_VERSION}"]],
                        userRemoteConfigs:[[credentialsId: 'GITEA-APPS-DEPLOY',
                        url:"ssh://git@${GITTEA_HOST}:${GITTEA_PORT}/${params.GITEA_ORGANIZATION}/${env.JOB_BASE_NAME}"]]
                    ])
                script {
                    def pomFile = 'pom.xml'
                    pom = readMavenPom file: pomFile
                    if (!pom.groupId) {
                        pom.groupId = sh script: 'mvn  help:evaluate -Dexpression=project.groupId -q -DforceStdout', returnStdout: true
                    }
                    print "${pom.version}"
                    print "${pom.groupId}"
                    print "${pom.artifactId}"
                }
            }
        }
        stage('test') {
            steps {
                script {
                    try {
                        sh 'mvn versions:display-dependency-updates'
                        sh 'mvn test'
                    } catch (err) {
                        unstable("WARNING no test")
                    }
                }
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
                
                    withSonarQubeEnv(installationName: 'sonarqube-maven', envOnly: true) {
                        println "${env.SONAR_HOST_URL}"
                        sh 'env'
                        sh "echo ${sonarHome}"

                        withCredentials([
                            file(credentialsId: 'CICD-CA-JKS', variable: 'CICDCAJKS' ),
                            string(credentialsId: 'CICD-CA-JKS-KEY', variable: 'CICDCAJKSKEY' )
                        ]) {
                            if (pom.packaging == "pom" ) {
                                echo "SKIP sonarqube scanning for pom"
                            } else {
                                sh "export SONAR_SCANNER_OPTS=\"-Djava.net.debug=all -Djavax.net.ssl.trustStore=${sonarHome}/cicd-ca.jks -Djavax.net.ssl.keyStore=${sonarHome}/cicd-ca.jks -Djavax.net.ssl.keyStorePassword=changeit -Djavax.net.ssl.trustStorePassword=changeit -Dmaven.wagon.http.ssl.insecure=true\""
                                sh "${sonarHome}/bin/sonar-scanner -Dsonar.projectKey=${pom.groupId}.${pom.artifactId} -Dsonar.java.binaries=target"
                            }
                            for (module in pom.modules) {
                                echo "module: ${module}"
                                dir(module) {
                                    
                                    def modulePom = readMavenPom file: 'pom.xml'
                                    if (!modulePom.groupId) {
                                        modulePom.groupId = sh script: 'mvn  help:evaluate -Dexpression=project.groupId -q -DforceStdout', returnStdout: true
                                    }
                                    if (modulePom.packaging == pom ) {
                                        echo "SKIP sonarqube scanning for pom"
                                    } else {
                                        sh "export SONAR_SCANNER_OPTS=\"-Djava.net.debug=all -Djavax.net.ssl.trustStore=${sonarHome}/cicd-ca.jks -Djavax.net.ssl.keyStore=${sonarHome}/cicd-ca.jks -Djavax.net.ssl.keyStorePassword=changeit -Djavax.net.ssl.trustStorePassword=changeit -Dmaven.wagon.http.ssl.insecure=true\""
                                        sh "${sonarHome}/bin/sonar-scanner -Dsonar.projectKey=${pom.groupId}.${modulePom.artifactId} -Dsonar.java.binaries=target"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('deploy to nexus') {
            steps {
                script {
                    sh 'mvn deploy -DskipTests -DaltDeploymentRepository=deploy-to-nexus::default::https://nexus-cicd.smartplay-np.lcsd.hksarg:8443/repository/lcsd-maven2'                    
                    for (module in pom.modules) {
                        echo "module: ${module}"
                        dir(module) {
                            def modulePom = readMavenPom file: 'pom.xml'
                            if (!modulePom.groupId) {
                                modulePom.groupId = sh script: 'mvn  help:evaluate -Dexpression=project.groupId -q -DforceStdout', returnStdout: true
                            }
                            sh 'mvn deploy -DskipTests -DaltDeploymentRepository=deploy-to-nexus::default::https://nexus-cicd.smartplay-np.lcsd.hksarg:8443/repository/lcsd-maven2'
                        }
                    }
                }
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
                            openshift.withProject("${params.NAMESPACE_PREFIX}-${DEV_SUFFIX}"){
                                def bc = openshift.selector("bc/${pom.artifactId}")
                                if (!bc.exists()) {
                                    def bcResult = openshift.raw("new-build  --name ${pom.artifactId} --binary=true --to-docker=true --to=${DOCKER_HOST_ONPRIM}/${DOCKER_GROUP_NAME}/${pom.artifactId}:latest --strategy=source --push-secret docker-cicd --image-stream=cicd-common/redhat-ubi8-openjdk-8:latest")
                                    echo "${bcResult.out}"
                                }
                                def result = openshift.raw("start-build --follow ${pom.artifactId} --from-file=./${pom.artifactId}-${pom.version}.jar")
                                echo "${result}"
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
                script {
                    try {
                        withCredentials([string(credentialsId: 'ACS_TOKEN', variable: 'ROX_API_TOKEN'), string(credentialsId: 'ACS_URL', variable: 'ROX_CENTRAL_ADDRESS')]){
                            sh """
                            roxctl --insecure-skip-tls-verify -e ${ROX_CENTRAL_ADDRESS} image check --image ${DOCKER_HOST_ONPRIM}/${DOCKER_GROUP_NAME}/${pom.artifactId}
                            """
                        }
                    } catch (err) {
                        unstable("ACS Scan ERROR")
                    }
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
                        openshift.withProject("${params.NAMESPACE_PREFIX}-${DEV_SUFFIX}"){
                            def result1 = openshift.tag("--source=docker","${DOCKER_HOST_ONPRIM}/${DOCKER_GROUP_NAME}/${pom.artifactId}", "cicd-common/${pom.artifactId}:latest")
                            echo "${result1.out}"
                            def result2 = openshift.tag("cicd-common/${pom.artifactId}:latest","cicd-common/${pom.artifactId}:${params.RELEASE_VERSION}")
                            echo "${result2.out}"
                            
                        }
                        openshift.withProject('cicd-common'){
                            def istag = openshift.selector("istag/${pom.artifactId}:${params.RELEASE_VERSION}").object()
                            imagereference  =istag.image.dockerImageReference
                            echo "IMAGEREF ${imagereference}"
                        }
                         openshift.withProject('cicd-tools'){
                            def argoProject = openshift.selector("appproject/${params.NAMESPACE_PREFIX}-${pom.artifactId}")
                            if (!argoProject.exists()) {
                                def newArgoProject = [
                                    "apiVersion": "argoproj.io/v1alpha1",
                                    "kind": "AppProject",
                                    "metadata": [
                                        "name": "${params.NAMESPACE_PREFIX}-${pom.artifactId}",
                                        "namespace": "cicd-tools"
                                    ],
                                    "spec": [
                                        "clusterResourceWhitelist" : [
                                            [
                                                "group": "*",
                                                "kind": "*"
                                            ]
                                        ],
                                        "destinations": [
                                            [
                                                "namespace": "*",
                                                "server": "*"
                                            ]
                                        ],
                                        "sourceRepos": [
                                            "*"
                                        ]
                                    ]
                                ]
                                openshift.create(newArgoProject,'--save-config', '--validate' )
                            }
                            def argoApplication = openshift.selector("application/${params.NAMESPACE_PREFIX}-${pom.artifactId}-${DEV_SUFFIX}")
                            if (!argoApplication.exists()) {
                                def newArgoApplication = [
                                    "apiVersion": "argoproj.io/v1alpha1",
                                    "kind": "Application",
                                    "metadata": [
                                        "name": "${params.NAMESPACE_PREFIX}-${pom.artifactId}-${DEV_SUFFIX}",
                                        "namespace": "cicd-tools"
                                    ],
                                    "spec": [
                                        "destination": [
                                            "namespace": "${params.NAMESPACE_PREFIX}-${DEV_SUFFIX}",
                                            "server": "https://kubernetes.default.svc"
                                        ],
                                        "project": "${pom.artifactId}",
                                        "source": [
                                            "path": "environments/dev",
                                            "repoURL": "https://${GITTEA_OPS_HOST}:8443/${params.GITEA_ORGANIZATION}/${env.JOB_BASE_NAME}",
                                            targetRevision: "main"
                                        ],
                                        "syncPolicy": [
                                            "automated": [
                                                "selfHeal": true
                                            ]
                                        ]
                                    ]
                                ]
                                openshift.create(newArgoApplication,'--save-config', '--validate' )
                            } 
                        }
                        dir('git-ops') {
                            withCredentials([sshUserPrivateKey(credentialsId: 'GITEA-APPS-DEPLOY', keyFileVariable: 'keyfile')]) {
                                sh """
                                  rm -rf ${env.JOB_BASE_NAME}
                                  export GIT_SSH_COMMAND="ssh -i ${keyfile} -o IdentitiesOnly=yes -o StrictHostKeyChecking=no" 
                                  git clone 'ssh://git@${GITTEA_OPS_HOST}:${GITTEA_OPS_PORT}/${params.GITEA_ORGANIZATION}/${env.JOB_BASE_NAME}'
                                  cd ${env.JOB_BASE_NAME}/environments/dev
                                  git config user.email "cicd@smartplay-np.lcsd.hksarg"
                                  git config user.name "cicd"
                                  git status
                                  kustomize edit set image IMAGE:VERSION=${imagereference}
                                  git add ./kustomization.yaml
                                  git commit -m "CI Image Update"
                                  git tag -a ${params.RELEASE_VERSION} -m "CI Image Update" --force
                                  git push origin main
                                  git push --tags --force
                                """
                            }
                            
                        }
                        openshift.withProject('cicd-tools'){
                            def resultLogin = openshift.raw('rsh argocd-application-controller-0  argocd --config /tmp/config  login  --insecure --core')
                            echo "${resultLogin.out}"
                            
                            try {
                                def resultSyn = openshift.raw("rsh argocd-application-controller-0  argocd --config /tmp/config app sync \"cicd-tools/${params.NAMESPACE_PREFIX}-${pom.artifactId}-${DEV_SUFFIX}\"")
                                echo "${resultSyn.out}"
                            } catch (Exception e){
                                echo "Exception "+e.toString() +" expectred"
                            }
                            sh 'sleep 3'
                            def resultWait = openshift.raw("rsh argocd-application-controller-0  argocd --config /tmp/config app wait \"cicd-tools/${params.NAMESPACE_PREFIX}-${pom.artifactId}-${DEV_SUFFIX}\"")
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
                    //create projects and deployment keys if not there
                    openshift.withCluster('alicloud-nonprod'){
                        openshift.withProject('cicd-tools'){
                            def pod = openshift.selector('pod',['app':'gitea-gitea']).object()
                            echo pod.metadata.name
                            withCredentials([string(credentialsId: 'ALICLOUD-NONPROD-GITEA-OPS-API-TOKEN', variable: 'ALICLOUDNONPRODGITEAOPSAPITOKEN')]) {
                                try {
                                    def thisResult = openshift.raw("exec ${pod.metadata.name} -- curl -k -H \"Authorization: token  ${ALICLOUDNONPRODGITEAOPSAPITOKEN}\" -H \"content-type: application/json\"  -XPOST \"http://localhost:3000/api/v1/orgs/demo/repos\" -d '{\"name\": \"${env.JOB_BASE_NAME}\"}' ")
                                    echo ${thisResult.out}
                                } catch (ignore) {}
                                try {
                                    def thisResult = openshift.raw("exec ${pod.metadata.name} -- curl -k -H \"Authorization: token  ${ALICLOUDNONPRODGITEAOPSAPITOKEN}\" -H \"content-type: application/json\" -XPOST \"http://localhost:3000/api/v1/orgs/demo/repos/${env.JOB_BASE_NAME}/keys\" -d '{\"key\":\"ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDKbyYrTQnsQ8Yb+9gebzQ+yPLcasVroAXWwavSzZwimT9YDirIeM9YjGlRUTla+qiEMORE3ZcUlIV5/reR5HoxrsK05TIwJgxq89+Q9uCjMM6sSl1p0f5ngfgSfqk3GZUUktgexfrHClbpidLyyS73nYl2CqmaSvyi2B0Xdc+KsEDZpvXPfRRQFWbYjCwvQu8Q5AQKPTMCN44hsnqXHayBw1k/3orbb5ipV+yHr28DqRk/sZY28zxJiC2UrCHkXGdow1ISxhSFJTLDlsYbsTM/5uA97+t66lBkqkFf1RQZvy0qdtNni2f0BqJKV3Tv8HOMGHFOv8pYT7yeZ/84yR2Ok1Ep4B3GyqTmsGiWxLnExtGTBscUc6PXwidojpoxvUse0tET0+nzktrVeNF72tCurF1Y4jmpVifDcLnJs7g7DbxTtSJQTGqVSmJRrXYqpBQUOSROZYx5gMBXZbCySbSyBXE7B+UQxUtG6IenZq7cpUpx2rsyB0C4Q+YQhfMVWKBd1QAdbBd1dyLOmgdWaMYvOvJxpkkwVAewDLN2hTlIjpbkWGptj5erWCjPEOnPQ6E0/mwAb0osDpwaJznoUXTFO2gn/d0peEA4yqM8R4ewnvd/LMGDbf4+PzkigvmgE06YIe6HzaV31XSyNnauxUeJ7UETMFHB42dv2jeaSR7q3Q== noname\",\"title\":\"cicd\",\"read_only\":false}' ")
                                    echo ${thisResult.out}
                                } catch (ignore) {}
                            }
                            withCredentials([sshUserPrivateKey(credentialsId: 'GITEA-APPS-DEPLOY', keyFileVariable: 'keyfile')]) {
                                dir("alicloud-gitops") {
                                    sh """
                                        rm -rf ${env.JOB_BASE_NAME}
                                        export GIT_SSH_COMMAND="ssh -i ${keyfile} -o IdentitiesOnly=yes -o StrictHostKeyChecking=no" 

                                        git clone 'ssh://git@${GITTEA_OPS_HOST}:${GITTEA_OPS_PORT}/${params.GITEA_ORGANIZATION}/${env.JOB_BASE_NAME}'
                                        rm -rf ${env.JOB_BASE_NAME}/.git
                                    """
                                    def thisResult = openshift.raw("exec ${pod.metadata.name} -- mkdir /tmp/${env.JOB_BASE_NAME}-${env.BUILD_NUMBER}")
                                    echo ${thisResult.out}
                                    thisResult = openshift.raw("cp ./${env.JOB_BASE_NAME} ${pod.metadata.name}:/tmp/${env.JOB_BASE_NAME}-${env.BUILD_NUMBER}/source-dev")
                                    echo ${thisResult.out}                                    
                                }
                            }
                        }
                    }
                    
                    withCredentials([file(credentialsId: 'DOCKER-CICD-AUTH', variable: 'DOCKERCICDAUTH'), 
                                 file(credentialsId: 'ALICLOUD-NP-REGISTRY-AUTH', variable: 'ALICLOUDNPREGISTRYAUTH')]) {
                        sh """
                        podman pull --authfile ${DOCKERCICDAUTH} --tls-verify=false  ${imagereference}
                        podman tag ${imagereference} ${DOCKER_HOST_ALICLOUD}/${DOCKER_GROUP_ALICLOUD_NAME}/${pom.artifactId}:${params.RELEASE_VERSION}
                        podman push --authfile ${ALICLOUDNPREGISTRYAUTH} --tls-verify=false ${DOCKER_HOST_ALICLOUD}/${DOCKER_GROUP_ALICLOUD_NAME}/${pom.artifactId}:${params.RELEASE_VERSION}
                        podman rmi ${DOCKER_HOST_ALICLOUD}/${DOCKER_GROUP_ALICLOUD_NAME}/${pom.artifactId}:${params.RELEASE_VERSION}
                        podman rmi ${imagereference}
                        """
                                 }
                    openshift.withCluster('alicloud-nonprod'){
                        openshift.withProject('cicd-tools'){
                            def argoProject = openshift.selector("appproject/${params.NAMESPACE_PREFIX}-${pom.artifactId}")
                            if (!argoProject.exists()) {
                                def newArgoProject = [
                                    "apiVersion": "argoproj.io/v1alpha1",
                                    "kind": "AppProject",
                                    "metadata": [
                                        "name": "${params.NAMESPACE_PREFIX}-${pom.artifactId}",
                                        "namespace": "cicd-tools"
                                    ],
                                    "spec": [
                                        "clusterResourceWhitelist" : [
                                            [
                                                "group": "*",
                                                "kind": "*"
                                            ]
                                        ],
                                        "destinations": [
                                            [
                                                "namespace": "*",
                                                "server": "*"
                                            ]
                                        ],
                                        "sourceRepos": [
                                            "*"
                                        ]
                                    ]
                                ]
                                openshift.create(newArgoProject,'--save-config', '--validate' )
                            }
                            def argoApplication = openshift.selector("application/${params.NAMESPACE_PREFIX}-${pom.artifactId}-${DEV_SUFFIX}")
                            if (!argoApplication.exists()) {
                                def newArgoApplication = [
                                    "apiVersion": "argoproj.io/v1alpha1",
                                    "kind": "Application",
                                    "metadata": [
                                        "name": "${params.NAMESPACE_PREFIX}-${pom.artifactId}-${DEV_SUFFIX}",
                                        "namespace": "cicd-tools"
                                    ],
                                    "spec": [
                                        "destination": [
                                            "namespace": "${params.NAMESPACE_PREFIX}-${DEV_SUFFIX}",
                                            "server": "https://kubernetes.default.svc"
                                        ],
                                        "project": "${pom.artifactId}",
                                        "source": [
                                            "path": "environments/dev",
                                            "repoURL": "https://${GITTEA_OPS_ALICLOUD_HOST}:3000/${params.GITEA_ORGANIZATION}/${env.JOB_BASE_NAME}",
                                            targetRevision: "main"
                                        ],
                                        "syncPolicy": [
                                            "automated": [
                                                "selfHeal": true
                                            ]
                                        ]
                                    ]
                                ]
                                openshift.create(newArgoApplication,'--save-config', '--validate' )
                            }
                        } 
                        openshift.withProject('cicd-tools'){
                            withCredentials([sshUserPrivateKey(credentialsId: 'GITEA-APPS-DEPLOY', keyFileVariable: 'keyfile')]) {
                                writeFile file: "${env.JOB_BASE_NAME}-${env.BUILD_NUMBER}-gitcommand.txt", text: """
                                    rm -rf /tmp/git-ops
                                    mkdir -p /tmp/git-ops
                                    cd /tmp/git-ops
                                    export GIT_SSH_COMMAND="ssh -i /tmp/keyfile.txt -o IdentitiesOnly=yes -o StrictHostKeyChecking=no" 
                                    git clone 'ssh://git@${GITTEA_OPS_ALICLOUD_HOST}:${GITTEA_OPS_ALICLOUD_PORT}/${params.GITEA_ORGANIZATION}/${env.JOB_BASE_NAME}'
                                    cp -rf /tmp/${env.JOB_BASE_NAME}-${env.BUILD_NUMBER}/source-dev/* ${env.JOB_BASE_NAME}
                                    cd ${env.JOB_BASE_NAME}/environments/dev
                                    git config user.email "cicd@smartplay-np.lcsd.hksarg"
                                    git config user.name "cicd"
                                    git status
                                    kustomize edit set image IMAGE:VERSION=${DOCKER_HOST_ALICLOUD}/${DOCKER_GROUP_ALICLOUD_NAME}/${pom.artifactId}:${params.RELEASE_VERSION}
                                    git add ./kustomization.yaml
                                    git commit -m "CI Image Update"
                                    git tag -a ${params.RELEASE_VERSION} -m "CI Image Update" --force
                                    git push origin main
                                    git push --tags --force
                                    rm -f /tmp/${env.JOB_BASE_NAME}-${env.BUILD_NUMBER}-keyfile.txt
                                    rm -f /tmp/${env.JOB_BASE_NAME}-${env.BUILD_NUMBER}-gitcommand.txt
                                """
                                def pod = openshift.selector('pod',['app.kubernetes.io/name':'argocd-server']).object()
                                echo pod.metadata.name
                                def resultCp = openshift.raw("cp ${keyfile} ${pod.metadata.name}:/tmp/${env.JOB_BASE_NAME}-${env.BUILD_NUMBER}-keyfile.txt")
                                echo "${resultCp.out}"
                                def resultCp2 = openshift.raw("cp gitcommand.txt ${pod.metadata.name}:/tmp/${env.JOB_BASE_NAME}-${env.BUILD_NUMBER}-gitcommand.txt")
                                echo "${resultCp2.out}"
                                def resultExec = openshift.raw("exec ${pod.metadata.name} -- bash /tmp/${env.JOB_BASE_NAME}-${env.BUILD_NUMBER}-gitcommand.txt")
                                echo "${resultExec.out}"
                                def resultLogin = openshift.raw("rsh ${pod.metadata.name}  argocd --config /tmp/config  login  --insecure --core")
                                echo "${resultLogin.out}"
                                
                                try {
                                    def resultSyn = openshift.raw("rsh ${pod.metadata.name}  argocd --config /tmp/config app sync \"cicd-tools/${params.NAMESPACE_PREFIX}-${pom.artifactId}-dev\"")
                                    echo "${resultSyn.out}"
                                } catch (Exception e){
                                    echo "Exception "+e.toString() +" expectred"
                                }
                                sh 'sleep 3'
                                def resultWait = openshift.raw("rsh ${pod.metadata.name}   argocd --config /tmp/config app wait \"cicd-tools/${params.NAMESPACE_PREFIX}-${pom.artifactId}-dev\"")
                                echo "${resultWait.out}"
                            }
                        }  
                    }
                }
            }
        }
    }
}

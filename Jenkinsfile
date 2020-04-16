@Library('retort-lib') _

def label = "jenkins-${UUID.randomUUID().toString()}"
 
def USERID = 'admin'
def DOCKER_IMAGE = 'az-cicd-dev/petshop'
def TARGET_DOCKER_IMAGE = 'skt-hcp/petshop'
env.DEFAULT_NAMESPACE = 'skcc-admin'
def DEV_VERSION = SRC_IMAGE_TAG
def PROD_VERSION = 'prod'
def VERSION = "PRD-${BUILD_NUMBER}"
def PUBLIC_REGISTRY='068371015812.dkr.ecr.ap-northeast-2.amazonaws.com'

timestamps {
podTemplate(label:label,
        serviceAccount: "zcp-system-sa-${USERID}",
        containers: [
            containerTemplate(name: 'docker', image: 'earth1223/docker:19-dind-bmt', ttyEnabled: true, command: 'dockerd-entrypoint.sh', privileged: true, alwaysPullImage: true),
            containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl', ttyEnabled: true, command: 'cat'),
        ],
        volumes: [
            persistentVolumeClaim(mountPath: '/home/jenkins/.docker', claimName: 'zcp-jenkins-docker-sam-icp4a')
        ]) {
     
        node(label) {
            stage('CHECKOUT') {
                git 'https://vup-git.cloudzcp.io/sktbmt/sam-zcp-lab.git'
            }
            
            stage('ANCHORE EVALUATION') {
                def imageLine = "${INTERNAL_REGISTRY_CREDENTIALS}/${DOCKER_IMAGE}:${DEV_VERSION}"
                writeFile file: 'anchore_images', text: imageLine
                try {
                    anchore name: 'anchore_images', engineRetries: '3000', policyBundleId: 'anchore_skt_hcp_bmt'
                } catch (e) {
                    echo 'keep going..'
                }
            }
     
            stage('PULL IMAGE') {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: 'INTERNAL_REGISTRY_CREDENTIALS', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                        sh """
                            docker login -u '${USERNAME}' -p '${PASSWORD}' ${INTERNAL_REGISTRY}
                            docker pull ${INTERNAL_REGISTRY}/${DOCKER_IMAGE}:${DEV_VERSION}
                            docker logout ${INTERNAL_REGISTRY}
                        """
                    }
                }
            }
            
            stage('RETAG IMAGE') {
                container('docker') {
                    sh "docker tag ${INTERNAL_REGISTRY}/${DOCKER_IMAGE}:${DEV_VERSION} ${HARBOR_REGISTRY}/${TARGET_DOCKER_IMAGE}:${VERSION}"
                }
            }
            
            stage('SIGN AND PUSH IMAGE') {
                withCredentials([usernamePassword(credentialsId: 'HARBOR_CREDENTIALS', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME'),
                    string(credentialsId: 'DOCKER_CONTENT_TRUST_ROOT_PASSPHRASE', variable: 'TRUST_ROOT_PASSPHRASE'),
                    string(credentialsId: 'DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE', variable: 'TRUST_REPOSITORY_PASSPHRASE')]) {
                    container('docker') {
                        sh "docker login -u '${USERNAME}' -p '${PASSWORD}' ${HARBOR_REGISTRY}"
                        sh """
                            export DOCKER_CONTENT_TRUST=1
                            export DOCKER_CONTENT_TRUST_SERVER=${NOTARY_SERVER}
                            export DOCKER_CONTENT_TRUST_ROOT_PASSPHRASE=${TRUST_ROOT_PASSPHRASE}
                            export DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE=${TRUST_REPOSITORY_PASSPHRASE}
                            
                            docker push ${HARBOR_REGISTRY}/${TARGET_DOCKER_IMAGE}:${VERSION}
                            docker logout ${HARBOR_REGISTRY}
                        """
                        sh "docker logout ${HARBOR_REGISTRY}"
                    }
                }
            }
         
            stage('DEPLOY') {
                container('kubectl') {
                    cluster {
                        echo "Work in ${it.cluster}"
                        sh "kubectl get po -n ${it.namespace}"
                        kubeCmd.apply file: 'k8s/service.yaml', namespace: it.namespace
                        yaml.update file: 'k8s/deployment.yaml', update: ['.spec.template.spec.containers[0].image': "${PUBLIC_REGISTRY}/${TARGET_DOCKER_IMAGE}:${VERSION}"]
                        kubeCmd.apply file: 'k8s/deployment.yaml', namespace: it.namespace, wait: 300
                    }
                }
            }
        }
    }
}
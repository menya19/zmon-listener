@Library('retort-lib') _
def label = "jenkins-${UUID.randomUUID().toString()}"

podTemplate(label:label,
    containers: [
        containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:v2.9.1', ttyEnabled: true, command: 'cat')
    ],
    volumes: [
        configMapVolume(mountPath: '/var/config/zmon', configMapName: 'zmon-kube-config')
    ]) {

    node(label) {
        stage('SOURCE CHECKOUT') {
            def repo = checkout scm
        }

        stage('LISTENER') {
            container('helm') {
              withEnv(['KUBECONFIG=/var/config/zmon/kube-config-zmon.yml']) {
                withFolderProperties{
                  yaml.update file: 'values.yaml', update: ['.config.inputs[0].http_listener.basic_username': "${TENANT}", '.config.inputs[0].http_listener.basic_password': "${PASSWORD}", '.config.outputs[0].kafka.brokers[0]':"${env.KAFKA_BROKER_1}", '.config.outputs[0].kafka.brokers[1]':"${env.KAFKA_BROKER_2}", '.config.outputs[0].kafka.brokers[2]':"${env.KAFKA_BROKER_3}", '.config.outputs[0].kafka.topic':"telegraf__${TENANT}"]
                }
                sh "cat values.yaml"
                sh "helm install --namespace ${TENANT} --name listener-${TENANT} -f values.yaml ."
              }
            }
        }

        stage('LISTENER INGRESS'){
          container('kubectl') {
            withEnv(['KUBECONFIG=/var/config/zmon/kube-config-zmon.yml']) {
              sh "kubectl get secret star-zmon-cloud --namespace zmon --export -o yaml | kubectl apply --namespace=${TENANT} -f -"
              yaml.update file: 'ingress.yaml', update: ['.metadata.name': "${TENANT}-ingress", '.metadata.namespace': "${TENANT}", '.spec.tls[0].hosts[0]': "${TENANT}.zmon.cloud", '.spec.rules[0].host': "${TENANT}.zmon.cloud", '.spec.rules[0].http.paths[0].backend.serviceName': "listener-${TENANT}"]
              sh "cat ingress.yaml"
              kubeCmd.apply file: 'ingress.yaml', namespace: TENANT
            }
          }
        }

    }
}

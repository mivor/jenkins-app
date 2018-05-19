node {

    def appName = 'gceme'
    def feSvcName = "${appName}-frontend"
    def imageTag = "mivor/${appName}:${env.BRANCH_NAME}-${env.BUILD_NUMBER}"

    checkout scm

    stage('Preparation') {
        sh("curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.9.7/bin/linux/amd64/kubectl")
        sh("chmod +x ./kubectl && mv kubectl /usr/local/sbin")
        sh("printenv")
    }

    stage('Build image') {
        sh("docker build -t ${imageTag} .")
    }

    stage('Run Go tests') {
        sh("docker run ${imageTag} go test")
    }

    withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
        stage('Push image to registry') {
            sh("docker login -u ${USER} -p ${PASS}")
            sh("docker push ${imageTag}")
        }
    }

    stage("Deploy Application") {
        switch (env.BRANCH_NAME) {
            // Roll out to canary environment
            case "canary":
                // Change deployed image in canary to the one we just built
                sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
                sh("sed -i.bak 's#mivor/gceme:canary-7#${imageTag}#' ./k8s/canary/*.yaml")
                sh("kubectl --namespace=production apply -f k8s/services/")
                sh("kubectl --namespace=canary apply -f k8s/canary/")
                sh("sleep 4")
                sh("export PUBLIC_IP=$$(kubectl get -o jsonpath='{.status.loadBalancer.ingress[0].ip}'  --namespace=production services gceme-frontend)")
                sh("echo http://$PUBLIC_IP")
                break

            // Roll out to production
            case "master":
                // Change deployed image in canary to the one we just built
                sh("kubectl get ns production || kubectl create ns production")
                sh("sed -i.bak 's#mivor/gceme:master-1#${imageTag}#' ./k8s/production/*.yaml")
                sh("kubectl --namespace=production apply -f k8s/services/")
                sh("kubectl --namespace=production apply -f k8s/production/")
                sh("sleep 4")
                sh("export PUBLIC_IP=$$(kubectl get -o jsonpath='{.status.loadBalancer.ingress[0].ip}'  --namespace=production services gceme-frontend)")
                sh("echo http://$$PUBLIC_IP")
                break

            // Roll out a dev environment
            default:
                // Create namespace if it doesn't exist
                sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
                // Don't use public load balancing for development branches
                sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/frontend.yaml")
                sh("sed -i.bak 's#mivor/gceme:dev-1#${imageTag}#' ./k8s/dev/*.yaml")
                sh("kubectl --namespace=production apply -f k8s/services/")
                sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/dev/")
                echo 'To access your environment run `kubectl proxy`'
                echo "Then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/${feSvcName}:80/"
        }
    }
}

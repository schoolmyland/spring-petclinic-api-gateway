pipeline {
    environment {
        DOCKER_ID = "poissonchat13"
        DOCKER_IMAGE = "spring-petclinic-api-gateway"
        DOCKER_TAG = "1.v.${BUILD_ID}"
        CLIENT_LIST = "rouge,bleu,violet"

    }
    agent any

    stages {
        stage('Maven build test') {
            steps {
                script {
                    sh '''
                    ./mvnw clean package
                    sleep 15
                    '''
                }
            }
        }
        stage('Docker Build Dev') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    sh '''
                    docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG-dev .
                    sleep 10
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    docker tag $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG-dev $DOCKER_ID/$DOCKER_IMAGE:latest-dev
                    docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG-dev
                    docker push $DOCKER_ID/$DOCKER_IMAGE:latest-dev
                    '''
                }
            }
        }
        stage('Deploiement Developpement') {
            environment {
                KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
            steps {
                script {
                    sh '''
                    cp -r /opt/helm/* ./
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    helm upgrade --install app spring-pet-clinic-litecloud --values=./spring-pet-clinic-litecloud/value.yaml
                    sleep 120 
                    '''
                }
            }
        }
        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                    curl localhost:30105
                    '''
                }
            }
        }
        stage('Demontage Env Dev') {
            environment {
                KUBECONFIG = credentials("config") // we retrieve  kubeconfig f>
            }
            steps {
                script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    helm uninstall app 
                    '''
                }
            }
        }
        stage('Recup css and transform') {
            steps {
                script {
                    sh '''
                    cp ./src/main/less/header.less /opt/custom/rouge/
                    cp ./src/main/less/petclinic.less /opt/custom/rouge/
                    sed -i s/#6db33f/#fe2e2e/ /opt/custom/rouge/header.less
                    sed -i s/#6db33f/#fe2e2e/ /opt/custom/rouge/petclinic.less
                    cp ./src/main/less/header.less /opt/custom/bleu/
                    cp ./src/main/less/petclinic.less /opt/custom/bleu/
                    sed -i s/#6db33f/#2e2efe/ /opt/custom/bleu/header.less
                    sed -i s/#6db33f/#2e2efe/ /opt/custom/bleu/petclinic.less
                    cp ./src/main/less/header.less /opt/custom/violet/
                    cp ./src/main/less/petclinic.less /opt/custom/violet/
                    sed -i s/#6db33f/#ac58fa/ /opt/custom/violet/header.less
                    sed -i s/#6db33f/#ac58fa/ /opt/custom/violet/petclinic.less
                    '''
                }
            }
        }
        stage('Customisation build image') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    def clients = CLIENT_LIST.split(",")
                    for (client in clients) {
                        echo "Processing client: ${client}"
                        sh """
                        cp /opt/custom/${client}/img/* ./src/main/resources/static/images/
                        cp /opt/custom/${client}/*.less ./src/main/less/
                        ./mvnw clean package
                        sleep 15
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG-qa-${client} .
                        sleep 10
                        docker login -u $DOCKER_ID -p $DOCKER_PASS
                        docker tag $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG-qa-${client} $DOCKER_ID/$DOCKER_IMAGE:latest-qa-${client}
                        docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG-qa-${client}
                        docker push $DOCKER_ID/$DOCKER_IMAGE:latest-qa-${client}
                        sleep 10
                        """
                    }
                }
            }
        }
        stage('Deploiement QA par client') {
            environment {
                KUBECONFIG = credentials("config") // we retrieve  kubeconfig f>
            }
            steps {
                script {
                    def clients = CLIENT_LIST.split(",")
                    for (client in clients) {
                        sh  """
                        rm -Rf .kube
                        mkdir .kube
                        ls
                        cat $KUBECONFIG > .kube/config
                        helm upgrade --install app spring-pet-clinic-litecloud-dns --values=./spring-pet-clinic-litecloud-dns/${client}-value.yaml
                        sleep 60 
                        """
                    }
                }
            }
        }
        stage('Test access client') {
            steps {
                script {
                    sh '''
                    curl https://rouge.pet-clinic-tpr.cloudns.ph
                    curl https://bleu.pet-clinic-tpr.cloudns.ph
                    curl https://violet.pet-clinic-tpr.cloudns.ph
                    '''
                }
            }
        }
        stage('notification client') {
            steps {
                script {
                    sh '''

                    '''
                }
            }
        }
    }
}

pipeline {
    agent any
    stages {
        stage('Mise en place des variables') {
            steps {
                script {
                    VERSION_MAJEUR = sh(script: 'head -n 5 ./README.md | tail -n 1', returnStdout: true).trim()
                    env.DOCKER_TAG = "${VERSION_MAJEUR}.${BUILD_ID}"
                    def jobDirName = env.JOB_NAME
                    def splitDir = jobDirName.split('/') 
                    def jobName = splitDir[1]
                    def splitParts = jobName.split('_')
                    def prefix = "spring-petclinic-"
                    env.JMETER_TAG = splitParts[0]
                    env.DOCKER_IMAGE = splitParts[1]
                    env.SERVICE_NAME =  env.DOCKER_IMAGE.replaceFirst(prefix, '')
                }
            }
        }
        stage('Docker Build Dev') {
            steps {
                sh '''
                docker build -t localhost:5000/$DOCKER_IMAGE:latest .
                docker push localhost:5000/$DOCKER_IMAGE:latest 
                '''
            }
        }
        stage('Deploiement Env Dev') {
            environment {
                KUBECONFIG = credentials("confkub")
                BDD_PASS = credentials("bdd_pass_dev")
            }
            steps {
                sh '''
                $JENK_TOOLBOX/ctrl/checkNamespaceUse.sh developpement
                cp -r ${JENK_TOOLBOX}/helm/* ./
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                helm install petclinic-dev petclinic-dev --values=./petclinic-dev/value.yaml --set $JMETER_TAG.repo=localhost:5000 --set petclinic.bdpwd=$BDD_PASS
                sleep 180 
                '''
            }
        }
        stage('Test Acceptance') {
            steps {
                script {
                sh '$JENK_TOOLBOX/ctrl/checkpod.sh developpement'
                }
            }
            post {
                failure {
                    sh 'helm uninstall petclinic-dev'
                }
            }
        }
        stage('Test Performance Jmeter') {
            steps {
                sh '''
                echo "$DOCKER_IMAGE:$DOCKER_TAG $(date +"%Y-%m-%d-%H-%M")" >> $JENK_TOOLBOX/logs/jmeter-commit-version.log
                $JENK_TOOLBOX/apache-jmeter/bin/jmeter -n -t $JENK_TOOLBOX/test/petclinic_test_plan.jmx -l $JENK_TOOLBOX/test/petclinic_result_test.jtl
                '''
            }
        }
        stage('Build Jmeter pour report') {
            environment {
                API_TOKEN = credentials("API_TOKEN")
            }
            steps {
                script {
                    def displayName = "${JMETER_TAG}-${DOCKER_TAG}"
                    def description = "Build trigger by the job ${JOB_NAME}  On the microservice ${DOCKER_IMAGE} "

                    def buildResult = build job: 'jmeter-perf-central', 
                                           wait: true, 
                                           propagate: false

                    def jobUrl = buildResult.getAbsoluteUrl()

                    echo "Job URL: ${jobUrl}"
                    
                    sh """
                    curl -X POST -u ${API_TOKEN} -F 'json={"displayName":"${displayName}","description":"${description}"}' \
                    '${jobUrl}configSubmit'
                    """
                }
            }
        }
        stage('Push Image pour prod') {
            environment {
                DOCKER_ID = credentials("DOCKER_HUB_ID")
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker tag localhost:5000/$DOCKER_IMAGE:latest $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                docker tag $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG $DOCKER_ID/$DOCKER_IMAGE:latest
                docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                docker push $DOCKER_ID/$DOCKER_IMAGE:latest
                '''
            }
        }
        stage('Demontage Env Dev') {
            environment {
                KUBECONFIG = credentials("confkub")
            }
            steps {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                helm uninstall petclinic-dev
                '''
            }
        }
	stage('Lancement des Personnalisations par clients') {
    	environment {
        	API_TOKEN = credentials("API_TOKEN")
    	}
    	steps {
        	script {
            		def csvFile = readFile(env.CSV_FILE)
            		def csvLines = csvFile.split('\n')
            		csvLines.each { line ->
                		def params = line.split(';')
                		def clientName = params[0].trim()
                		def hexCode = params[1].trim()
                		def nodePort = params[2].trim()
                		def displayName = "${clientName}-${JMETER_TAG}-${DOCKER_TAG}"
                		def description = "Build trigger by the job ${JOB_NAME}  On the microservice ${DOCKER_IMAGE}-${DOCKER_TAG} for the customer ${clientName}"
                		echo "Job custom client launch for : clientName=${clientName} Parameters hexCode=${hexCode}, nodePort=${nodePort}"
                		def triggeredBuild = build job: 'api-gateway-client', parameters: [
                    			string(name: 'CLIENT_NAME', value: clientName),
                    			string(name: 'HEX_CODE', value: hexCode),
                    			string(name: 'NODE_PORT', value: nodePort),
                    			string(name: 'DOCKER_TAG', value: DOCKER_TAG),
					string(name: 'DISPLAY_NAME', value: displayName),
                    			string(name: 'DESCRIPTION', value: description)
                		], wait: false
            	    }
		}
	    }
        }
    }
}

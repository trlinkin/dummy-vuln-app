pipeline {
    agent {
       kubernetes {
           yaml """
apiVersion: v1 
kind: Pod 
metadata: 
    name: dind
    annotations:
      container.apparmor.security.beta.kubernetes.io/dind: unconfined
      container.seccomp.security.alpha.kubernetes.io/dind: unconfined
spec: 
    containers: 
      - name: dind
        image: docker:dind
        securityContext:
          privileged: true
        tty: true
        volumeMounts:
        - name: var-run
          mountPath: /var/run
      - name: jnlp
        securityContext:
          runAsUser: 0
          fsGroup: 0
        volumeMounts:
        - name: var-run
          mountPath: /var/run
        
    volumes:
    - emptyDir: {}
      name: var-run
"""
       }
   }

    parameters { 
        string(name: 'DOCKER_REPOSITORY', defaultValue: 'sysdigcicd/cronagent', description: 'Name of the image to be built (e.g.: sysdiglabs/dummy-vuln-app)') 
    }
    
    environment {
        DOCKER = credentials('docker-repository-credentials')
        SYSDIG_API_KEY = credentials('sysdig-secure-api-credentials')
    }
    
    stages {
        stage('Checkout') {
            steps {
                container("dind") {
                    checkout scm
                }
            }
        }
        stage('Build Image') {
            steps {
                container("dind") {
                    sh "docker build -f Dockerfile -t ${params.DOCKER_REPOSITORY} ."
                    sh "echo ${params.DOCKER_REPOSITORY} > sysdig_secure_images"
                }
            }
        }
        stage('Scanning Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'sysdig-secure-api-credentials', passwordVariable: 'SECURE_API_TOKEN', usernameVariable: '')]) {
                    sh '''
                        VERSION=$(curl -L -s https://download.sysdig.com/scanning/inlinescan/latest_version.txt)
                        curl -LO "https://download.sysdig.com/scanning/inlinescan/inlinescan_${VERSION}_linux_amd64"
                        chmod +x ./inlinescan_${VERSION}_linux_amd64
                        ./inlinescan_${VERSION}_linux_amd64 --apiurl https://secure.sysdig.com ./sysdig_secure_images --policy sysdig-best-practices -u
                    '''
                }
            }
        }
   }
}

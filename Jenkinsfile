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
        image: jenkins/inbound-agent:4.6-1
        
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
                    sh "docker save ${params.DOCKER_REPOSITORY} -o artifact.tar"
                    sh "ls -lah"
                }
            }
        }
        stage('Scanning Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'sysdig-secure-api-credentials', passwordVariable: 'SECURE_API_TOKEN', usernameVariable: '')]) {
                    container("jnlp") {
                        sh '''
                            VERSION=$(curl -L -s https://download.sysdig.com/scanning/inlinescan/latest_version.txt)
                            /usr/bin/curl -LO "https://download.sysdig.com/scanning/inlinescan/inlinescan_${VERSION}_linux_amd64"
                            chmod +x ./inlinescan_${VERSION}_linux_amd64
                            ls -lah
                            ./inlinescan_${VERSION}_linux_amd64 --apiurl https://secure.sysdig.com file://artifact.tar --policy sysdig-best-practices -u
                        '''
                    }
                }
            }
        }
   }
}

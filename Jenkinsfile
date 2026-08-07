pipeline {
    agent {
        kubernetes {
            label 'ansible-argocd-deploy'
            defaultContainer 'ansible'
            yaml '''
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-agent
  containers:
    - name: ansible
      image: docker.io/ansible/ansible-core:2.17
      imagePullPolicy: Always
      command:
        - cat
      tty: true
      env:
        - name: HOME
          value: /home/jenkins
        - name: KUBECONFIG
          value: /home/jenkins/.kube/config
      volumeMounts:
        - name: workspace
          mountPath: /home/jenkins/agent
  volumes:
    - name: workspace
      emptyDir: {}
'''
        }
    }

    parameters {
        string(
            name: 'ARGOCD_NAMESPACE',
            defaultValue: 'argocd',
            description: 'Namespace to deploy ArgoCD (use unique name for ephemeral, e.g., argocd-test-${BUILD_NUMBER})'
        )
        string(
            name: 'ARGOCD_RELEASE_NAME',
            defaultValue: 'argocd',
            description: 'Helm release name'
        )
        string(
            name: 'ARGOCD_CHART_VERSION',
            defaultValue: '7.7.7',
            description: 'ArgoCD Helm chart version'
        )
        booleanParam(
            name: 'EPHEMERAL_CLEANUP',
            defaultValue: false,
            description: 'Delete namespace after testing'
        )
        choice(
            name: 'TARGET_CLUSTER',
            choices: ['diskless-k8s', 'k3s-dev'],
            description: 'Target Kubernetes cluster'
        )
    }

    environment {
        ANSIBLE_DIR = "${WORKSPACE}"
        PATH = "/home/jenkins/.local/bin:/usr/local/bin:/usr/bin:/bin"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        skipDefaultCheckout false
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup Ansible Config') {
            steps {
                container('ansible') {
                    writeFile file: 'ansible.cfg', text: '''
[defaults]
inventory = inventories/dev
roles_path = roles
collections_path = collections
retry_files_enabled = False
gathering = smart
host_key_checking = False
interpreter_python = auto_silent
display_skipped_hosts = False
deprecation_warnings = False

[privilege_escalation]
become = False

[ssh_connection]
pipelining = True
'''
                }
            }
        }

        stage('Setup Kubeconfig') {
            steps {
                container('ansible') {
                    withCredentials([file(credentialsId: 'jenkins-kubeconfig-${TARGET_CLUSTER}', variable: 'KUBECONFIG_SECRET')]) {
                        sh '''
                            mkdir -p /home/jenkins/.kube
                            cp ${KUBECONFIG_SECRET} /home/jenkins/.kube/config
                            chmod 600 /home/jenkins/.kube/config
                        '''
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                container('ansible') {
                    sh '''
                        pip install --upgrade pip
                        pip install ansible ansible-lint kubernetes netaddr
                        ansible-galaxy collection install -r requirements.yml || true
                    '''
                }
            }
        }

        stage('Lint Playbooks') {
            steps {
                container('ansible') {
                    sh 'ansible-lint playbooks/*.yml'
                }
            }
        }

        stage('Syntax Check') {
            steps {
                container('ansible') {
                    sh '''
                        ansible-playbook --syntax-check playbooks/playbook-argocd.yml -i inventories/dev/
                    '''
                }
            }
        }

        stage('Dry Run') {
            steps {
                container('ansible') {
                    sh '''
                        ansible-playbook playbooks/playbook-argocd.yml \
                            -i inventories/dev/ \
                            --check \
                            -e "argocd_namespace=${params.ARGOCD_NAMESPACE}" \
                            -e "argocd_release_name=${params.ARGOCD_RELEASE_NAME}" \
                            -e "argocd_chart_version=${params.ARGOCD_CHART_VERSION}"
                    '''
                }
            }
        }

        stage('Deploy ArgoCD') {
            steps {
                container('ansible') {
                    script {
                        if (params.EPHEMERAL_CLEANUP) {
                            input message: "Deploy ArgoCD to namespace '${params.ARGOCD_NAMESPACE}'? Will be cleaned up after.", ok: 'Deploy'
                        } else {
                            input message: "Deploy ArgoCD to namespace '${params.ARGOCD_NAMESPACE}'?", ok: 'Deploy'
                        }
                    }
                    sh '''
                        ansible-playbook playbooks/playbook-argocd.yml \
                            -i inventories/dev/ \
                            -e "argocd_namespace=${params.ARGOCD_NAMESPACE}" \
                            -e "argocd_release_name=${params.ARGOCD_RELEASE_NAME}" \
                            -e "argocd_chart_version=${params.ARGOCD_CHART_VERSION}"
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                container('ansible') {
                    sh '''
                        echo "Verifying ArgoCD deployment in namespace: ${params.ARGOCD_NAMESPACE}"
                        kubectl get pods -n ${params.ARGOCD_NAMESPACE}
                        kubectl get svc -n ${params.ARGOCD_NAMESPACE}
                    '''
                }
            }
        }

        stage('Ephemeral Cleanup') {
            when {
                equals expected: true, actual: params.EPHEMERAL_CLEANUP
            }
            steps {
                container('ansible') {
                    input message: "Clean up ephemeral ArgoCD in namespace '${params.ARGOCD_NAMESPACE}'?", ok: 'Clean Up'
                    sh '''
                        echo "Cleaning up ArgoCD from namespace: ${params.ARGOCD_NAMESPACE}"
                        kubectl delete namespace ${params.ARGOCD_NAMESPACE}
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*.log', allowEmptyArchive: true
            cleanWs()
        }
        success {
            echo 'ArgoCD deployment pipeline succeeded!'
        }
        failure {
            echo 'ArgoCD deployment pipeline failed!'
        }
    }
}

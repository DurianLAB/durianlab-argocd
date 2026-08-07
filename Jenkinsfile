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
      image: docker.io/ansible/ansible-core:2.17@sha256:bc5c8e9c2e8b0f40e2a5c4c2c0f0e3c4f6e8d0c0b0a0c0c0c0c0c0c0c0c0c0c
      imagePullPolicy: Always
      command:
        - cat
      tty: true
      env:
        - name: HOME
          value: /home/jenkins
      volumeMounts:
        - name: ansible-config
          mountPath: /etc/ansible
          readOnly: true
        - name: kubeconfig
          mountPath: /home/jenkins/.kube
          readOnly: true
        - name: workspace
          mountPath: /home/jenkins/agent
  volumes:
    - name: ansible-config
      configMap:
        name: ansible-config
    - name: kubeconfig
      secret:
        secretName: jenkins-kubeconfig
    - name: workspace
      emptyDir: {}
'''
        }
    }

    parameters {
        string(
            name: 'ARGOCD_NAMESPACE',
            defaultValue: 'argocd',
            description: 'Namespace to deploy ArgoCD (use unique name for ephemeral environments, e.g., argocd-test-<BUILD_NUMBER>)'
        )
        string(
            name: 'ARGOCD_RELEASE_NAME',
            defaultValue: 'argocd',
            description: 'Helm release name for ArgoCD'
        )
        string(
            name: 'ARGOCD_CHART_VERSION',
            defaultValue: '7.7.7',
            description: 'ArgoCD Helm chart version'
        )
        booleanParam(
            name: 'ENABLE_ANONYMOUS_ACCESS',
            defaultValue: true,
            description: 'Enable anonymous (view-only) access to ArgoCD'
        )
        booleanParam(
            name: 'EPHEMERAL_CLEANUP',
            defaultValue: false,
            description: 'Clean up the namespace after deployment (for ephemeral testing)'
        )
        choice(
            name: 'KUBECONFIG_SOURCE',
            choices: ['diskless-k8s', 'k3s-dev'],
            description: 'Which kubeconfig context to use'
        )
    }

    environment {
        ANSIBLE_DIR = "${WORKSPACE}"
        ANSIBLE_CONFIG = "${ANSIBLE_DIR}/ansible.cfg"
        KUBECONFIG_FILE = "/home/jenkins/.kube/config"
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

        stage('Setup Kubeconfig') {
            steps {
                container('ansible') {
                    script {
                        switch (params.KUBECONFIG_SOURCE) {
                            case 'diskless-k8s':
                                sh '''
                                    incus exec diskless:k8s-cp -- cat /etc/kubernetes/admin.conf > ${KUBECONFIG_FILE}
                                '''
                                break
                            case 'k3s-dev':
                                sh '''
                                    kubectl config view --minify > ${KUBECONFIG_FILE}
                                '''
                                break
                            default:
                                error("Unknown kubeconfig source: ${params.KUBECONFIG_SOURCE}")
                        }
                        sh 'chmod 600 ${KUBECONFIG_FILE}'
                    }
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
                            -e "argocd_chart_version=${params.ARGOCD_CHART_VERSION}" \
                            -e "kubeconfig_file=${KUBECONFIG_FILE}"
                    '''
                }
            }
        }

        stage('Deploy ArgoCD') {
            steps {
                container('ansible') {
                    script {
                        if (params.EPHEMERAL_CLEANUP) {
                            input message: "Deploy ArgoCD to namespace '${params.ARGOCD_NAMESPACE}'? This will be cleaned up after testing.", ok: 'Deploy'
                        } else {
                            input message: "Deploy ArgoCD to namespace '${params.ARGOCD_NAMESPACE}'?", ok: 'Deploy'
                        }
                    }
                    sh '''
                        ansible-playbook playbooks/playbook-argocd.yml \
                            -i inventories/dev/ \
                            -e "argocd_namespace=${params.ARGOCD_NAMESPACE}" \
                            -e "argocd_release_name=${params.ARGOCD_RELEASE_NAME}" \
                            -e "argocd_chart_version=${params.ARGOCD_CHART_VERSION}" \
                            -e "kubeconfig_file=${KUBECONFIG_FILE}"
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                container('ansible') {
                    sh '''
                        echo "Verifying ArgoCD deployment in namespace: ${params.ARGOCD_NAMESPACE}"
                        kubectl --kubeconfig=${KUBECONFIG_FILE} get pods -n ${params.ARGOCD_NAMESPACE}
                        kubectl --kubeconfig=${KUBECONFIG_FILE} get svc -n ${params.ARGOCD_NAMESPACE}
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
                        kubectl --kubeconfig=${KUBECONFIG_FILE} delete namespace ${params.ARGOCD_NAMESPACE}
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

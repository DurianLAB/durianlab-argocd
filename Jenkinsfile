pipeline {
    agent any

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
        KUBECONFIG_FILE = "${WORKSPACE}/kubeconfig"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Setup Kubeconfig') {
            steps {
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

        stage('Install Dependencies') {
            steps {
                sh '''
                    pip install --upgrade pip
                    pip install ansible ansible-lint
                    ansible-galaxy collection install -r requirements.yml || true
                '''
            }
        }

        stage('Lint Playbooks') {
            steps {
                sh 'ansible-lint playbooks/*.yml'
            }
        }

        stage('Syntax Check') {
            steps {
                sh '''
                    ansible-playbook --syntax-check playbooks/playbook-argocd.yml -i inventories/dev/
                '''
            }
        }

        stage('Dry Run') {
            steps {
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

        stage('Deploy ArgoCD') {
            steps {
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

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Verifying ArgoCD deployment in namespace: ${params.ARGOCD_NAMESPACE}"
                    kubectl --kubeconfig=${KUBECONFIG_FILE} get pods -n ${params.ARGOCD_NAMESPACE}
                    kubectl --kubeconfig=${KUBECONFIG_FILE} get svc -n ${params.ARGOCD_NAMESPACE}
                '''
            }
        }

        stage('Ephemeral Cleanup') {
            when {
                equals expected: true, actual: params.EPHEMERAL_CLEANUP
            }
            steps {
                input message: "Clean up ephemeral ArgoCD in namespace '${params.ARGOCD_NAMESPACE}'?", ok: 'Clean Up'
                sh '''
                    echo "Cleaning up ArgoCD from namespace: ${params.ARGOCD_NAMESPACE}"
                    kubectl --kubeconfig=${KUBECONFIG_FILE} delete namespace ${params.ARGOCD_NAMESPACE}
                '''
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

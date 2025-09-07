pipeline {
  agent any

  options {
    ansiColor('xterm')
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'v1', description: 'Docker image tag for this build')
    string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS region for EKS')
    string(name: 'CLUSTER_NAME', defaultValue: 'micro-eks', description: 'EKS cluster name')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        sh 'git submodule update --init --recursive || true'
      }
    }

    stage('Tools & Versions') {
      steps {
        sh '''
          echo "Docker:   $(docker version --format '{{.Server.Version}}' || echo not-installed)"
          echo "Terraform:$(terraform version | head -1 || echo not-installed)"
          echo "Ansible:  $(ansible --version | head -1 || echo not-installed)"
          echo "Kubectl:  $(kubectl version --client --short || echo not-installed)"
          echo "AWS CLI:  $(aws --version 2>&1 | head -1 || echo not-installed)"
        '''
      }
    }

    stage('CI - Build images') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            export DOCKERHUB_USER="$DH_USER"
            export IMAGE_TAG="${IMAGE_TAG}"
            docker compose -f docker-compose.ci.yml build --no-cache
          '''
        }
      }
    }

    stage('CI - Push images') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            for img in api web worker; do
              docker push "$DH_USER/$img:${IMAGE_TAG}"
            done
          '''
        }
      }
    }

    stage('Infra - Terraform apply (EKS)') {
      steps {
        withAWS(credentials: 'aws-creds', region: "${params.AWS_REGION}") {
          dir('infra/terraform') {
            sh '''
              terraform init -input=false
              terraform apply -auto-approve -input=false                 -var="region=${AWS_REGION}"                 -var="cluster_name=${CLUSTER_NAME}"
            '''
          }
        }
      }
    }

    stage('Kubeconfig for EKS') {
      steps {
        withAWS(credentials: 'aws-creds', region: "${params.AWS_REGION}") {
          sh '''
            mkdir -p $WORKSPACE/.kube
            aws eks update-kubeconfig --name "${CLUSTER_NAME}" --region "${AWS_REGION}" --kubeconfig "$WORKSPACE/.kube/config"
            export KUBECONFIG="$WORKSPACE/.kube/config"
            kubectl get nodes
          '''
        }
      }
    }

    stage('CD - Ansible deploy to K8s') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          withAWS(credentials: 'aws-creds', region: "${params.AWS_REGION}") {
            sh '''
              set -e
              export KUBECONFIG="$WORKSPACE/.kube/config"
              export DOCKERHUB_USER="$DH_USER"
              export IMAGE_TAG="${IMAGE_TAG}"

              # Create an isolated Python env for Ansible's k8s module
              python3 -m venv "$WORKSPACE/.ansible-venv"
              . "$WORKSPACE/.ansible-venv/bin/activate"
              pip install --upgrade pip
              # Kubernetes Python client + openshift helper (required by kubernetes.core)
              pip install "kubernetes" "openshift" "requests"

              # Ensure k8s Ansible collection is present
              ansible-galaxy collection install kubernetes.core --force

              # Sanity check (optional)
              python - <<'PY'
    import sys; import kubernetes; print("kubernetes OK", sys.version)
    PY

              # Run the playbook, telling Ansible which Python to use
              ansible-playbook -e "ansible_python_interpreter=$WORKSPACE/.ansible-venv/bin/python" \
                -i ansible/inventory.ini ansible/deploy.yaml

              kubectl -n micro get deploy,svc,pod
            '''
          }
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'infra/terraform/*.tfstate, **/*.log', allowEmptyArchive: true
    }
  }
}

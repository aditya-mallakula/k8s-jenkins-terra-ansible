pipeline {
  agent any

  options {
    ansiColor('xterm')
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  parameters {
    // Use "auto" for automatic immutable tags. You can still override manually if needed.
    string(name: 'IMAGE_TAG', defaultValue: 'auto', description: 'Docker image tag (use "auto" for v$BUILD_NUMBER-$GIT_SHA)')
    string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS region for EKS')
    string(name: 'CLUSTER_NAME', defaultValue: 'micro-eks', description: 'EKS cluster name')
  }

  environment {
    K8S_NAMESPACE = 'micro'
    WEB_SERVICE   = 'web'
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
          echo "Kubectl:  $(kubectl version --client || echo not-installed)"
          echo "AWS CLI:  $(aws --version 2>&1 | head -1 || echo not-installed)"
        '''
      }
    }

    // NEW: Make an immutable tag like v123-1a2b3c7
    stage('Prepare build metadata') {
      steps {
        sh '''
          set -e
          GIT_SHA=$(git rev-parse --short=7 HEAD || echo unknown)
          if [ -z "${IMAGE_TAG}" ] || [ "${IMAGE_TAG}" = "auto" ]; then
            IMAGE_TAG="v${BUILD_NUMBER}-${GIT_SHA}"
          fi
          echo "GIT_SHA=$GIT_SHA"   | tee image.env
          echo "IMAGE_TAG=$IMAGE_TAG" >> image.env
          echo "Resolved IMAGE_TAG=$IMAGE_TAG (GIT_SHA=$GIT_SHA)"
        '''
      }
    }

    stage('CI - Build images') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            set -e
            source image.env
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin

            export DOCKERHUB_USER="$DH_USER"
            export IMAGE_TAG="$IMAGE_TAG"

            # ensure amd64 images for your amd64 nodes
            export DOCKER_DEFAULT_PLATFORM=linux/amd64
            docker buildx create --use --name ci-builder || true

            docker compose -f docker-compose.ci.yml build --no-cache
          '''
        }
      }
    }

    stage('CI - Push images (immutable & latest)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            set -e
            source image.env
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin

            for img in api web worker; do
              # push immutable tag
              docker push "$DH_USER/$img:$IMAGE_TAG"
              # also tag & push :latest for convenience
              docker tag  "$DH_USER/$img:$IMAGE_TAG" "$DH_USER/$img:latest"
              docker push "$DH_USER/$img:latest"
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
              terraform apply -auto-approve -input=false \
                -var="region=${AWS_REGION}" \
                -var="cluster_name=${CLUSTER_NAME}"
            '''
          }
        }
      }
    }

    stage('Kubeconfig for EKS') {
      steps {
        withAWS(credentials: 'aws-creds', region: "${params.AWS_REGION}") {
          sh '''
            mkdir -p "$WORKSPACE/.kube"
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
              source image.env
              export KUBECONFIG="$WORKSPACE/.kube/config"
              export DOCKERHUB_USER="$DH_USER"
              export IMAGE_TAG="$IMAGE_TAG"

              # Clean Python env for Ansible k8s module
              python3 -m venv "$WORKSPACE/.ansible-venv"
              . "$WORKSPACE/.ansible-venv/bin/activate"
              pip install --upgrade pip
              pip install kubernetes openshift requests
              ansible-galaxy collection install kubernetes.core --force
              python -c "import kubernetes, sys; print('kubernetes OK', sys.version)"
              export ANSIBLE_PYTHON_INTERPRETER="$WORKSPACE/.ansible-venv/bin/python"

              cd ansible
              ansible-playbook -i inventory.ini deploy.yaml \
                -e dockerhub_user="$DOCKERHUB_USER" -e image_tag="$IMAGE_TAG"
              cd -
            '''
          }
        }
      }
    }

    stage('Smoke test & publish link') {
      steps {
        withAWS(credentials: 'aws-creds', region: "${params.AWS_REGION}") {
          sh '''
            set -euo pipefail
            export KUBECONFIG="$WORKSPACE/.kube/config"

            for d in web api worker; do
              echo "Waiting for $d rollout..."
              kubectl -n "$K8S_NAMESPACE" rollout status deploy/$d --timeout=180s
            done

            ELB=$(kubectl -n "$K8S_NAMESPACE" get svc "$WEB_SERVICE" -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
            echo "ELB=$ELB" | tee elb.env

            echo "Probing http://$ELB ..."
            for i in {1..20}; do
              code=$(curl -s -o /dev/null -w '%{http_code}' "http://$ELB" || true)
              if [ "$code" = "200" ]; then echo "OK (HTTP 200)"; break; fi
              sleep 5
            done

            curl -s "http://$ELB" | head -n 40 | tee web-response.txt
            kubectl -n "$K8S_NAMESPACE" get deploy,svc,pod -o wide | tee k8s-summary.txt
          '''
        }
      }
      post {
        always {
          archiveArtifacts artifacts: 'web-response.txt, elb.env, k8s-summary.txt, image.env', allowEmptyArchive: true
        }
        success {
          script {
            def elb = sh(script: "source elb.env && echo \$ELB", returnStdout: true).trim()
            def tag = sh(script: "source image.env && echo \$IMAGE_TAG", returnStdout: true).trim()
            currentBuild.displayName = "#${env.BUILD_NUMBER} ${tag}"
            currentBuild.description = "Web: http://${elb}"
            echo "Open your app: http://${elb}"
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

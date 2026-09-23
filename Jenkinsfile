pipeline {
    agent {
        kubernetes {
            label 'ci-buildkit-helm'
            idleMinutes 20
            yaml """
            apiVersion: v1
            kind: Pod
            metadata:
              annotations:
                container.apparmor.security.beta.kubernetes.io/buildkit: unconfined
            spec:
              serviceAccountName: jenkins-deployer
              securityContext:
                fsGroup: 1000
              containers:
              - name: buildkit
                image: moby/buildkit:v0.27.0-rootless
                imagePullPolicy: IfNotPresent
                command: ['sleep', 'infinity']
                env:
                - name: BUILDKITD_FLAGS
                  value: --oci-worker-no-process-sandbox --oci-worker-snapshotter=native
                - name: DOCKER_CONFIG
                  value: /home/user/.docker
                securityContext:
                  runAsUser: 1000
                  runAsGroup: 1000
                  seccompProfile:
                    type: Unconfined
                volumeMounts:
                - name: docker-config
                  mountPath: /home/user/.docker
                - name: buildkit-state
                  mountPath: /home/user/.local/share/buildkit
              - name: helm
                image: mirror.gcr.io/alpine/k8s:1.32.3
                imagePullPolicy: IfNotPresent
                command: ['sleep', 'infinity']
              - name: git
                image: mirror.gcr.io/alpine/git
                imagePullPolicy: IfNotPresent
                command: ['sleep', 'infinity']
              volumes:
              - name: docker-config
                secret:
                  secretName: regcred
                  items: [{key: .dockerconfigjson, path: config.json}]
              - name: buildkit-state
                hostPath:
                  path: /home/juanfran/jenkins-cache/buildkit
                  type: DirectoryOrCreate
            """
        }
    }

    environment {
        GITHUB_USER      = 'juanfranvelilla'
        GIT_CREDENTIAL   = 'github-personal-token'
        REGISTRY         = "ghcr.io/${GITHUB_USER}"
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
        HELM_CHARTS_REPO = 'https://github.com/juanFRANvelilla/helm-charts.git'
        HELM_CHARTS_BRANCH = 'develop'
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    if (!params.GIT_URL?.trim() || !params.APP_NAME?.trim()) {
                        error 'Faltan parámetros del job: GIT_URL y APP_NAME son obligatorios.'
                    }

                    def buildRoot = params.BUILD_ROOT?.toString() == 'true'
                    def repoName  = params.GIT_URL.tokenize('/')[-1].replaceAll(/\.git$/, '').toLowerCase()

                    env.BRANCH    = params.BRANCH?.trim() ?: 'main'
                    env.APP_DIR   = buildRoot ? '.' : params.APP_NAME
                    env.CHART_DIR = "${env.APP_DIR}/k8s"
                    env.IMAGE     = buildRoot ? "${REGISTRY}/${repoName}" : "${REGISTRY}/${repoName}-${params.APP_NAME}"
                    env.CACHE_IMAGE = "${env.IMAGE}/cache"
                    env.RELEASE   = buildRoot ? repoName : "${repoName}-${params.APP_NAME}"
                    env.CHART_PUBLISH_DIR = "charts/app/${repoName}/${params.APP_NAME}"
                    env.REPO_NAME = repoName

                    currentBuild.displayName = "#${env.BUILD_NUMBER} ${env.BRANCH}"

                    echo """
                    Repo:    ${params.GIT_URL}
                    Rama:    ${env.BRANCH}
                    Carpeta: ${env.APP_DIR}
                    Imagen:  ${env.IMAGE}:${IMAGE_TAG}
                    Caché:   ${env.CACHE_IMAGE}
                    Release: ${env.RELEASE}
                    Charts:  ${env.CHART_PUBLISH_DIR}
                    """.stripIndent()
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout([$class: 'GitSCM',
                          branches: [[name: "*/${env.BRANCH}"]],
                          userRemoteConfigs: [[url: params.GIT_URL, credentialsId: env.GIT_CREDENTIAL]]
                ])
            }
        }

        stage('Build and Push') {
            steps {
                container('buildkit') {
                    script {
                        def contextDir = "${env.WORKSPACE}/${env.APP_DIR}"
                        sh """
                        set -eu
                        export DOCKER_CONFIG=/home/user/.docker
                        buildctl-daemonless.sh build \\
                          --frontend dockerfile.v0 \\
                          --local context=${contextDir} \\
                          --local dockerfile=${contextDir} \\
                          --opt filename=Dockerfile \\
                          --import-cache type=registry,ref=${CACHE_IMAGE} \\
                          --export-cache type=registry,ref=${CACHE_IMAGE},mode=max \\
                          --output type=image,name=${IMAGE}:${IMAGE_TAG},push=true
                        """
                        echo "Imagen subida: ${env.IMAGE}:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Publish Helm Chart') {
            steps {
                lock(resource: 'helm-charts') {
                    container('git') {
                        withCredentials([usernamePassword(
                            credentialsId: env.GIT_CREDENTIAL,
                            usernameVariable: 'GIT_USER',
                            passwordVariable: 'GIT_TOKEN'
                        )]) {
                            sh """
                            set -eu
                            git config --global user.email "jenkins@ci.local"
                            git config --global user.name "Jenkins"
                            rm -rf helm-charts-repo
                            git clone --depth 1 --branch ${HELM_CHARTS_BRANCH} --single-branch \
                              https://\${GIT_USER}:\${GIT_TOKEN}@github.com/juanFRANvelilla/helm-charts.git \
                              helm-charts-repo

                            DEST=helm-charts-repo/${CHART_PUBLISH_DIR}
                            mkdir -p "\$DEST"
                            rm -rf "\$DEST"/*
                            cp -a ${CHART_DIR}/. "\$DEST"/

                            if [ -f "\$DEST/Chart.yaml" ]; then
                              sed -i "s/^appVersion:.*/appVersion: \\"${IMAGE_TAG}\\"/" "\$DEST/Chart.yaml"
                            fi
                            for values in "\$DEST"/values-pre.yaml "\$DEST"/values-pro.yaml; do
                              if [ -f "\$values" ]; then
                                sed -i "s/^  tag:.*/  tag: \\"${IMAGE_TAG}\\"/" "\$values"
                              fi
                            done

                            cd helm-charts-repo
                            git add "${CHART_PUBLISH_DIR}"
                            if git diff --cached --quiet; then
                              echo "Helm chart ${CHART_PUBLISH_DIR} already at ${IMAGE_TAG}"
                            else
                              git commit -m "Add Helm chart for ${REPO_NAME}-${params.APP_NAME} version ${IMAGE_TAG}"
                              git push origin ${HELM_CHARTS_BRANCH}
                            fi
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy PRE') {
            when {
                expression { params.DEPLOY_PRE?.toString() != 'false' }
            }
            steps {
                container('helm') {
                    script {
                        env.NAMESPACE = sh(
                            script: "yq '.namespace' ${CHART_DIR}/values-pre.yaml",
                            returnStdout: true
                        ).trim()

                        if (!env.NAMESPACE || env.NAMESPACE == 'null') {
                            error "El chart ${CHART_DIR} no define 'namespace' en values-pre.yaml"
                        }

                        sh """
                        helm upgrade --install ${RELEASE} ${CHART_DIR} \
                            --namespace ${NAMESPACE} \
                            -f ${CHART_DIR}/values-pre.yaml \
                            --set image.tag=${IMAGE_TAG} \
                            --atomic \
                            --wait \
                            --timeout 5m
                        """

                        sh "kubectl get pods -n ${NAMESPACE} -l app.kubernetes.io/instance=${RELEASE} -o wide"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "OK: ${env.IMAGE}:${IMAGE_TAG}" + (params.DEPLOY_PRE?.toString() != 'false' ? " desplegada en ${env.NAMESPACE}" : '')
        }
        failure {
            echo "Fallo en ${env.RELEASE ?: params.APP_NAME} (rama ${env.BRANCH ?: params.BRANCH})."
        }
    }
}

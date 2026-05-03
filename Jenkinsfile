pipeline {
    agent any

    environment {
        // ── Repo & server config ───────────────────────────────────────────
        GIT_REPO_URL  = 'https://github.com/HusnainNadeem/fileforge-docker-aws.git'
        GIT_BRANCH    = 'main'
        SERVER_IP     = '16.176.232.43'
        COMPOSE_FILE  = 'docker-compose.yml'
        HEALTH_URL    = 'http://16.176.232.43'
    }

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {

        // ══════════════════════════════════════════════════════════════════
        // STEP 1 — STOP RUNNING CONTAINERS
        // Gracefully stops and removes all containers defined in compose.
        // MongoDB named volume (mongo_data) is preserved — data is safe.
        // ══════════════════════════════════════════════════════════════════
        stage('Stop Running Containers') {
            steps {
                echo '🛑 Stopping all running containers...'
                sh '''
                    # Stop and remove containers (keeps volumes & networks)
                    docker compose -f ${COMPOSE_FILE} down --remove-orphans || true

                    # Safety: force-stop anything still running with our names
                    for NAME in nginx frontend backend mongo; do
                        if [ "$(docker ps -q -f name=^${NAME}$)" ]; then
                            echo "  Force-stopping container: ${NAME}"
                            docker stop ${NAME} && docker rm ${NAME} || true
                        fi
                    done

                    echo "✅ All containers stopped."
                    docker ps
                '''
            }
        }

        // ══════════════════════════════════════════════════════════════════
        // STEP 2 — PULL CODE FROM GITHUB
        // Clones / updates from your GitHub repo on the specified branch.
        // ══════════════════════════════════════════════════════════════════
        stage('Pull Code from GitHub') {
            steps {
                echo "📥 Pulling code from: ${GIT_REPO_URL} (branch: ${GIT_BRANCH})"
                git(
                    url: "${GIT_REPO_URL}",
                    branch: "${GIT_BRANCH}",
                    changelog: true,
                    poll: true
                )
                echo '✅ Code pulled successfully.'
                sh '''
                    echo "--- Repo structure ---"
                    ls -la
                '''
            }
        }

        // ══════════════════════════════════════════════════════════════════
        // STEP 3 — BUILD DOCKER IMAGES
        // Builds frontend (multi-stage: npm build → serve) and
        // backend (node:22-alpine → node server.js) images fresh.
        // ══════════════════════════════════════════════════════════════════
        stage('Build Docker Images') {
            steps {
                echo '🔨 Building Docker images...'
                sh '''
                    # ── Frontend: multi-stage (builder + runner) ────────────
                    echo "[1/2] Building frontend image..."
                    docker build \
                        --no-cache \
                        --tag frontend:latest \
                        --tag frontend:${BUILD_NUMBER} \
                        --file frontend/Dockerfile \
                        ./frontend

                    # ── Backend: node:22-alpine ─────────────────────────────
                    echo "[2/2] Building backend image..."
                    docker build \
                        --no-cache \
                        --tag backend:latest \
                        --tag backend:${BUILD_NUMBER} \
                        --file backend/Dockerfile \
                        ./backend

                    echo "✅ Both images built successfully."
                    docker images | grep -E "frontend|backend"
                '''
            }
        }

        // ══════════════════════════════════════════════════════════════════
      
        // ══════════════════════════════════════════════════════════════════
        // STEP 5 — DEPLOY TO SERVER (16.176.232.43)
        // Brings the full production stack up with the newly built images.
        // MongoDB container is NOT recreated → zero data loss.
        // ══════════════════════════════════════════════════════════════════
        stage('Deploy') {
            steps {
                echo "🚀 Deploying FileForge PRO to ${SERVER_IP}..."
                sh '''
                    # Start MongoDB first (if not already running)
                    docker compose -f ${COMPOSE_FILE} up -d mongo
                    sleep 5

                    # Recreate app containers with new images
                    docker compose -f ${COMPOSE_FILE} up -d \
                        --force-recreate \
                        --remove-orphans \
                        backend frontend nginx

                    echo "--- Running containers after deploy ---"
                    docker compose -f ${COMPOSE_FILE} ps
                '''
            }
        }

        // ══════════════════════════════════════════════════════════════════
        // STEP 6 — HEALTH CHECK ON PRODUCTION SERVER
        // Retries 6 × 10s (60s window) to confirm nginx is live on
        // the production IP 16.176.232.43
        // ══════════════════════════════════════════════════════════════════
        stage('Health Check') {
            steps {
                echo "❤️  Verifying production server at ${HEALTH_URL} ..."
                retry(6) {
                    sleep(time: 10, unit: 'SECONDS')
                    sh "curl --fail --silent --max-time 10 ${HEALTH_URL}"
                }
                echo "✅ FileForge PRO is LIVE at http://${SERVER_IP}"
            }
        }

        // ══════════════════════════════════════════════════════════════════
        // CLEANUP — Remove dangling images, keep last 3 builds per image
        // ══════════════════════════════════════════════════════════════════
        stage('Cleanup Old Images') {
            steps {
                echo '🧹 Removing dangling/old Docker images...'
                sh '''
                    docker image prune -f

                    # Keep only 3 most recent numbered tags per image
                    docker images frontend --format "{{.Tag}}" | \
                        grep -E "^[0-9]+$" | sort -rn | tail -n +4 | \
                        xargs -r -I{} docker rmi frontend:{} || true

                    docker images backend --format "{{.Tag}}" | \
                        grep -E "^[0-9]+$" | sort -rn | tail -n +4 | \
                        xargs -r -I{} docker rmi backend:{} || true

                    echo "✅ Cleanup done."
                    docker images
                '''
            }
        }
    }

    // ══════════════════════════════════════════════════════════════════════
    // POST PIPELINE — Success banner / failure rollback
    // ══════════════════════════════════════════════════════════════════════
    post {
        success {
            echo """
╔═══════════════════════════════════════════════════════╗
║   ✅  FileForge PRO — Build #${BUILD_NUMBER} DEPLOYED OK     ║
║   🌐  Live at → http://16.176.232.43                  ║
╚═══════════════════════════════════════════════════════╝
            """
        }

        failure {
            echo '❌ Pipeline FAILED — rolling back to previous build...'
            sh '''
                PREV=$((BUILD_NUMBER - 1))
                FRONTEND_EXISTS=$(docker image inspect frontend:${PREV} > /dev/null 2>&1 && echo yes || echo no)
                BACKEND_EXISTS=$(docker image inspect backend:${PREV}  > /dev/null 2>&1 && echo yes || echo no)

                if [ "$FRONTEND_EXISTS" = "yes" ] && [ "$BACKEND_EXISTS" = "yes" ]; then
                    echo "Found build #${PREV} images. Rolling back..."
                    docker tag frontend:${PREV} frontend:latest
                    docker tag backend:${PREV}  backend:latest
                    docker compose -f ${COMPOSE_FILE} up -d \
                        --force-recreate frontend backend nginx
                    echo "✅ Rolled back to build #${PREV}."
                else
                    echo "⚠️  No rollback images found. Stack may be down."
                    echo "    Run manually: docker compose up -d"
                fi
            '''
        }

        always {
            echo "Build #${BUILD_NUMBER} finished → ${currentBuild.currentResult}"
        }
    }
}

pipeline {
  agent any

  options {
    timeout(time: 20, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
  }

  environment {
    IMAGE = 'anhvty/mern-todo-be'
  }

  stages {
    stage('Prepare') {
      steps {
        script {
          env.TAG   = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
          env.CI_ID = 'todo-ci-' + sh(returnStdout: true, script: 'openssl rand -hex 4').trim()
        }
      }
    }

    stage('Build backend image') {
      steps {
        sh 'docker build --tag "$IMAGE:$TAG" .'
      }
    }

    stage('Test register & login with MongoDB') {
      steps {
        sh '''#!/usr/bin/env bash
set -euo pipefail

NET="$CI_ID-net"
MONGO="$CI_ID-mongo"
BE="$CI_ID-be"

docker network create "$NET"
docker run -d --name "$MONGO" --network "$NET" \
  mongodb/mongodb-community-server:8.0-ubi8

timeout 90s bash -c "until docker exec $MONGO mongosh --quiet \
  --eval 'db.adminCommand({ ping: 1 })' >/dev/null 2>&1; do sleep 2; done"

docker run -d --name "$BE" --network "$NET" \
  -p 127.0.0.1::8000 \
  -e PORT=8000 \
  -e MONGO_URI="mongodb://$MONGO:27017/mern_todo_ci" \
  -e JWT_SECRET="$(openssl rand -hex 32)" \
  "$IMAGE:$TAG"

# Lấy port ngẫu nhiên Docker gán, tránh đụng port trên máy
HOST_PORT=$(docker port "$BE" 8000/tcp | head -1 | awk -F: '{print $NF}')
BASE="http://127.0.0.1:$HOST_PORT"

curl --silent --show-error --max-time 5 --retry-max-time 60 \
  --retry 20 --retry-delay 2 --retry-all-errors \
  "$BASE/api/user/getuser" -o /dev/null

curl --fail --silent --show-error --max-time 45 \
  -H 'Content-Type: application/json' \
  --data '{"name":"CI Demo","email":"ci@example.com","password":"CiTodo@2026"}' \
  "$BASE/api/user/register" \
  | jq -e '.token | strings | length > 0' > /dev/null

curl --fail --silent --show-error --max-time 10 \
  -H 'Content-Type: application/json' \
  --data '{"email":"ci@example.com","password":"CiTodo@2026"}' \
  "$BASE/api/user/login" \
  | jq -e '.token | strings | length > 0' > /dev/null

echo "Register + login OK"
'''
      }
      post {
        failure {
          sh '''#!/usr/bin/env bash
docker logs "$CI_ID-mongo" || true
docker logs "$CI_ID-be" || true
'''
        }
      }
    }

    stage('Push backend image') {
      when {
        allOf {
          branch 'main'
          not { changeRequest() }
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                          usernameVariable: 'DH_USER',
                                          passwordVariable: 'DH_PASS')]) {
          sh '''#!/usr/bin/env bash
set -euo pipefail
echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
docker push "$IMAGE:$TAG"
'''
        }
      }
    }
  }

  post {
    always {
      sh '''#!/usr/bin/env bash
docker rm -f "$CI_ID-be" "$CI_ID-mongo" 2>/dev/null || true
docker network rm "$CI_ID-net" 2>/dev/null || true
docker logout 2>/dev/null || true
'''
    }
  }
}
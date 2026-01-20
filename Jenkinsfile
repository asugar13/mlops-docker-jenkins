pipeline {
  agent any

  // Choose ONE trigger:
  // triggers { cron('H 2 * * *') }           // nightly at ~2am
  triggers { githubPush() }                  // build on GitHub webhook push

  environment {
    IMAGE     = "adult-model"
    CONT      = "model"
    THRESHOLD = "0.8"
    META_PATH = "/home/jovyan/results/train_metadata.json"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build image + start container') {
      steps {
        sh '''
          set -e
          docker rm -f "$CONT" 2>/dev/null || true
          docker build -t "$IMAGE" .
          docker run -d --name "$CONT" "$IMAGE" tail -f /dev/null
        '''
      }
    }

    stage('Preprocess') {
      steps {
        sh 'docker exec "$CONT" python3 preprocessing.py'
      }
    }

    stage('Train') {
      steps {
        sh 'docker exec "$CONT" python3 train.py'
      }
    }

    stage('Validate (and Test if OK)') {
      steps {
        sh '''
          set -e
          val_acc=$(docker exec "$CONT" jq -r .validation_acc "$META_PATH")
          echo "validation_acc=$val_acc threshold=$THRESHOLD"

          awk -v a="$val_acc" -v t="$THRESHOLD" 'BEGIN{exit !(a>=t)}'
          echo "OK: running test"
          docker exec "$CONT" python3 test.py

          echo "=== METADATA ==="
          docker exec "$CONT" cat "$META_PATH" || true
        '''
      }
    }
  }

  post {
    always {
      sh 'docker rm -f "$CONT" 2>/dev/null || true'
    }
  }
}

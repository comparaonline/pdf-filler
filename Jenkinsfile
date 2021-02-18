pipeline {
  agent { node { label 'jenkins-gcp' } }
  options {
    timeout(time: 15, unit: "MINUTES")
  }
  stages {
    stage("Initialize") {
      steps {
        initialize_function()
      }
    }
    stage("Test") {
      steps {
        test_function()
      }
    }
    stage("Build") {
      when {
        anyOf {
          branch "release"
          branch "master"
        }
      }
      steps {
        build_function()
      }
    }
    stage("Deploy") {
      when {
        anyOf {
          branch "release"
          branch "master"
        }
      }
      steps {
        deploy_function()
      }
    }
    stage("Accept Release") {
      when {
        anyOf {
          branch "release"
        }
      }
      steps {
        initialize_function()
        accept_release_function()
      }
    }
    stage("Merge to Destination") {
      when {
        environment name: "PROMOTE_ENVIRONMENT", value: "yes";
      }
      steps {
        merge_function()
      }
    }
  }
  post {
    always {
      cleanup_function()
    }
  }
}

def initialize_function() {
  // - PLEASE ONLY MODIFY VALUES IN THIS FUNCTION!!!
  if(env.BRANCH_NAME == "release") {
      env.STAGE = "staging"
      env.DESTINATION_BRANCH = "master"
      env.DESTINATION_ENVIRONMENT = "PRODUCTION"
  }
  else if(env.BRANCH_NAME == "master") {
    env.STAGE = "production"
  }

  env.APP_NAME = "pdf-filler"
  env.DOCKER_SERVICE_NAME = "app"
  env.DOCKER_COMPOSE_FILENAME = "docker-compose.yml"
  env.DOCKER_NETWORK_NAME = "confluent_kafka"
  env.TEST_COMMAND = "yarn test"
  env.VERSION_TAG = env.BRANCH_NAME + '_' + env.BUILD_NUMBER
  env.KUBECTL_PATH = "/snap/bin/kubectl"
  env.GCLOUD_PATH = "/snap/bin/gcloud"

  acceptance_message = "Accept changes and deploy: ${env.APP_NAME}:${env.BRANCH_NAME} ${env.BUILD_NUMBER} Go to: ${env.BUILD_URL}"
  input_message = "Accept changes and deploy to ${env.DESTINATION_ENVIRONMENT}?"
  choice_name = "Deploy to ${env.DESTINATION_ENVIRONMENT}"
  choice_description = 'Choose "yes" if you want to DEPLOY TO ' + env.DESTINATION_ENVIRONMENT
  choice_options = "no\nyes"
  slack_channel = "#release"
  slack_color = "warning"

  sh "docker --version"
  sh "docker-compose --version"
  sh "docker network inspect $DOCKER_NETWORK_NAME > /dev/null 2>&1 || docker network create $DOCKER_NETWORK_NAME"
  sh 'git config --global user.email "jenkins@comparaonline.com"'
  sh 'git config --global user.name "Jenkins"'
}

def test_function() {
  sh "docker-compose -f $DOCKER_COMPOSE_FILENAME run --rm $DOCKER_SERVICE_NAME $TEST_COMMAND"
}

def build_function() {
  sh "docker-compose -f $DOCKER_COMPOSE_FILENAME build"
}

def deploy_function() {
  sh "STAGE=$STAGE deploy -k $KUBECTL_PATH -g $GCLOUD_PATH -v $VERSION_TAG $APP_NAME"
}

def accept_release_function() {
  mattermostSend channel: slack_channel, color: slack_color, message: acceptance_message
  env.PROMOTE_ENVIRONMENT = input message: input_message,
  parameters: [choice(name: choice_name, choices: choice_options, description: choice_description)]
}

def merge_function() {
  sh "git stash"
  sh "git remote set-branches --add origin $DESTINATION_BRANCH"
  sh "git fetch"
  sh "git checkout $DESTINATION_BRANCH || git checkout -b $DESTINATION_BRANCH origin/$DESTINATION_BRANCH"
  sh "git pull"
  sh "git merge origin/$BRANCH_NAME"
  sh "git push origin $DESTINATION_BRANCH"
}

def cleanup_function() {
  sh "docker-compose -f $DOCKER_COMPOSE_FILENAME down --rmi all"
  sh "docker-compose -f $DOCKER_COMPOSE_FILENAME rm -sfv"
}

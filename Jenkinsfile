pipeline {

  parameters {
    string(defaultValue: null, description:'Please choose your team: ', name: Team)
    string(defaultValue: null, description:'Please choose your project: ', name: Project)
    string(defaultValue: null, description:'Please choose your environment: ', name: Environment)
    string(defaultValue: null, description:'Please choose your git branch/tag/commit: ', name: Git_Commit)
  }

  agent {
    node {
      label 'docker.ci.uktrade.io'

      stages {

        def options_json
        def envars
        def deployer = docker.image('ukti/deployer:latest')

        deployer.pull()
        deployer.inside {
          stage('checkout') {
            git 'https://github.com/uktrade/ci-pipeline.git'
            sh 'bundle install'
          }
          stage('init') {
            sh "${env.WORKSPACE}/bootstrap.rb"
            options_json = readJSON file: "${env.WORKSPACE}/option.json"
          }
        }

        stage('input') {
          script {
            options = load "${env.WORKSPACE}/input.groovy"
            if (env.Team == null) {
              team = input(
                id: 'team', message: 'Please choose your team: ', parameters: [
                [$class: 'ChoiceParameterDefinition', name: 'Team', description: 'Team', choices: options.get_team(options_json)]
              ])
              env.Team = team
            }
            if (env.Project == null) {
              project = input(
                id: 'project', message: 'Please choose your project: ', parameters: [
                [$class: 'ChoiceParameterDefinition', name: 'Project', description: 'Project', choices: options.get_project(options_json,team)]
              ])
              env.Project = project
            }
            if (env.Environment == null) {
              environment = input(
                id: 'environment', message: 'Please choose your environment: ', parameters: [
                [$class: 'ChoiceParameterDefinition', name: 'Environment', description: 'Environment', choices: options.get_env(options_json, team, project)]
              ])
              env.Environment = environment
            }
            if (env.Git_Commit == null) {
              git_commit = input(
                id: 'git_commit', message: 'Please enter your git branch/tag/commit: ', parameters: [
                [$class: 'StringParameterDefinition', name: 'Git Commit', description: 'GitCommit', defaultValue: 'master']
              ])
              env.Git_Commit = git_commit
            }
          }
        }

        deployer.inside {
          stage('setup') {
            git 'https://github.com/uktrade/ci-pipeline.git'
            sh 'bundle install'
            withCredentials([string(credentialsId: env.VAULT_TOKEN_ID, variable: 'TOKEN')]) {
              env.VAULT_TOKEN = TOKEN
              sh "${env.WORKSPACE}/bootstrap.rb ${env.Team} ${env.Project} ${env.Environment}"
            }
            envars = readProperties file: "${env.WORKSPACE}/env"
          }
        }

        stage('load') {
          script {
            envars.each {
              var = it.toString().split("=", 2)
              env."${var[0]}" = var[1]
            }
          }
        }

        deployer.inside {
          stage('deploy') {
            git url: env.SCM, branch: env.Git_Commit
            sh """
              env | sort
              bash -c "${env.PAAS_RUN}"
            """
          }
        }
      }
    }
  }
}

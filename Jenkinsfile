import hudson.model.*

def credentialsId = 'gitlab_sshkey'
//def credentialsId = 'gitlab-corp-jenkins-user'

node {

  stage ("Checkout") {

    checkout([$class: 'GitSCM', branches: [[name: ("*/${scm.branches[0].name}") ]],
    doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
    userRemoteConfigs: [[credentialsId: credentialsId,
    url: scm.getUserRemoteConfigs()[0].getUrl()]],
    workspaceUpdater: [$class: 'CheckoutUpdater']])
  }

  stage ("Build") {

    wrap([$class: 'BuildUser']) {
      echo "getting api key from current user: $BUILD_USER_ID"
      def user = User.get("$BUILD_USER_ID")

      def apiToken = user.getProperty(jenkins.security.ApiTokenProperty.class).getApiToken()

      user = null

      def projects = []
      projects.addAll(sh(script: 'ls defs/', returnStdout: true).split("\n"))

      projects.remove('sample')

      def userInput = [:]
      try {
          timeout(time: 2, unit: 'MINUTES') {
            userInput = input(message: 'Informe as opções',
              parameters: [
                string(name: 'API_TOKEN', description: """Enter your API Token.
                    Configure it <a href='$JENKINS_URL/me/configure#yui-gen2'>here</a>""",
                  defaultValue: "$apiToken"),
                choice(name: 'PROJECT',
                   choices: ['all'].plus(projects).join("\n"),
                   description: 'Escolha o projeto'),
               choice(name: 'TARGET',
                  choices: ['all', 'views_only', 'jobs_only'].join("\n"),
                  description: 'Tipo de objeto para criar'),
              choice(name: 'RUN_MODE',
                 choices: ['update', 'delete'].join("\n"),
                 description: 'Modo de execução')
             ])
           }
      } catch(err) { // timeout reached or input false
          throw err
      }

     // Build+User+Vars+Plugin

      echo """starting job-builder with user $BUILD_USER_ID
        using apiToken: ${params.API_TOKEN}"""

      sh "cp jenkins_jobs-sample.ini jenkins_jobs.ini"
      sh "sed -i 's~JENKINS_URL~${JENKINS_URL}~g' jenkins_jobs.ini"
      sh "sed -i 's~JENKINS_USER_ID~${BUILD_USER_ID}~g' jenkins_jobs.ini"
      sh "sed -i 's~JENKINS_USER_APIKEY~${userInput.API_TOKEN}~g' jenkins_jobs.ini"

      def mode = userInput.RUN_MODE ?: 'update'
      def target = userInput.TARGET ?: 'all'
      def project = userInput.PROJECT

      target = 'all'.equals(target) ? '' : "-- only-$target"
      project = 'all'.equals(project) ? 'defs' : "defs/$project"

      def cmd = "jenkins-jobs --conf jenkins_jobs.ini $mode"

      if ("update".equals(mode)) {
          cmd+= " --delete-old -x defs/sample -r $project:globals"
      }

      sh "$cmd $target"
    }
  }
}


//@Library('shared-jenkins-library') _
node('jenkins-agent-helm') {


        echo "Start of pipeline!!"

         def deploymentStrategy = "INSTALL"
         def AutomaticPROpening = true
         def AutomaticPRMerging = false
         def gitHubAccountOrganizationName = "zvigrinberg"
         def gitOpsRepoName = "gitops-cd-pipeline-jenkins
         def gitBranch = "${env.BRANCH_NAME}"
         def latestInstallReleaseVersion
         def releaseExists=false
         def nameOfChart
         def chartFileName
         def releaseName
         def repo
         def version
         def clusterAddress
         def releaseNamespace
         String inputStrategy
         String releaseRevision
         def final mainBranch = "main"
         echo "git branch: ${gitBranch}"
         echo "default deployment strategy =  ${deploymentStrategy}"
         withCredentials([string(credentialsId: 'jenkins-token-for-helm', variable: 'ocpToken'),
                     file(credentialsId : 'openshift-ca' , variable: 'caLocation'),
                     file(credentialsId : 'gpg-private-key-for-cd', variable: 'privateKeyLocation' )]) {
             stage('Init workspace')
                     {
                         //set the pipeline terminal with essential ENV Variable for working in helm with container registry
                         sh(script: "export HELM_EXPERIMENTAL_OCI=1", returnStdout: true).trim()
                         sh(script: "gpg --allow-secret-key-import --import ${privateKeyLocation}")

                     }

             stage('read parameters from GIT')
                     {
                         checkout scm
                         def yaml = readYaml file: 'Charts.yaml'
                         nameOfChart = yaml.charts[0].name
                         releaseName = yaml.charts[0].releaseName
                         repo = yaml.charts[0].repo
                         version = yaml.charts[0].version
                         clusterAddress = yaml.clusterAddress
                         releaseNamespace = yaml.charts[0].releaseNamespace
                         inputStrategy = yaml.charts[0].strategy


                         echo "name of chart to deploy: ${nameOfChart}"
                         echo "release name: ${releaseName}"
                         echo "chart repo: ${repo}"
                         echo "chart version: ${version}"
                         echo "cluster Address to deploy to: ${clusterAddress}"
                         if(inputStrategy == null)
                         {
                             inputStrategy = new String(" ")
                         }
                         if(inputStrategy.trim() != "")
                         {
                             deploymentStrategy = inputStrategy
                             echo "overriding default strategy with value=${inputStrategy}"

                         }


                     }


             //feature/topic branch, needs only to validate parameters, test chart(linting and helm template) , perform a dry-run install
             //and open a pull-request to main branch
             stage('get latest Release Revision') {
                 //compute latestInstallReleaseVersion
                 releaseRevision = sh(script: "helm ls -n ${releaseNamespace} | grep -i ${releaseName} | grep -v NAME | awk '{print \$3}'", returnStdout: true).trim()
                 if (!releaseRevision.trim().equals("")) {
                     releaseExists = true
                 }

             }
             if (gitBranch != mainBranch) {
                 echo "executing pipeline on ${gitBranch}"
                 stage('Validate CD parameters') {
                     def result
                     // Checks that cluster exists, and valid.
                     //need to try and catch all errors and in the catch clause ask for the ifs
                     try {
                         result = sh(script: "helm ls --kube-apiserver ${clusterAddress} ", returnStdout: true).trim()
                     }
                     catch (Exception e) {
                         echo "validated cluster existance : ${result}"
                         if (result.toString().endsWith("no such host")) {
                             error "Kubernetes Cluster ${clusterAddress} not reachable or malformed"
                         } else {
                             echo "Validated existence of kubernetes cluster ${clusterAddress}"
                         }
                     }
                     if (inputStrategy != "INSTALL" && inputStrategy != "UNINSTALL" && inputStrategy != "ROLLBACK"
                             && inputStrategy.trim() != "") {
                         error "Unsupported strategy: ${inputStrategy}, must be one of [ INSTALL , UNINSTALL , ROLLBACK ] or empty, exiting... "
                     }

                 }
                 // Checks that chart url exists and that chart can be downloaded.

                 stage('Check Validity Of Chart')
                         {

                             chartFileName = pullChart((String) repo, (String) version)
                             def lintResult = sh(script: "helm lint ${chartFileName} ", returnStdout: true).trim()
                             echo "result of linting the chart: \n ${lintResult}"
                             def templateResult = sh(script: "helm template ${releaseName} ${chartFileName} -f ${nameOfChart}/values.yaml ", returnStdout: true).trim()
                             echo "result of rendering the chart: \n ${templateResult}"
                         }
                 if (deploymentStrategy.equals("INSTALL")) {
                     loginToOCPCluster((String) clusterAddress, (String) ocpToken, (String) caLocation)
                     stage('Run Helm dry-run')
                             {
                                 def templateResult
                                 def installUpgrade
                                 GString upgradeEdition
                                 if (!releaseExists) {
                                     installUpgrade = "install"
                                     upgradeEdition = GString.EMPTY
                                 } else {
                                     installUpgrade = "upgrade"
                                     upgradeEdition = "-n ${releaseNamespace}"
                                 }

                                 echo "Running helm ${installUpgrade} --dry-run"
                                 templateResult = sh(script: "helm ${installUpgrade} ${releaseName} ${chartFileName} -f ${nameOfChart}/values.yaml ${upgradeEdition} --dry-run ", returnStdout: true).trim()

//                                 error "Failed to install helm-install dry-run, details: ${templateResult}"

                             }

                 }
                 withCredentials([string(credentialsId: 'gh-pat-for-pr', variable: 'GH_TOKEN')]) {
                     String prNumber
                     if (AutomaticPROpening) {
                         stage('Open PR to main branch')
                                 {
                                     prNumber = sh(script: "curl -X POST -H \"Accept: application/vnd.github+json\" -H \"Authorization: token ${GH_TOKEN}\" https://api.github.com/repos/${gitHubAccountOrganizationName}/${gitOpsRepoName}/pulls -d '{\"title\": \"CD Validations passed successfully for build number ${BUILD_NUMBER} - CD strategy = ${deploymentStrategy} \",\"body\": \"Before reviewing, please look on the validation job at: ${BUILD_URL}\",\"head\": \"${gitBranch}\",\"base\": \"${mainBranch}\"}' | jq .number ", returnStdout: true).trim()
                                     echo "Number of PR Created : "
                                     echo "${prNumber}"

                                 }
                     }
                     if (AutomaticPRMerging) {
                         stage('Merge PR to main branch')
                                 {
                                     def Result = sh(script: "curl -X PUT -H \"Accept: application/vnd.github+json\" -H \"Authorization: token ${GH_TOKEN}\" https://api.github.com/repos/${gitHubAccountOrganizationName}/${gitOpsRepoName}/pulls/${prNumber}/merge -d '{\"commit_title\":\"Automatic CD approval and merging of PR\",\"commit_message\":\"Jenkins build URL: ${BUILD_URL} \",\"head\":\"${gitBranch}\",\"base\":\"${mainBranch}\"}'", returnStdout: true).trim()
                                 }
                     }
                 }
             }
             //main branch, need to install/upgrade/rollback release.
             else {
                 echo "executing pipeline on ${mainBranch}"
                 chartFileName = pullChart((String) repo, (String) version)
                 loginToOCPCluster((String) clusterAddress,(String)ocpToken,(String)caLocation)
                 
                 if (deploymentStrategy == "INSTALL" || deploymentStrategy == "") {
                     //installs a chart                     
                   //decrypt values-secrets.yaml using private keys before installing/upgrading chart
                     sh(script: "sops -d --in-place ${nameOfChart}/values-secrets.yaml")                         
                     if (!releaseExists) {
                         stage('Deploy Chart on Cluster') {
                             echo "About to install release ${releaseName} on release namespace ${releaseNamespace}"
                             def installationOutput = sh(script: "helm install ${releaseName} ${chartFileName}  -n ${releaseNamespace} --create-namespace -f ${nameOfChart}/values.yaml -f ${nameOfChart}/values-secrets.yaml ", returnStdout: true).trim()
                             echo "helm intall output : \n ${installationOutput}"
                         }


                     }
                     //Upgrade a release
                     else {
                         stage('Upgrade a release on Cluster') {

                             String upgradeRevision = (releaseRevision.toInteger().intValue() + 1).toString()
                             echo "About to upgrade a release ${releaseName} with version ${releaseRevision} to version ${upgradeRevision}  on release namespace ${releaseNamespace}"
                             def upgradeOutput = sh(script: "helm upgrade ${releaseName} ${chartFileName}  -n ${releaseNamespace} -f ${nameOfChart}/values.yaml -f ${nameOfChart}/values-secrets.yaml", returnStdout: true).trim()
                             echo "helm upgrade output : \n ${upgradeOutput}"
                         }
                     }
                 } else if (deploymentStrategy.equals("UNINSTALL")) {
                     stage('Uninstall Release')
                             {
                                 echo "About to uninstall release ${releaseName} from  release namespace ${releaseNamespace}"
                                 def uninstallOutput = sh(script: "helm uninstall ${releaseName}  -n ${releaseNamespace}", returnStdout: true).trim()
                                 echo "helm uninstall output : \n ${uninstallOutput}"
                             }
                 } else if (deploymentStrategy.equals("ROLLBACK")) {
                     stage('Rollback to Previous version')
                             {
                                 String rollbackRevision = (releaseRevision.toInteger().intValue() - 1).toString()
                                 echo "About to rollback release ${releaseName} from revision ${releaseRevision} to former revision ${rollbackRevision} on release namespace ${releaseNamespace}"
                                 try{
                                 def rollbackOutput = sh(script: "helm rollback ${releaseName} ${rollbackRevision}  -n ${releaseNamespace}", returnStdout: true).trim()
                                 echo "helm rollback output : \n ${rollbackOutput}"
                                 }
                                 catch (Exception e)
                                 {
                                     echo "failed to rollback, try one last time"
                                     def rollbackOutput = sh(script: "helm rollback ${releaseName} ${rollbackRevision}  -n ${releaseNamespace} --cleanup-on-fail" , returnStdout: true).trim()
                                     echo "helm rollback output : \n ${rollbackOutput}"
                                 }
                             }
                 } else {
                     error "pipeline failed, unsupported deployment Strategy, exiting..."
                 }

             }
         }
        stage('Clean Workspace')
          {
              cleanWs()
          }
}

private void loginToOCPCluster(String clusterAddress,String ocpToken,String caLocation) {
    stage('Authenticate to Openshift Cluster') {
            def result = sh(script: "oc login --token ${ocpToken} --server=${clusterAddress} --certificate-authority=${caLocation}", returnStdout: true).trim()
            echo "Response from connecting to openshift cluster : \n ${result}"
        }
    }

private String pullChart(String repo,String version ) {
    def chartName= " "
    stage('Pulling Chart from Registry') {

         int firstSlash = repo.indexOf("/")
         if(firstSlash != -1)
         {
             String registryServer = repo.substring(0,firstSlash)
             withCredentials([usernamePassword(credentialsId: 'quay-io-registry-credentials', usernameVariable: 'USER', passwordVariable: 'PASSWORD')]) {
                 loginToRegistry(registryServer,(String)USER,(String)PASSWORD)
                 def result = sh(script: "helm pull oci://${repo} --version ${version} ", returnStdout: true).trim()
                 echo "Response from pulling chart : \n ${result}"
                 chartName = sh(script: "ls *${version}.tgz ", returnStdout: true).trim()
             }

         }
         else
         {
             error "repo's server is not valid"
         }


    }
    return chartName
}

private void loginToRegistry(String server, String user, String password) {
    def result = sh(script: "helm registry login ${server} -u ${user} -p ${password} ", returnStdout: true).trim()
    echo "Response from connecting to helm OCI registry : \n ${result}"

}

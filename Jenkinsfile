pipeline {
	agent any

	//Configure the following environment variables before executing the Jenkins Job	
	environment {
		IntegrationFlowID = "${env.IntegrationFlowID}"
		CPIHost = "${env.CPI_HOST}"
		CPIOAuthHost = "${env.CPI_OAUTH_HOST}"
		CPIOAuthCredentials = "${env.CPI_OAUTH_CRED}"	
		GITRepositoryURL  =  "${env.GITRepositoryURL}"
		GITCredentials = "${env.GITCredentials}"
		GITToken = "${env.GITToken}"
		GITBranch = "${env.GITBranch}"
        GITFolder = "${env.GITFolder}"
        GITComment = "${env.GITComment}"
   	}
	
	stages {
		stage('download integration artefact and store it in Git') {
			steps {
			 	deleteDir()
				script {
					//clone repo 
					checkout([
						$class: 'GitSCM',
						branches: [[name: env.GITBranch]],
						doGenerateSubmoduleConfigurations: false,
						extensions: [
							[$class: 'RelativeTargetDirectory',relativeTargetDir: "."],
							[$class: 'SparseCheckoutPaths',  sparseCheckoutPaths:[[$class:'SparseCheckoutPath', path: env.GITFolder]]]
						],
						submoduleCfg: [],
						userRemoteConfigs: [[
							credentialsId: env.GITCredentials,
							url: 'https://' + env.GITRepositoryURL
						]]
					])
					
					//get token
					println("Request token");
					def token;
					try{
					def getTokenResp = httpRequest acceptType: 'APPLICATION_JSON', 
						authentication: env.CPIOAuthCredentials, 
						contentType: 'APPLICATION_JSON', 
						httpMode: 'POST', 
						responseHandle: 'LEAVE_OPEN', 
						timeout: 30, 
						url: 'https://' + env.CPIOAuthHost + '/oauth/token?grant_type=client_credentials';
					def jsonObjToken = readJSON text: getTokenResp.content
					token = "Bearer " + jsonObjToken.access_token
				   	} catch (Exception e) {
						error("Requesting the oauth token for Cloud Integration failed:\n${e}")
					}
					//delete the old flow content so that only the latest content gets stored
					dir(env.GITFolder + '/' + env.IntegrationFlowID){
						deleteDir();
					}
					//download and extract artefact from tenant
					println("Downloading artefact");
					def tempfile = UUID.randomUUID().toString() + ".zip";
					def cpiDownloadResponse = httpRequest acceptType: 'APPLICATION_ZIP', 
						customHeaders: [[maskValue: false, name: 'Authorization', value: token]], 
						ignoreSslErrors: false, 
						responseHandle: 'LEAVE_OPEN', 
						validResponseCodes: '100:399, 404',
						timeout: 30,  
						outputFile: tempfile,
						url: 'https://' + env.CPIHost + '/api/v1/IntegrationDesigntimeArtifacts(Id=\''+ env.IntegrationFlowID + '\',Version=\'active\')/$value';
					if (cpiDownloadResponse.status == 404){
						//invalid Flow ID
						error("Received http status code 404. Please check if the Artefact ID that you have provided exists on the tenant.");
					}
					def disposition = cpiDownloadResponse.headers.toString();
					def index=disposition.indexOf('filename')+9;
					def lastindex=disposition.indexOf('.zip', index);
					def filename=disposition.substring(index + 1, lastindex + 4);
					def folder=env.GITFolder + '/' + filename.substring(0, filename.indexOf('.zip'));
					fileOperations([fileUnZipOperation(filePath: tempfile, targetLocation: folder)])
					cpiDownloadResponse.close();

					//remove the zip
					fileOperations([fileDeleteOperation(excludes: '', includes: tempfile)])
						
					dir(folder){
						bat 'git add .'
					}
					println("Store integration artefact in Git")
					withCredentials([string(credentialsId: env.GITToken, variable: 'GIT_TOKEN')]) {  
						bat '''
						  git checkout ${GITBranch} || git checkout -b ${GITBranch}
                          git diff-index --quiet HEAD || git commit -m "Integration Artefacts update from CICD pipeline"
       					  git status
						  git remote -v
        				  git remote set-url origin https://$GIT_TOKEN@${GITRepositoryURL}
       					  git push origin HEAD:${env.GITBranch}
                        '''
						// echo "About to push changes"
						// bat('git push https://${GIT_TOKEN}@' + env.GITRepositoryURL + ' HEAD:' + env.GITBranch)
					}				
				}
			}
		}
    }
}
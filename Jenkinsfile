pipeline {
  agent any

  //Configure the following environment variables before executing the Jenkins Job
  environment {
    IntegrationFlowID = "S0024648617-flow4"
    GetEndpoint = false //If you don't need the endpoint or the artefact does not provide an endpoint, set the value to false
    DeploymentCheckRetryCounter = 20 //multiply by 3 to get the maximum deployment time
	  CPIHost = "${env.CPI_HOST}"
	  CPIOAuthHost = "${env.CPI_OAUTH_HOST}"
	  CPIOAuthCredentials = "${env.CPI_OAUTH_CRED}"	
  }

  stages {
    stage('Generate oauth bearer token') {
      steps {
        script {
          //get oauth token for Cloud Integration
          println("requesting oauth token");
          def getTokenResp = httpRequest acceptType: 'APPLICATION_JSON',
            authentication: "${env.CPIOAuthCredentials}",
            contentType: 'APPLICATION_JSON',
            httpMode: 'POST',
            responseHandle: 'LEAVE_OPEN',
            timeout: 30,
            url: 'https://' + env.CPIOAuthHost + '/oauth/token?grant_type=client_credentials';
          def jsonObjToken = readJSON text: getTokenResp.content
          def token = "Bearer " + jsonObjToken.access_token
          env.token = token
          getTokenResp.close();
        }
      }
    }

    stage('Deploy integration flow and check for deployment success') {
      steps {
        script {
          //deploy integration flow as specified in the configuration
          println("Deploying integration flow.");
          def deployResp = httpRequest httpMode: 'POST',
            customHeaders: [
              [maskValue: false, name: 'Authorization', value: env.token]
            ],
            ignoreSslErrors: true,
            timeout: 30,
            url: 'https://' + "${env.CPIHost}" + '/api/v1/DeployIntegrationDesigntimeArtifact?Id=\'' + "${env.IntegrationFlowID}" + '\'&Version=\'active\'';

          //check deployment status
          println("Start checking integration flow deployment status.");
          Integer counter = 0;
          def deploymentStatus;
          def continueLoop = false;
		  
		      //check until max check counter reached or we have a final state
          while (counter < env.DeploymentCheckRetryCounter.toInteger() & continueLoop == true) {
            Thread.sleep(3000);
            counter = counter + 1;
            def statusResp = httpRequest acceptType: 'APPLICATION_JSON',
              customHeaders: [
                [maskValue: false, name: 'Authorization', value: env.token]
              ],
              httpMode: 'GET',
              responseHandle: 'LEAVE_OPEN',
              timeout: 30,
              url: 'https://' + "${env.CPIHost}" + '/api/v1/IntegrationRuntimeArtifacts(\'' + "${env.IntegrationFlowID}" + '\')';
            def jsonObj = readJSON text: statusResp.content;
            deploymentStatus = jsonObj.d.Status;

            println("Deployment status: " + deploymentStatus);
			
            
          }
		     
           
        }
      }
    }
  }
}
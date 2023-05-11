pipeline {
  agent any

  //Configure the following environment variables before executing the Jenkins Job
  environment {
    IntegrationFlowID = "S0024648617-flow4"
    FailJobOnFailedMPL = true //if you are expecting your message to fail, set this to false, so that your job won't fail
    DeploymentCheckRetryCounter = 50 //multiply by 3 to get the maximum deployment time
    MPLCheckRetryCounter = 20 //multiply by 3 to get the maximum processing time. Example: 10 would be sufficient for message processings <30s
    CPIHost = "${env.CPI_HOST}"
    CPIOAuthHost = "${env.CPI_OAUTH_HOST}"
    CPIOAuthCredentials = "${env.CPI_OAUTH_CRED}"
  }

  stages {
    stage('Generate oauth bearer token') {
      steps {
        script {
          //Get Oauth token
          try {
            def getTokenResp = httpRequest acceptType: 'APPLICATION_JSON',
              authentication: env.CPIOAuthCredentials,
              contentType: 'APPLICATION_JSON',
              httpMode: 'POST',
              responseHandle: 'LEAVE_OPEN',
              timeout: 30,
              url: 'https://' + env.CPIOAuthHost + '/oauth/token?grant_type=client_credentials';
            def jsonObjToken = readJSON text: getTokenResp.content
            def token = "Bearer " + jsonObjToken.access_token
            env.token = token
            getTokenResp.close();
          } catch (Exception e) {
            error("Oauth token generation failed:\n${e}")
          }
        }
      }
    }

    stage('Deploy Iflow and check for deployment success else fail the job') {
      steps {
        script {
          //Deploy integration artefact
          println("Deploying integration flow");
          try {
            def deployResp = httpRequest httpMode: 'POST',
              customHeaders: [
                [maskValue: false, name: 'Authorization', value: env.token]
              ],
              ignoreSslErrors: true,
              timeout: 30,
              url: 'https://' + env.CPIHost + '/api/v1/DeployIntegrationDesigntimeArtifact?Id=\'' + env.IntegrationFlowID + '\'&Version=\'active\'';
          } catch (Exception e) {
            error("Deploying the integration flow failed:\n${e}")
          }
          //check deployment status
          println("Start checking integration flow status.");
          Integer counter = 0;
          def deploymentStatus;
          def continueLoop = false
		      //turning loops until we get a final status
          while (counter < env.DeploymentCheckRetryCounter.toInteger() & continueLoop == true) {
            Thread.sleep(3000);
            counter = counter + 1;
            def checkDeploymentResp = httpRequest acceptType: 'APPLICATION_JSON',
              customHeaders: [
                [maskValue: false, name: 'Authorization', value: env.token]
              ],
              httpMode: 'GET',
              responseHandle: 'LEAVE_OPEN',
              timeout: 30,
              url: 'https://' + env.CPIHost + '/api/v1/IntegrationRuntimeArtifacts(\'' + env.IntegrationFlowID + '\')';

            def jsonObj = readJSON text: checkDeploymentResp.content;
            deploymentStatus = jsonObj.d.Status;

            println("Deployment status: " + deploymentStatus);
			
            if (deploymentStatus.equals("Error")) {
              //In case of error, get the error details
              def deploymentErrorResp = httpRequest acceptType: 'APPLICATION_JSON',
                customHeaders: [
                  [maskValue: false, name: 'Authorization', value: env.token]
                ],
                httpMode: 'GET',
                responseHandle: 'LEAVE_OPEN',
                timeout: 30,
                url: 'https://' + env.CPIHost + '/api/v1/IntegrationRuntimeArtifacts(\'' + env.IntegrationFlowID + '\')' + '/ErrorInformation/$value';
              def jsonErrObj = readJSON text: deploymentErrorResp.content
              def deployErrorInfo = jsonErrObj.parameter
              checkDeploymentResp.close();
              deploymentErrorResp.close();
              error("Error Details: " + deployErrorInfo);
            } else if (deploymentStatus.equals("Started")) {
              println("Integration flow deployment successful")
              checkDeploymentResp.close();
              continueLoop = false
            } else {
              println("The integration flow is not yet started. Will wait 3s and then check again.")
            }
          }
          if (!deploymentStatus.equals("Started")) {
            error("No final deployment status reached. Current status: \'" + deploymentStatus);
          }
        }
      }
    }
    stage('Check Message processing status and fail if not completed') {
      steps {
        script {
          println("Checking message processing log status");
         
          Integer counter = 0;
          def mPLStatus = '';
          def continueLoop = false
          def mplId = '';
          while (counter < env.MPLCheckRetryCounter.toInteger() & continueLoop == true) {
            //get the latest MPL (excluding those in status Discarded
            try {
              def checkMPLResp = httpRequest acceptType: 'APPLICATION_JSON',
                customHeaders: [
                  [maskValue: false, name: 'Authorization', value: env.token]
                ],
                httpMode: 'GET',
                responseHandle: 'LEAVE_OPEN',
                timeout: 30,
                url: 'https://' + env.CPIHost + '/api/v1/MessageProcessingLogs?$filter=IntegrationArtifact/Id%20eq%20\'' + env.IntegrationFlowID + '\'and%20Status%20ne%20\'DISCARDED\'&$orderby=LogEnd+desc&$top=1';
           
              //extract MPL Status
              def jsonMPLStatus = readJSON text: checkMPLResp.content
              jsonMPLStatus.d.results.each {
                value ->
                  mplStatus = value.Status;
                mplId = value.MessageGuid;
                //if status processing, keep going
                if (mplStatus.equalsIgnoreCase("Processing")) {
                  println("message processing not over yet, trying again in a short moment");
                  Thread.sleep(3000);
                  counter = counter + 1;
                } else {
                  //we got a final state, ending the loop
                  continueLoop = false;
                  checkMPLResp.close();
                }
              }
              println("Final message status of MPL ID \'" + mplId + "\' : \'" + mplStatus + "\'");
              if (mplStatus.equalsIgnoreCase("Processing")) {
                error("The message processing did not finish within the check frame. If it is a long running flow, increase the retry counter in the job configuration.");
              } else if (mplStatus.equalsIgnoreCase("Failed") || mplStatus.equalsIgnoreCase("Retry")) {
                //get error information
                def cpiMplError = httpRequest acceptType: 'APPLICATION_ZIP',
                  contentType: 'APPLICATION_ZIP',
                  customHeaders: [
                    [maskValue: false, name: 'Authorization', value: env.token]
                  ],
                  ignoreSslErrors: false,
                  responseHandle: 'LEAVE_OPEN',
                  timeout: 30,
                  url: 'https://' + env.CPIHost + '/api/v1/MessageProcessingLogs(\'' + mplId + '\')/ErrorInformation/$value';
                println("Message processing failed! Error information: " + cpiMplError.content);
                cpiMplError.close();
				        if (env.FailJobOnFailedMPL.equalsIgnoreCase("true")) {
                  error("The job is configured to fail on a failed MPL. Stopping now.");
                }
              } else if (mplStatus.equalsIgnoreCase("Abandoned")) {
                error("Message processing not successful. It seems as the processing was interrupted.");
              } else if (mplStatus.equalsIgnoreCase("Completed")) {
                println("Message processing successful");
              } else {
                error("Kindly check the tenant, something went wrong");
              }
            } catch (Exception e) {
              error("Determination of message processing log status failed:\n${e}")
            }
          }
        }
      }
    }
  }
}
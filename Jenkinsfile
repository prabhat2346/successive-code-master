node('master'){

		def environment=ENVIRONMENT_VAR
		def serviceName="trigger"
		echo "${serviceName}"

		env.POD_TURN = 6
		//UAT Regisrty Details.
		def imageName = "trigger"
		env.UAT_IMAGE_TAG = "${env.uatDockerRegistry}/${imageName}"
		env.STG_IMAGE_TAG = "${env.stgDockerRegistry}/${imageName}"
		env.PRD_IMAGE_TAG = "${env.prdDockerRegistry}/${imageName}"

		def imageTag=""
		def imagePostfix=""
		def dockerRegistry=""
		def fileName=""
		def dockerCredential=""
		def servicePrincipal=""
		def deploymentDataDir=""

		def lastPod = 0
		

		def active_profile="${environment}"

		def send_mail = { subject, body ->
			mail to: 'agnivesh.tiwari@india.nec.com',
			subject: "${subject}",
			body: "${body}"
		}

		def failure = { log, title ->
			send_mail(log, title);
		}

		//Env spesfication
		switch(ENVIRONMENT_VAR) {
		  case "uat":
		    imageTag=env.UAT_IMAGE_TAG 
		    imagePostfix="uat"
		    dockerRegistry=env.uatDockerRegistry
		    fileName="uat-build.log"
		    dockerCredential="docker-uat"
		    servicePrincipal="sp-uat"
		    deploymentDataDir=env.uatDeploymentDataDir
		    break
		  case "stg":
		    imageTag=env.STG_IMAGE_TAG
		    imagePostfix="stg"
		    dockerRegistry=env.stgDockerRegistry
		    fileName="stg-build.log"
		    dockerCredential="docker-stg"
		    servicePrincipal="sp-stg"
		    deploymentDataDir=env.stgDeploymentDataDir
		    break
		  case "prd":
		    imageTag=env.PRD_IMAGE_TAG
		    imagePostfix="prd"
		    dockerRegistry=env.prdDockerRegistry
		    fileName="prd-build.log"
		    dockerCredential="docker-prd"
		    servicePrincipal="sp-prd"
		    deploymentDataDir=env.prdDeploymentDataDir
		    break
		  default:
		    error("Please provide a currect environment")
		    break
		}



		stage('SCM') {
			checkout scm
			}

		

		stage('Unit Test') {
				try{
				sh """
				cd ./Utility/HttpClient
				mvn clean install > text.log
				cd ../..
				cd ./cloudcommunication
				mvn clean install > test.log
				cd ..
				cd ./clips
				mvn install:install-file -Dfile=CLIPSJNI.jar -DgroupId=com.rj.clips -DartifactId=clips -Dversion=1.0 -Dpackaging=jar 
				cd ..
				cd ./trigger
				mvn test > test.log
				"""
			}
			catch(e) {		//If Unit Testing failed. Then sending mail to the Developer and DevOps Team.
				def logs = readFile('test.log')
				subject= "${currentBuild.projectName} - Unit Test # FAILED! for ${environment}"
				body= "Hi there,\n\nPlease find the below logs:\n\n${logs}\n\nRegards,\nALP Devops Team"
				send_mail(subject, body)
				error('Failed to execute test cases'+e.toString())
			}
		}

		stage('SonarQube analysis') {
			
			 sh """
			 cd ./Utility/HttpClient
			 mvn clean verify sonar:sonar -Dsonar.scm.disabled=true -DskipTests
			 """
			 sh """
			 cd ./cloudcommunication
			 mvn clean verify sonar:sonar -Dsonar.scm.disabled=true -DskipTests
			 """
			 sh """
			 cd ./trigger
			 mvn clean verify sonar:sonar -Dsonar.scm.disabled=true -DskipTests
			 """
			 
		}
		
		//Build & Push process for UAT Environment.
    stage('Build & Push on ${environment}') {
		 

		try{
			//Building and creating Docker Image for UAT Environment while capturing the logs in uat-build.log.
			sh"""
			cp ./clips/libCLIPSJNI.so ./trigger
			cd ./trigger
			mvn clean install -DskipTests  "-Dimage.active.profile=${active_profile}" "-Ddocker.image.prefix=${dockerRegistry}"  dockerfile:build > ${fileName}
			"""
			//reading version number from properties file.
			script {
	      VERSION = sh(returnStdout: true, script: '''
	      		cd ./trigger
				cat version.properties | grep revision | cut -d "=" -f2 ''')
	    }
			//defining IMAGE name for UAT Environment & pushing the Image.
			def image = imageTag+":"+VERSION.toInteger()
			echo "${image}"
			withCredentials([usernamePassword(credentialsId: dockerCredential, passwordVariable: 'pass', usernameVariable: 'user')]) {
				sh"""
					docker login ${user}.azurecr.io -u ${user} -p ${pass}
					echo ${image}
					docker push "${image}"
				"""
			}
			currentBuild.result="Success"
			  subject= "${currentBuild.projectName} - Build # ${currentBuild.number} - ${currentBuild.result}!"
		      body= "Hi there,\n\nBuild has Success for ${serviceName} on ${environment}\n\nRegards,\nALP Devops Team"
		      send_mail(subject, body)
		} catch(e) {
				//If build fails or something went wrong mail to Developer & DevOps Team.
			  currentBuild.result = "FAILED"
			  def build_logs = readFile("./trigger/"+fileName)
		      subject= "${currentBuild.projectName} - Build # ${currentBuild.number} - ${currentBuild.result}!"
		      body= "Hi there,\n\nBuild has failed for ${serviceName}.\nPlease find logs below:\n\n${build_logs}\n\nRegards,\nALP Devops Team"
		      send_mail(subject, body)
		      error('Build Failed')
					echo ''+e.toString()
			}
    }
    	
}

node {
    files = ['deploy.yml']

    withCredentials([usernamePassword(credentialsId: 'prisma_cloud', passwordVariable: 'PC_PASS', usernameVariable: 'PC_USER')]) {
    PC_TOKEN = sh(script:"curl -s -k -H 'Content-Type: application/json' -H 'accept: application/json' --data '{\"username\":\"$PC_USER\", \"password\":\"$PC_PASS\"}' https://${AppStack}/login | jq --raw-output .token", returnStdout:true).trim()
    }

    stage('Clone repository') {
        checkout scm
    }


    stage('Check image Git dependencies has no vulnerabilities') {
        try {
            withCredentials([usernamePassword(credentialsId: 'twistlock_creds', passwordVariable: 'TL_PASS', usernameVariable: 'TL_USER')]) {
                sh('chmod +x files/checkGit.sh && ./files/checkGit.sh')
            }
        } catch (err) {
            echo err.getMessage()
            echo "Error detected"
			throw RuntimeException("Build failed for some specific reason!")
        }
    }

    //$PC_USER,$PC_PASS,$PC_CONSOLE when Galileo is released. 
    stage('Apply security policies (Policy-as-Code) for evilpetclinic') {
        withCredentials([usernamePassword(credentialsId: 'twistlock_creds', passwordVariable: 'TL_PASS', usernameVariable: 'TL_USER')]) {
            sh('chmod +x files/addPolicies.sh && ./files/addPolicies.sh')
        }
    }

    //$PC_USER,$PC_PASS,$PC_CONSOLE when Galileo is released. 
    stage('Download latest twistcli') {
        withCredentials([usernamePassword(credentialsId: 'prisma_cloud', passwordVariable: 'PC_PASS', usernameVariable: 'PC_USER')]) {
            sh 'curl -k -u $PC_USER:$PC_PASS --output ./twistcli https://$PC_CONSOLE/api/v1/util/twistcli'
            sh 'sudo chmod a+x ./twistcli'
        }
    }
	

stage('Scan image with twistcli and Publish results to Jenkins') {
      try {
	    sh 'docker pull itresoldi/evilpetclinic:latest'  
            withCredentials([usernamePassword(credentialsId: 'twistlock_creds', passwordVariable: 'TL_PASS', usernameVariable: 'TL_USER')]) {
            prismaCloudScanImage ca: '', cert: '', dockerAddress: 'unix:///var/run/docker.sock', ignoreImageBuildTime: true, image: 'itresoldi/evilpetclinic:latest', key: '', logLevel: 'debug', podmanPath: '', project: '', resultsFile: 'prisma-cloud-scan-results.json'
	    sh 'curl -k -u $TL_USER:$TL_PASS --output ./twistcli https://$TL_CONSOLE/api/v1/util/twistcli'
            sh 'sudo chmod a+x ./twistcli'
            sh "./twistcli images scan --u $TL_USER --p $TL_PASS --address https://$TL_CONSOLE --details itresoldi/evilpetclinic:latest"
            }
	 } finally {
            prismaCloudPublish resultsFilePattern: 'prisma-cloud-scan-results.json'
        }
}

stage('Sandbox analysis of the Container Image') { 
	sh "./twistcli sandbox --u $TL_USER --p $TL_PASS --address https://$TL_CONSOLE itresoldi/evilpetclinic:latest"

stage('Scan K8s yaml manifest with Bridgecrew/checkov') {
	withDockerContainer(image: 'bridgecrew/jenkins_bridgecrew_runner:latest') {              
	sh "/run.sh aaba9c95-c632-5403-9762-24bbcd0a4611 https://github.com/ivan-tresoldi/shiftleftdemo"
	}
}

stage('Deploy evilpetclinic') {
        sh 'kubectl create ns evil --dry-run -o yaml | kubectl apply -f -'
        sh 'kubectl delete --ignore-not-found=true -f files/deploy.yml -n evil'
        sh 'kubectl apply -f files/deploy.yml -n evil'
        sh 'sleep 10'
    }

stage('Run bad Runtime attacks') {
        sh('chmod +x files/runtime_attacks.sh && ./files/runtime_attacks.sh')
    }

    stage('Run bad HTTP stuff for WAAS to catch') {
        sh('chmod +x files/waas_attacks.sh && ./files/waas_attacks.sh')
    }
	
}

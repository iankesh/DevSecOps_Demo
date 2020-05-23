pipeline 
{
    agent any
    tools 
    { 
        maven 'Maven' 
    }
    stages 
    {
        stage ('Initialize') 
        {
            steps 
            {
                sh '''
                    echo "PATH = ${PATH}"
                    echo "M2_HOME = ${M2_HOME}"
                    echo ${USER}
                ''' 
            }
        }
        stage ('Trufflehog: Check-Git-Secrets') 
        {
		    steps 
            {
	            sh 'rm trufflehog || true'
		        sh 'docker pull gesellix/trufflehog'
		        sh 'docker run -t gesellix/trufflehog --json https://github.com/iankesh/DevSecOps_Demo.git > trufflehog'
		        sh 'cat trufflehog'
	        }
	    }
        stage ('OWASP-Dependency-Checker: Source-Composition-Analysis') {
		    steps 
            {
                sh 'rm owasp-* || true'
                sh 'wget https://raw.githubusercontent.com/devopssecure/webapp/master/owasp-dependency-check.sh'	
                sh 'chmod +x owasp-dependency-check.sh'
                sh 'bash owasp-dependency-check.sh'
                sh 'cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml'
		    }
	    }
        stage ('Sonarqube: Static-Application-Security-Testing')
        {
		    steps 
            {
		        withSonarQubeEnv('sonarqube') 
                {
                    sh 'mvn sonar:sonar'
                    sh 'cat target/sonar/report-task.txt'
		        }
		    }
	    }
        stage ('Build') 
        {
            steps 
            {
                sh 'mvn clean package'
            }
        }  
        stage ('Deploy-To-Tomcat') 
        {
            steps 
            {
                sshagent(['tomcat-key']) 
                {
                    sh 'scp -o StrictHostKeyChecking=no target/WebApp.war devops@104.46.51.176:/home/devops/apache-tomcat-8.5.38/webapps/webapp.war'
                }      
            }       
        }
        stage ('Nmap: Web-Server-Port-Scan') 
        {
		    steps 
            {
                sh 'rm nmap* || true'
                sh 'docker run --rm -v "$(pwd)":/data uzyexe/nmap -sS -sV -oX nmap 104.46.51.176'
                sh 'cat nmap'
		    }
	    }
        stage ('ZAP-Baseline: Dynamic-Application-Security-Testing') 
        {
		    steps 
            {    
                sh 'docker run -t owasp/zap2docker-stable zap-baseline.py -t http://104.46.51.176:8080/webapp/ || true'
			}
		} 
        stage ('Nikto-Web-Scan: Web-Server-Vulnerabilities') 
        {
		    steps 
            {
                sh 'rm nikto-output.xml || true'
                sh 'docker pull secfigo/nikto:latest'
                sh 'docker run --user $(id -u):$(id -g) --rm -v $(pwd):/report -i secfigo/nikto:latest -h 104.46.51.176 -p 8080 -output /report/nikto-output.xml'
                sh 'cat nikto-output.xml'   
		    }
	    } 
        stage ('SSL Checks') 
        {
		    steps 
            {
                sh 'pip install sslyze==1.4.2'
                sh 'python -m sslyze --regular 104.46.51.176:8080 --json_out sslyze-output.json'
                sh 'cat sslyze-output.json'
		    }
	    }
        stage ('Upload Reports to Defect Dojo') 
        {
		    steps 
            {
                sh 'pip install requests'
                sh 'wget https://raw.githubusercontent.com/devopssecure/webapp/master/upload-results.py'
                sh 'chmod +x upload-results.py'
                sh 'python upload-results.py --host 52.174.83.134 --api_key 28228a0cba3731814a31f53cd09151dba624a1ef --engagement_id 4 --result_file trufflehog --username admin --scanner "SSL Labs Scan"'
                sh 'python upload-results.py --host 52.174.83.134 --api_key 28228a0cba3731814a31f53cd09151dba624a1ef --engagement_id 4 --result_file /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml --username admin --scanner "Dependency Check Scan"'
                sh 'python upload-results.py --host 52.174.83.134 --api_key 28228a0cba3731814a31f53cd09151dba624a1ef --engagement_id 4 --result_file nmap --username admin --scanner "Nmap Scan"'
                sh 'python upload-results.py --host 52.174.83.134 --api_key 28228a0cba3731814a31f53cd09151dba624a1ef --engagement_id 4 --result_file sslyze-output.json --username admin --scanner "SSL Labs Scan"'
                sh 'python upload-results.py --host 52.174.83.134 --api_key 28228a0cba3731814a31f53cd09151dba624a1ef --engagement_id 4 --result_file nikto-output.xml --username admin' 
		    }
	    }
        stage ('Clean Running Docker Containers') 
        {
		    steps 
            {
                sh 'docker rm -f $(docker ps -a | grep gesellix/trufflehog | cut -d " " -f 1) || true'
                sh 'docker rm -f $(docker ps -a | grep owasp/zap2docker-stable | cut -d " " -f 1) || true'
                sh 'docker rm -f $(docker ps -a | grep uzyexe/nmap | cut -d " " -f 1) || true'
                sh 'docker rm -f $(docker ps -a | grep secfigo/nikto | cut -d " " -f 1) || true'
                sh 'docker rm -f $(docker ps -a | grep owasp/dependency-check | cut -d " " -f 1) || true'
		    }
	    }
    }
}
pipeline 
{
    agent any
    tools 
    { 
        maven 'Maven' 
    }
    stages 
    {
        stage ('Initialize Step') 
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
        stage ('Pre-Build Checks')
        {
            parallel
            {
                stage ('TRUFFLEHOG: Check-Git-Secrets') 
                {
                    steps 
                    {
                        sh 'rm trufflehog* || true'
                        sh 'docker pull gesellix/trufflehog'
                        sh 'docker run -t gesellix/trufflehog --json https://github.com/iankesh/DevSecOps_Demo.git > trufflehog.json'
                        sh 'cat trufflehog.json'
                    }
                }
                stage ('OWASP DEPENDENCY CHECKER: Source-Composition-Analysis') {
                    steps 
                    {
                        sh 'rm owasp-* || true'
                        sh 'wget https://raw.githubusercontent.com/iankesh/DevSecOps_Demo/master/owasp-dependency-check.sh'	
                        sh 'chmod +x owasp-dependency-check.sh'
                        sh 'bash owasp-dependency-check.sh'
                        sh 'cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml'
                    }
                }
                stage ('SONARQUBE: Static-Application-Security-Testing')
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
            }
        }
        stage ('BUILD Step') 
        {
            steps 
            {
                sh 'mvn clean package'
            }
        }  
        stage ('Deploy-To-Tomcat Step') 
        {
            steps 
            {
                sshagent(['tomcat-key']) 
                {
                    sh 'scp -o StrictHostKeyChecking=no target/WebApp.war devops@51.136.57.150:/home/devops/apache-tomcat-8.5.55/webapps/webapp.war'
                }      
            }       
        }
        stage ('NMAP: Web-Server-Port-Scan') 
        {
		    steps 
            {
                sh 'rm nmap* || true'
                sh 'docker run --rm -v "$(pwd)":/data uzyexe/nmap -sS -sV -oX nmap 51.136.57.150'
                sh 'cat nmap'
		    }
	    }
        stage ('ZAP BASELINE: Dynamic-Application-Security-Testing') 
        {
		    steps 
            {    
                sh 'docker run -t owasp/zap2docker-stable zap-baseline.py -t http://51.136.57.150:8080/webapp/ || true'
			}
		} 
        stage ('NIKTO WEB SCAN: Web-Server-Vulnerabilities') 
        {
		    steps 
            {
                sh 'rm nikto-output.xml || true'
                sh 'docker pull secfigo/nikto:latest'
                sh 'docker run --user $(id -u):$(id -g) --rm -v $(pwd):/report -i secfigo/nikto:latest -h 51.136.57.150 -p 8080 -output /report/nikto-output.xml'
                sh 'cat nikto-output.xml'   
		    }
	    } 
        stage ('SSL CHECKS: Checking ssl') 
        {
		    steps 
            {
                sh 'rm -rf ~/.local || true'
                sh 'pip install sslyze==1.4.2'
                sh 'python -m sslyze --regular 51.136.57.150:8080 --json_out sslyze-output.json'
                sh 'cat sslyze-output.json'
		    }
	    }
	    stage ('DEFECT DOJO: Upload-Reports') 
        {
            when 
            {
                branch 'develop'
            }
		    steps 
            {
                sh 'rm upload-results.py* || true'
                sh 'pip install requests'
                sh 'wget https://raw.githubusercontent.com/iankesh/DevSecOps_Demo/master/upload-results.py'
                sh 'chmod +x upload-results.py'
                sh 'python upload-results.py --host 52.174.83.134:8081 --api_key 28228a0cba3731814a31f53cd09151dba624a1ef --engagement_id 1 --result_file trufflehog.json --username admin --scanner "SSL Labs Scan" || true'
                sh 'python upload-results.py --host 52.174.83.134:8081 --api_key 28228a0cba3731814a31f53cd09151dba624a1ef --engagement_id 1 --result_file /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml --username admin --scanner "Dependency Check Scan" || true'
                sh 'python upload-results.py --host 52.174.83.134:8081 --api_key 28228a0cba3731814a31f53cd09151dba624a1ef --engagement_id 1 --result_file nmap --username admin --scanner "Nmap Scan" || true'
                sh 'python upload-results.py --host 52.174.83.134:8081 --api_key 28228a0cba3731814a31f53cd09151dba624a1ef --engagement_id 1 --result_file sslyze-output.json --username admin --scanner "SSL Labs Scan" || true'
                sh 'python upload-results.py --host 52.174.83.134:8081 --api_key 28228a0cba3731814a31f53cd09151dba624a1ef --engagement_id 1 --result_file nikto-output.xml --username admin --scanner "Nikto Scan" || true' 
		    }
	    }

        stage ('CLEAN: Running Docker Containers') 
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

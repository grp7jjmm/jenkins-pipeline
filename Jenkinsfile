pipeline {
    agent any
    
    // environment for sonarqube
    environment {
        PATH = "$PATH:/usr/share/maven/bin"
    }
    
    // only one build can be run at a time
    options {
      disableConcurrentBuilds abortPrevious: true
    }
    
    stages {
        
        // This stage pulls the client's Github Repository
        stage('GitHub') {
            
            // This build will timeout after 2 mins of idle interaction
            options {
                timeout(time: 2, unit: 'MINUTES')
            }
            
            // An input field will be produced to allow clients to put their Github repository URL
            steps {
                script {
                    env.URL = input message: 'Place your Github Repository here:', parameters: [string(defaultValue: '', name: 'Repository URL')]
                }
                git "${env.URL}"
            }
        }
        
        // This stage will package the maven project into a .war file
        stage('Static Analysis') {
            steps {
                
                // Snyk and DependencyCheck will scan the source code of the project for vulnerabilities and unneccesary dependencies
                snykSecurity additionalArguments: '-d', failOnError: false, failOnIssues: false, snykInstallation: 'Snyk', snykTokenId: '813bd878-dd5a-414c-b3e4-d7e300a5f2f1'
                dir ("/jenkins-scripts/.scripts"){
                    sh "./snyk.sh"
                }
                dependencyCheck additionalArguments: '--scan pom.xml --out /dcheck_reports --format HTML', odcInstallation: 'Dependency-Check'
                dir ("/jenkins-scripts/.scripts"){
                    sh "./dcheck.sh"
                }
                
                // After the scan, the project will be compiled into a war file
                dir ("/jenkins-scripts/.scripts"){
                    sh "./mvn_war.sh"
                }
                
                // Install WarningNextGen plugins for Testing stage
                sh 'mvn install checkstyle:checkstyle findbugs:findbugs pmd:pmd'
                
                // Initializing SonarQube static analysis tool 
                withSonarQubeEnv('SonarQube') { 
                    sh "mvn sonar:sonar"
                }
                
                // Initializing the plugins of the WarningNextGen security tool for hybrid analysis
                recordIssues(
                    enabledForFailure: true, aggregatingResults: true, 
                    tools: [java(), checkStyle(), findBugs(), pmdParser()]
                )
                
            }
        }
        
        stage('Build') {
            steps{
                
                // Compiling the client's project into a .war file
                sh 'mvn compile war:war'
               
                // Fingerprinting and hashing of the war file 
                fingerprint '**/*.war'
                dir ("/jenkins-scripts/.scripts"){
                    sh "./sha256hash.sh"
                }
                
            }
        }
        
        // This stage will release the compiled .war file to Tomcat
        stage('Release') {
            steps {
                
                // Checking if the .war file created is still the legitimate one from the Build Stage.
                fingerprint '**/*.war'
                
                // The build will fail if the hash values do not match
                script{
                    check = sh(script: "/jenkins-scripts/.scripts/checkhash.sh", returnStatus: true)
                    if (check != 0) {
                        currentBuild.result = 'FAILURE'
                    }
                }
                
                // Deploying the .war file to Tomcat (port 8081) 
                deploy adapters: [tomcat8(credentialsId: '9d2180bc-6df6-4e09-ae05-2a5ca9e590ca', path: '', url: 'http://localhost:8081/')], contextPath: 'mvnwebapp', onFailure: false, war: '**/*.war'
               
            }
        }
        
        stage('Dynamic Analysis') {
            steps{
                
                // Initialing the debugger within JMeter
                dir("/apache-jmeter-5.5/bin"){
                    sh "./jenkinsreport.sh"
                }
                
                // Execute Nikto for Network Scanning 
                dir ("/jenkins-scripts/.scripts"){
                    sh "./nikto.sh"
                }
                
                // Link to Client's Web Application
                echo "If you would like to see your released Web Application, you can find it here -> http://127.0.0.1:8081/mvnwebapp"
                
                // Link to the main HTML report
                echo "If you would like to see the reports of the various security tools, you can find them here -> http://127.0.0.1:8083/reports.php"
                
                // Copying the console output to the reports page
                dir ("/jenkins-scripts/.scripts"){
                    sh "./console.sh"
                }
                
            }
        }
        
    }
}

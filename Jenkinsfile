import java.text.SimpleDateFormat 
def startDate 
def endDate 
def projectId= "1" 
def projectName= "vote" 
def moduleId= "1" 
def moduleName= "Voting" 
def newBuildId= "3.0.0.${BUILD_ID}" 
def sonarUrl= "http://10.0.0.4:9000" 
def nameSpace="vote"

def CODE =  readJSON text: '{"url":"http://10.0.0.4:8888/gitlab/root/vote.git","scm":"gitlab","branch":"master","credId":"gitlab"}'
def BUILD =  readJSON text: '{"command":"docker build --tag '+moduleName+'_'+projectName+':'+newBuildId+' ."}'
def DEPLOYMENT =  readJSON text: '{"contextPath":""}'

pipeline {
agent {label 'kubernetesnode'}

	stages {

		stage('CODE'){
		        steps{
		            git url: CODE.url, credentialsId: CODE.credId, branch: CODE.branch
		        }

		}
		stage('Build'){
		       steps{
			   
					println "Building image ...."
		       		script{
						   startDate = new Date()
		       		sh '''
					echo ${nameSpace}
					docker build --tag vote:${BUILD_NUMBER} .
					'''
		       		}
		       		
		            //executeCmd(BUILD.command);
		       }
			   post{
				   always{
					   script{
					   		endDate = new Date()
							adoptBuildFeedback  buildDisplayName: "vote.${BUILD_NUMBER}",
						 						buildStartedAt: "${startDate}",
						 						status: "${currentBuild.currentResult}",
						 						buildEndedAt: "${endDate}",
						 						buildUrl: "${BUILD_URL}"+"#"+CODE.url,
						 						projectId: projectId
					   }
				   }

				}
			  
		}
		stage('SonarQube analysis') {
				steps {
					println "Building code ..."
					withSonarQubeEnv('SonarQube') {
						executeCmd("/opt/sonar-scanner/bin/sonar-scanner -Dsonar.projectKey=voting.vote");  
					}
					 timeout(time: 1, unit: 'HOURS') {
						waitForQualityGate abortPipeline: true
					}
				}
				post{
				   success{
					   script{
						 adoptCodeAnalysisFeedback  buildDisplayName: "vote.${BUILD_NUMBER}",
						 							buildUrl: "${BUILD_URL}",
						 							sonarKey: "voting.vote",
						 							projectId: projectId
						 							
					   }
				   }
				 }
		}
    stage('Container Analysis') {
        steps{
                script{
                    def TRIVY =  readJSON text: sh( script: 'trivy -q image -f template -t \'{{- $critical := 0 }}{{- $high := 0 }}{{- $low := 0 }}{{- range . }}{{- range .Vulnerabilities }}{{- if  eq .Severity "CRITICAL" }}{{- $critical = add $critical 1 }}{{- end }}{{- if  eq .Severity "HIGH" }}{{- $high = add $high 1 }}{{- end }}{{- if  eq .Severity "LOW" }}{{- $low = add $low 1 }}{{- end }}{{- end }}{{- end }}{Critical: {{ $critical }}, High: {{ $high }}, Low:{{ $low }}}\' dockersamples/examplevotingapp_worker',returnStdout:true).trim()
                    env.HIGH = TRIVY.High
                    env.CRITICAL = TRIVY.Critical
                    env.LOW = TRIVY.Low
                }
            }
            post{
                    always{
                        script{
                                  adoptContainerAnalysisFeedback buildDisplayName: "vote.${BUILD_NUMBER}",
                                                                    projectId: projectId,
                                                                    critical: "${CRITICAL}",
                                                                    high: "${HIGH}",
                                                                    low:  "${LOW}"    
                        }
                    }
                            
            }
    }
	   stage('Kube_deployment') {
		steps{
			script {
				sh '''
				kubectl set image deployments/vote vote=vote:${BUILD_NUMBER} --namespace=''' +nameSpace+ '''
				echo " >> Pods"
				kubectl get pods --namespace=''' +nameSpace+ '''
				'''
				}
			}
			post{
		                always{
								script{
                                		startDate = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSS'Z'").format(new Date())
                                		adoptDevDeploymentFeedback  buildDisplayName: "vote.${BUILD_NUMBER}", 
                                									deploymentStartedAt: startDate,
                                									environment: "DEV",
                                									moduleId: moduleId,
                                									moduleName: moduleName,
                                									projectName: projectName,
                                									status: "${currentBuild.currentResult}"
								}
							}
		                   
		            }
		}
	}
}

//Helper Methods

void executeCmd(String CMD){
	if(isUnix()){
		sh "echo linux"
		sh CMD
	}
	else{
		 bat "echo windows"
		 bat CMD
	}
}

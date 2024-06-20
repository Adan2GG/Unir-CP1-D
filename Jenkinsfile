Pipeline
pipeline {   
     environment {
        GIT_CREDENTIALS = 'miTokenGitHub' // Reemplaza con el ID de tus credenciales en Jenkins
        GIT_REPOR_URL = 'https://github.com/Adan2GG/unir-CP1-C.git' // Reemplaza con la URL de tu repositorio
        BRANCH = 'develop' // Reemplaza con la rama a la que deseas hacer push
        STACK_NAME = 'staging-todo-list-aws'     // Cambia esto al nombre de tu stack SAM
    }

     stages {
        stage('Get Code') {
            steps {
                checkout([$class: 'GitSCM',
                          branches: [[name: "${BRANCH}"]],
                          doGenerateSubmoduleConfigurations: false,
                          extensions: [],
                          userRemoteConfigs: [[url: "${GIT_REPOR_URL}", credentialsId: "${GIT_CREDENTIALS}"]]
                ])
                stash name:'code', includes:'**'
            }
        }
        stage('Static Test') {
                stage('Static') {
                    steps{
                        unstash name:'code'
                        script{
                           sh'''
                           set PYTHONPATH=.
                           python -m flake8 --format=pylint --extend-ignore E501 --exit-zero  src >flake8.out
                            '''
                            catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                                recordIssues tools: [flake8(name: 'Flake8', pattern: 'flake8.out')], qualityGates: [[threshold: 8, type: 'TOTAL', unstable: true],[threshold:10, type: 'TOTAL', unstable: false]]
                            }
                        }
                    }
                }
                stage('Security') {
                    steps {
                     unstash name:'code'
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        script{
                           sh'''
                             python -m bandit -r .  --exit-zero -f custom -o bandit.out --severity-level medium --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                            '''
                        }
                        recordIssues tools: [pyLint(name: 'bandit', pattern: 'bandit.out')], qualityGates: [[threshold:2, type: 'TOTAL', unstable: true], [threshold: 4, type: 'TOTAL', unstable: true]]
                        }
                    }
                }
        }
        stage ('Deploy'){
            steps{
                sh 'sam build'
                sh 'sam deploy --no-fail-on-empty-changeset || true --config-file samconfig.toml'
                //Obtenemos las urls del servicio desplegado en con sam
                script {
                    def stackInfo = sh(script: "aws cloudformation describe-stacks --stack-name ${STACK_NAME}", returnStdout: true).trim()
                    def output = readJSON(text: stackInfo)
                    def apiUrl = output.Stacks[0].Outputs.find { it.OutputKey == 'BaseUrlApi' }?.OutputValue

                    if (apiUrl) {
                        env.BASE_URL = apiUrl // Guardar el valor en la variable de entorno
                        echo "API URL: ${env.BASE_URL}"
                    } else {
                        error "API URL not found in stack outputs"
                    }
                }
            }
        }
        stage('Tests Rest') {
                  steps {
                         unstash name:'code'
                        script {
                              def stageName = env.STAGE_NAME
                              echo "Stage: ${stageName}"
                              def nodeName = env.NODE_NAME
                              echo "Agent: ${nodeName}"
                              }
                               script{
                                    sh'''
                                          python -m pytest --junitxml=result-rest.xml test/integration/todoApiTest.py
                                    '''
                           }
                        }
            }
            stage('Promote') {
                  steps{
                        git url:"${GIT_REPOR_URL}", branch: 'develop', credentialsId:"{GIT_CREDENTIALS}"
                script {
                    echo"***Config global merge"
                    sh git config --global merge.ours.driver true
                        echo "***Checkout Develop Branch***"
                    sh 'git checkout develop'
                    sh 'git pull origin develop'
                        echo "***CheckOut Master***"
                        sh 'git checkout master'
                    sh 'git pull origin master'
                        echo "***Merge Develop into Master***"
                        sh 'git merge develop'
                        echo "***Push Master***"
                        withCredentials([string(credentialsId:"${GIT_CREDENTIALS}",variable: 'GIT_TOKEN')]){
                    sh 'git push https://${GIT_TOKEN}@github.com/Adan2GG/unir-CP1-C.git master'
                   }                  
                }
                        
                  }
            }
        post { 
            always { 
                echo 'Clean env: delete dir'
                cleanWs()
            }
        }
    }
}

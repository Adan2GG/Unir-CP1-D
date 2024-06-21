pipeline {   
     environment {
        GIT_REPOR_URL = 'https://github.com/Adan2GG/unir-CP1-D.git' // Reemplaza con la URL de tu repositorio
        STACK_NAME = 'todo-list-aws-production'     // Cambia esto al nombre de tu stack SAM
    }

     stages {
        stage('Get Code') {
            steps {
               git branch: 'master', url:'https://github.com/Adan2GG/unir-CP1-D.git'
               stash name:'code' , includes:'**'
            }
        }
        stage ('Deploy'){
            steps{
                sh 'sam build'
                sh 'sam deploy --no-fail-on-empty-changeset || true --config-file samconfig.toml --config-env production'
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
                                          python pytest -m category1 --junitxml=result-rest.xml test/integration/todoApiTest.py
                                    '''
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

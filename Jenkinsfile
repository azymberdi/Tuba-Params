def k8slabel = "jenkins-pipeline-${UUID.randomUUID().toString()}"
def slavePodTemplate = """
      metadata:
        labels:
          k8s-label: ${k8slabel}
        annotations:
          jenkinsjoblabel: ${env.JOB_NAME}-${env.BUILD_NUMBER}
      spec:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                - key: component
                  operator: In
                  values:
                  - jenkins-jenkins-master
              topologyKey: "kubernetes.io/hostname"
        containers:
        - name: buildtools
          image: fuchicorp/buildtools
          imagePullPolicy: IfNotPresent
          command:
          - cat
          tty: true
          volumeMounts:
            - mountPath: /var/run/docker.sock
              name: docker-sock
        - name: docker
          image: docker:latest
          imagePullPolicy: IfNotPresent
          command:
          - cat
          tty: true
          volumeMounts:
            - mountPath: /var/run/docker.sock
              name: docker-sock
        serviceAccountName: default
        securityContext:
          runAsUser: 0
          fsGroup: 0
        volumes:
          - name: docker-sock
            hostPath:
              path: /var/run/docker.sock
    """
    properties([
        parameters([
            booleanParam(defaultValue: false, description: 'Please select to apply the changes ', name: 'terraformApply'),
            booleanParam(defaultValue: false, description: 'Please select to destroy all ', name: 'terraformDestroy'), 
            choice(choices: ['us-west-2', 'us-west-1', 'us-east-2', 'us-east-1', 'eu-west-1'], description: 'Please select the region', name: 'aws_region'),
            booleanParam(defaultValue: false, description: 'yaml prints out if selected', name: 'yaml'),
            extendedChoice(description: 'Select a log level ', multiSelectDelimiter: ',', name: 'logs', quoteValue: false, saveJSONParameterToFile: false, type: 'PT_MULTI_SELECT', value: 'TRACE, DEBUG, INFO, WARN, ERROR', visibleItemCount: 5),
            choice(choices: ['dev', 'qa', 'stage', 'prod'], description: 'Please select the environment to deploy', name: 'environment'),
            string(defaultValue: 'None', description: 'Please provide an image ID ', name: 'ami_id', trim: false),
            string(defaultValue: 'None', description: 'Please provide a name', name: 'Name', trim: false)
        ])
    ])
    podTemplate(name: k8slabel, label: k8slabel, yaml: slavePodTemplate, showRawYaml: params.yaml) {
      node(k8slabel) { 
        stage("Pull SCM") {
            git 'https://github.com/tuyalou/jenkins-instance.git'
        }
        stage("Generate Variables") {
          dir('deployments/terraform') {
            println("Generate Variables")
            def deployment_configuration_tfvars = """
            environment = "${environment}"
            ami = "${ami_id}"
            Name = "${Name}"
            """.stripIndent()
            writeFile file: 'deployment_configuration.tfvars', text: "${deployment_configuration_tfvars}"
            sh 'cat deployment_configuration.tfvars >> dev.tfvars'
          }   
        }
        container("buildtools") {
            dir('deployments/terraform') {
                withCredentials([usernamePassword(credentialsId: "aws-access-${environment}", 
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    println("Selected cred is: aws-access-${environment}")
                    stage("Terraform Apply/plan") {
                        if (!params.terraformDestroy) {
                            if (params.terraformApply) {
                                println("Applying the changes")
                                sh """
                                #!/bin/bash
                                export AWS_DEFAULT_REGION=${aws_region}
                                source ./setenv.sh dev.tfvars
                                TF_LOG=${logs} terraform apply -auto-approve -var-file \$DATAFILE
                                """
                            } else {
                                println("Planing the changes")
                                sh """
                                #!/bin/bash
                                set +ex
                                ls -l
                                export AWS_DEFAULT_REGION=${aws_region}
                                source ./setenv.sh dev.tfvars
                                TF_LOG=${logs} terraform plan -var-file \$DATAFILE
                                """
                            }
                        }
                    }
                    stage("Terraform Destroy") {
                        if (params.terraformDestroy) {
                            println("Destroying the all")
                            sh """
                            #!/bin/bash
                            export AWS_DEFAULT_REGION=${aws_region}
                            source ./setenv.sh dev.tfvars
                            terraform destroy -auto-approve -var-file \$DATAFILE
                            """
                        } else {
                            println("Skiping the destroy")
                        }
                    }
                }
            }
        }
        stage("Jenkins Task 1") {
            println("Tuba Loughlin")
        }
      }
    } 
    podTemplate(name: k8slabel, label: k8slabel, yaml: slavePodTemplate) {
      node(k8slabel) {
        timestamps {
          dir('deployments/terraform/mysql_db_module') {
            container('buildtools') {
              stage("Pulling the code") {
                checkout scm
                gitCommitHash = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
              }
              stage("Generate Variables") {
                println("Generate Variables")
                def deployment_configuration_tfvars = """
                region = "${region}"
                username = "${username}"
                password = "${password}"
                mysql_name = "${mysql_name}"
                """.stripIndent()
                writeFile file: 'deployment_configuration.tfvars', text: "${deployment_configuration_tfvars}"
                sh 'cat deployment_configuration.tfvars >> data-miner.tfvars'  
              }
              stage("Terraform Apply/plan") {
                println("Selected cred is: aws-access-${environment}")
                    withCredentials([usernamePassword(credentialsId: "aws-access-${environment}", passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                      if (!params.terraformDestroy) {
                        if (params.terraformApply) {
                          println("Applying the changes")
                          sh """
                          #!/bin/bash
                          export AWS_DEFAULT_REGION=${region}
                          terraform init
                          terraform apply -auto-approve -var-file data-miner.tfvars
                          """
                        } else {
                          println("Planing the changes")
                          sh """
                          #!/bin/bash
                          set +ex
                          ls -l
                          export AWS_DEFAULT_REGION=${region}
                          terraform inits
                          terraform plan -var-file data-miner.tfvars
                          """
                         }
                      }
                    }
                    stage("Terraform Destroy") {
                      if (params.terraformDestroy) {
                        println("Destroying the all")
                        sh """
                        #!/bin/bash
                        export AWS_DEFAULT_REGION=${region}
                        terraform init
                        terraform destroy -auto-approve -var-file data-miner.tfvars
                        """
                      } else {
                        println("Skiping the destroy")
                      }
                    }
              }
            }
          }
             stage("Trigger Build Job") {
               build 'data-miner-build'
             }
        }
      }
    }
            stage('Push image') {
              withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "aws-access-${environment}", usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                println("ECR Auth")
                sh """
                #!/bin/bash
                aws --region ${aws_region} ecr get-login-password | docker login \
                --password-stdin --username AWS \
                "${aws_account_id}.dkr.ecr.${aws_region}.amazonaws.com/tubes"
                docker tag ${gitCommitHash} ${aws_account_id}.dkr.ecr.${aws_region}.amazonaws.com/tubes
                docker push ${aws_account_id}.dkr.ecr.${aws_region}.amazonaws.com/tubes
                """
              }
              // Tag the image
              docker tag ${gitCommitHash} ${aws_account_id}.dkr.ecr.${aws_region}.amazonaws.com/tubes
              // Push image to the ECR repository with new release
              docker push ${aws_account_id}.dkr.ecr.${aws_region}.amazonaws.com/tubes
              }
            }
           }

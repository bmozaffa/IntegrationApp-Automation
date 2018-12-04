def version, mvnCmd = "mvn fabric8:deploy"
      pipeline {
        agent {
          label 'maven'
        }
        stages {
          stage('Build App') {
            steps {
              git branch: 'master', url: 'https://github.com/redhatHameed/3ScaleFuseAMQ.git'
              script {
		  sh "pwd && ls -la maingateway-service"
                  def pom = readMavenPom file: 'maingateway-service/pom.xml'
                  version = pom.version
              }
              sh "cd maingateway-service && ${mvnCmd} install -DskipTests=true"
            }
          }
          stage('Test') {
            steps {
              sh "${mvnCmd} test"
              step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
            }
          }
          
          stage('Create Image Builder') {

            when {
              expression {
                openshift.withCluster() {
                  openshift.withProject(env.DEV_PROJECT) {
                    return !openshift.selector("bc", "maingateway-service").exists();
                  }
                }
              }
            }
            steps {
              script {
                openshift.withCluster() {
                  openshift.withProject() {
                    openshift.newBuild("--name=maingateway-service", "--image-stream=redhat-openjdk18-openshift:1.3", "--binary=true")
                  }
                }
              }
            }
          }
          stage('Build Image') {
            steps {
              sh "rm -rf ocp && mkdir -p ocp/deployments"
              sh "pwd && ls -la target "
              sh "cp target/bookstore-*.jar ocp/deployments"

              script {
                openshift.withCluster() {
                  openshift.withProject(env.DEV_PROJECT) {
                    openshift.selector("bc", "maingateway-service").startBuild("--from-dir=./ocp","--follow", "--wait=true")
                  }
                }
              }
            }
          }
          stage('Create DEV') {
            when {
              expression {
                openshift.withCluster() {
                  openshift.withProject(env.DEV_PROJECT) {
                    return !openshift.selector('dc', 'maingateway-service').exists()
                  }
                }
              }
            }
            steps {
              script {
                openshift.withCluster() {
                  openshift.withProject(env.DEV_PROJECT) {
                    def app = openshift.newApp("maingateway-service:latest")
                    app.narrow("svc").expose();

                    //http://localhost:8080/actuator/health
                    openshift.set("probe dc/bookstore --readiness --get-url=http://:8080/actuator/health --initial-delay-seconds=30 --failure-threshold=10 --period-seconds=10")
                    openshift.set("probe dc/bookstore --liveness  --get-url=http://:8080/actuator/health --initial-delay-seconds=180 --failure-threshold=10 --period-seconds=10")

                    def dc = openshift.selector("dc", "bookstore")
                    while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                        sleep 10
                    }
                    openshift.set("triggers", "dc/bookstore", "--manual")
                  }
                }
              }
            }
          }
          stage('Deploy DEV') {
            steps {
              script {
                openshift.withCluster() {
                  openshift.withProject(env.DEV_PROJECT) {
                    openshift.selector("dc", "maingateway-service").rollout().latest();
                  }
                }
              }
            }
          }
          stage('Promote to STAGE?') {
            steps {
              script {
                openshift.withCluster() {
                  openshift.tag("${env.DEV_PROJECT}/bookstore:latest", "${env.STAGE_PROJECT}/bookstore:${version}")
                }
              }
            }
          }
          stage('Deploy STAGE') {
            steps {
              script {
                openshift.withCluster() {
                  openshift.withProject(env.STAGE_PROJECT) {
                    if (openshift.selector('dc', 'bookstore').exists()) {
                      openshift.selector('dc', 'bookstore').delete()
                      openshift.selector('svc', 'bookstore').delete()
                      openshift.selector('route', 'bookstore').delete()
                    }

                    openshift.newApp("bookstore:${version}").narrow("svc").expose()
                    openshift.set("probe dc/bookstore --readiness --get-url=http://:8080/actuator/health --initial-delay-seconds=30 --failure-threshold=10 --period-seconds=10")
                    openshift.set("probe dc/bookstore --liveness  --get-url=http://:8080/actuator/health --initial-delay-seconds=180 --failure-threshold=10 --period-seconds=10")
                  }
                }
              }
            }
          }
        }
      }

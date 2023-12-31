/*
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-slave
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-slave-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins-slave
  namespace: jenkins
*/

pipeline {

    parameters {
        choice(description: "Action", name: "Action", choices: ["Plan", "Apply", "Destroy"])
        string(description: "Admin User name", name: "ADMIN_USERNAME", defaultValue: env.ADMIN_USERNAME ? env.ADMIN_USERNAME : '')
        string(description: "Admin User password", name: "ADMIN_USERPASSWORD", defaultValue: env.ADMIN_USERPASSWORD ? env.ADMIN_USERPASSWORD : '')
        string(description: "MariaDB User name", name: "MARIADB_USERNAME", defaultValue: env.MARIADB_USERNAME ? env.MARIADB_USERNAME : '')
        string(description: "MariaDB User password", name: "MARIADB_USERPASSWORD", defaultValue: env.MARIADB_USERPASSWORD ? env.MARIADB_USERPASSWORD : '')
        string(description: "MariaDB Host", name: "MARIADB_HOST", defaultValue: env.MARIADB_HOST ? env.MARIADB_HOST : '')
        string(description: "MariaDB Port", name: "MARIADB_PORT", defaultValue: env.MARIADB_PORT ? env.MARIADB_PORT : '')
        string(description: "MariaDB Database", name: "MARIADB_DATABASE", defaultValue: env.MARIADB_DATABASE ? env.MARIADB_DATABASE : '')
        string(description: "Istio Ingress Gateway", name: "ISTIO_INGRESS_GATEWAY", defaultValue: env.ISTIO_INGRESS_GATEWAY ? env.ISTIO_INGRESS_GATEWAY : '')
        string(description: "Istio Host", name: "ISTIO_HOST", defaultValue: env.ISTIO_HOST ? env.ISTIO_HOST : '')
        booleanParam(description: "Debug", name: "DEBUG", defaultValue: env.DEBUG ? env.DEBUG : "false")
        booleanParam(description: "Restart", name: "RESTART", defaultValue: env.RESTART ? env.RESTART : "false")
    }

    agent {
        kubernetes {
            yaml """
                apiVersion: "v1"
                kind: "Pod"
                spec:
                  securityContext:
                    runAsUser: 1001
                    runAsGroup: 1001
                    fsGroup: 1001
                  containers:
                  - command:
                    - "cat"
                    image: "alpine/helm:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "helm"
                    resources: {}
                    tty: true
                    volumeMounts:
                    - mountPath: "/.config"
                      name: "config-volume"
                      readOnly: false
                    - mountPath: "/.cache/helm/"
                      name: "cache-volume"
                      readOnly: false
                  - command:
                    - "cat"
                    image: "bitnami/kubectl:latest"
                    imagePullPolicy: "IfNotPresent"
                    name: "kubectl"
                    resources: {}
                    tty: true
                  - command:
                    - "cat"
                    image: "k8s.gcr.io/kustomize/kustomize:v5.0.1"
                    imagePullPolicy: "IfNotPresent"
                    name: "kustomize"
                    resources: {}
                    tty: true
                  serviceAccountName: jenkins-slave
                  volumes:
                  - emptyDir:
                      medium: ""
                    name: "config-volume"
                  - emptyDir:
                      medium: ""
                    name: "cache-volume"
            """
        }
    }

    stages {

        stage ("Helm Repo") {

            steps {

                container ("helm") {

                    script {
    
                        // Install repo
                        sh "helm repo add bitnami https://charts.bitnami.com/bitnami"
                        sh "helm repo update"
    
                    }

                }

            }

        }

        stage ("Namespace: Apply") {

            when {

                expression {
                    return env.ACTION.equals("Apply")
                }

            }

            steps {

                container ("kubectl") {

                    script {

                        // Apply
                        sh "kubectl apply -f namespace.yaml"

                    }

                }

            }

        }

        stage ("Template: Plan") {

            steps {

                container ("helm") {

                    script {

                        // Cluster agent options
                        KEYCLOAK_OPTIONS = " "

                        // If DEBUG is enabled
                        if (env.DEBUG.equals("true")) {

                            // Enable debug
                            KEYCLOAK_OPTIONS += "--debug "

                        }

                        // If Admin username and password is provided
                        if (env.ADMIN_USERNAME && env.ADMIN_USERPASSWORD) {

                            KEYCLOAK_OPTIONS += "--set auth.adminUser=${env.ADMIN_USERNAME},auth.adminPassword=${env.ADMIN_USERPASSWORD} "

                        }

                        // Provide user name and password
                        KEYCLOAK_OPTIONS += "--set externalDatabase.host=${env.MARIADB_HOST} "
                        KEYCLOAK_OPTIONS += "--set externalDatabase.port=${env.MARIADB_PORT} "
                        KEYCLOAK_OPTIONS += "--set externalDatabase.user=${env.MARIADB_USERNAME} "
                        KEYCLOAK_OPTIONS += "--set externalDatabase.password=${env.MARIADB_USERPASSWORD} "
                        KEYCLOAK_OPTIONS += "--set externalDatabase.database=${env.MARIADB_DATABASE} "
                        KEYCLOAK_OPTIONS += "--set extraEnvVars[2].name=\"KC_DB\" "
                        KEYCLOAK_OPTIONS += "--set extraEnvVars[2].value=\"mariadb\" "
                        KEYCLOAK_OPTIONS += "--set extraEnvVars[3].name=\"KC_DB_URL\" "
                        KEYCLOAK_OPTIONS += "--set extraEnvVars[3].value=\"jdbc:mariadb://${env.MARIADB_HOST}/${env.MARIADB_DATABASE}\" "

                        // Template
                        sh "helm template primary bitnami/keycloak -f keycloak-values.yaml ${KEYCLOAK_OPTIONS.trim()} --namespace keycloak > keycloak-base.yaml"

                    }

                }

                container("kustomize") {

                    script {

                        // Download envsubst
                        httpRequest url: "https://github.com/a8m/envsubst/releases/download/v1.2.0/envsubst-Linux-x86_64", outputFile: "envsubst"

                        // Add execution permission to envsubst
                        sh "chmod +x envsubst"
                        sh "ls -l -a"
    
                        sh '''
                        for file in ./custom-resource/*; do
                            ./envsubst < "${file}" > out.txt && mv out.txt "${file}";
                        done
                        '''

                        // Kustomize
                        sh "kustomize build > keycloak-template.yaml"

                        // Print Yaml
                        sh "cat keycloak-template.yaml"

                    }

                }

            }

        }

        stage ("Template: Apply") {

            when {
                
                expression {
                    return !env.ACTION.equals("Plan")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        // Apply
                        if (env.ACTION.equals("Apply")) {

                            // Apply
                            sh "kubectl apply -f keycloak-template.yaml"

                        // Destroy
                        } else if (env.ACTION.equals("Destroy")) {

                            try {
                                
                                // Destroy
                                sh "kubectl delete -f keycloak-template.yaml"

                            } catch (Exception e) {

                                // Do nothing

                            }

                        }

                        sh "rm keycloak-template.yaml"

                    }

                }

            }

        }

        stage ("Restart") {

            when {
                
                expression {
                    return !env.ACTION.equals("Plan") && env.RESTART.equals("true")
                }
                
            }

            steps {

                container ("kubectl") {

                    script {

                        sh "kubectl rollout restart deployment -n keycloak"
                        sh "kubectl rollout restart daemonset -n keycloak"
                        sh "kubectl rollout restart statefulset -n keycloak"

                    }

                }

            }

        }

    }
    
}

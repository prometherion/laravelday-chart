import groovy.json.JsonSlurperClassic 

def label = "laravelday-${UUID.randomUUID().toString()}"

String action
switch(X_GitHub_Event) {
    case "delete":
        action = "delete"
        break
    case "push":
        action = "push"
        break
    case "create":
        action = "create"
        break
    default:
        currentBuild.result = 'ABORTED'
        return
}
echo "`action` is: ${action}"

def payload = new groovy.json.JsonSlurperClassic().parseText(json)
String ref = payload.ref.replace("refs/heads/", "")
String refType = payload.ref_type ?: "commit"
echo "`ref` is: ${ref}"
echo "`refType` is: ${refType}"

String namespace
if (["branch", "commit"].contains(refType)) {
    parts = ref.split("/")
    switch(parts[0]) {
        case "staging":
            break
        default:
            currentBuild.result = 'ABORTED'
            return
    }
    namespace = parts[0] + "-" + parts[1]
}
echo "`namespace` is: ${namespace}"

// Merge Request has been merged or closed, so deleting Chart
if (action == "delete" && refType == "branch") {
    podTemplate(label: label, containers: [
      containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:latest', command: 'cat', ttyEnabled: true)
    ], serviceAccount: 'jenkins') {
      node(label) {
        stage('Run helm') {
          container('helm') {
            sh "helm init"
            sh "helm delete cache-${namespace} --purge"
            sh "helm delete database-${namespace} --purge"
            sh "helm delete laravelday-${namespace} --purge"
          }
        }
      }
    }
    return
}

// Continuous Integration alongside with tests
podTemplate(
    label: label,
    containers: [
        containerTemplate(name: 'composer', image: 'herloct/composer:1.4.2-php7.1', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'docker', image: 'docker:1.11', ttyEnabled: true, command: 'cat')
    ],
    volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')]
  ) {
    node(label) {
        checkout(
            changelog: false,
            poll: false,
            scm: [
                $class: 'GitSCM',
                branches: [[name: "*/${ref}"]],
                doGenerateSubmoduleConfigurations: false,
                extensions: [
                    [
                        $class: 'CloneOption',
                        depth: 0,
                        noTags: false,
                        reference: '',
                        shallow: false
                    ]
                ],
                userRemoteConfigs: [
                    [
                        credentialsId: 'GitHub Robot',
                        url: 'https://github.com/prometherion/laravelday-2018.git'
                    ]
                ]
        ])
        container('composer') {
            stage("Installing dev dependencies") {
                sh "composer install"
            }
            stage("Generating key") {
                sh "mv .env.example .env && php artisan key:generate"
            }
            stage("Running test suite") {
                sh "./vendor/bin/phpunit"
            }
        }
        container('docker') {
            sh "docker build -t prometherion/laravelday-2018:${namespace} ."
        }
    }
}

// Finally deploying Chart
if (["create", "push"].contains(action)) {
    podTemplate(label: label, containers: [
      containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:latest', command: 'cat', ttyEnabled: true)
    ], serviceAccount: 'jenkins') {
        node(label) {
            checkout(
                changelog: false,
                poll: false,
                scm: [
                    $class: 'GitSCM',
                    branches: [[name: "*/master"]],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [
                        [
                            $class: 'CloneOption',
                            depth: 0,
                            noTags: false,
                            reference: '',
                            shallow: false
                        ]
                    ],
                    userRemoteConfigs: [
                        [
                            credentialsId: 'GitHub Robot',
                            url: 'https://github.com/prometherion/laravelday-chart.git'
                        ]
                    ]
            ])
            stage('Installing chart') {
                container('helm') {
                    sh "helm init"
                    sh "helm upgrade --install cache-${namespace} stable/redis --namespace ${namespace}"
                    sh "helm upgrade --install database-${namespace} stable/mysql --namespace ${namespace}"
                    sh "helm upgrade --install laravelday-${namespace} . --set image=prometherion/laravelday-2018:${namespace} --namespace ${namespace}"
                }
            }
        }
    }
}
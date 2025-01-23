@Library('conduit@latest') _

String shortApplicationName = 'cms-file-processor'
String applicationName = "gov-solutions-dmom-$shortApplicationName"
env.SHORT_APPLICATION_NAME = shortApplicationName
env.APPLICATION_NAME = applicationName

String registry = 'registry.cigna.com'
String registryDev = 'registry.cigna.com'
String registryOrg = 'transactionhub'

//TODO: update application version before each release
env.APPLICATION_VERSION = '0.0.6'
def branchPattern = 'feature.*|stage|develop|master|pvs|mo.*'

// Creating place holders for vars within SWITCH statement
def kanikoRegistry = ""
def kanikoCred = ""
def helmCred = ""

// This is for registry login info when pushing to Quay
switch(env.BRANCH_NAME) {
    case ["develop", "pvs", "stage"]:
        kanikoRegistry = "registry-dev.cigna.com"
        kanikoCred = 'helm-transaction-registry-token-dev'
        helmCred = "openshift-credentials-non-prod"
        break
    case ~/feature.*/:
        kanikoRegistry = "registry-dev.cigna.com"
        kanikoCred = 'helm-transaction-registry-token-dev'
        helmCred = 'openshift-credentials-non-prod'
        break
    case ~/release.*/:
        kanikoRegistry = "registry-dev.cigna.com"
        kanikoCred = 'helm-transaction-registry-token-dev'
        helmCred = "openshift-credentials-non-prod"
        break
    default:
        kanikoRegistry = "registry.cigna.com"
        kanikoCred = 'registry-token-prod'
        helmCred = "openshift-credentials-prod"
        break
}


cignaBuildFlow {
    githubConnectionName = 'cigna-github'
    cloudName = 'us-med-gov-intsvc-dev-openshift-devops1'
    phases = [
        [
            buildType           : 'maven',
            veracodeEnabled     : false,
            branchPattern       : 'feature.*|release.*|develop|pvs|master',
            containers          : [
                [
                    name           : 'maven',
                    image          : 'registry.cigna.com/docker.io/maven',
                    version        : '3.6-jdk-11',
                    imagePullPolicy: 'Always',
                ],
            ],
            maven               : [
                hasSnapshotVersion: true,
                authSettings      : true,
                alterVersions     : true,
                debug             : false,
            ],
            junit               : [
                testResults: 'target/surefire-reports/*.xml'
            ],
            sonarQube           : [
                credentialsId     : 'sonarqube-service-id',
                mainBranch        : 'develop',
                scannerOptions    : '-Xmx1536m',
                containerMaxMemory: '2Gi',
                scannerProperties : [
                    'sonar.projectKey=$APPLICATION_NAME',
                    'sonar.sources=src/main/java',
                    'sonar.tests=src/test/java',
                    'sonar.java.binaries=target/',
                    'sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml',
                    'sonar.junit.reportPaths=target/surefire-reports/*.xml'
                ],
            ],
            artifactory         : [
                credentialsId: 'artifactory_service_id'
            ],
            checkmarx           : [
                credentialsId: 'checkmarx-service-creds',
                settings     : [
                    CX_PROJECT_TEAM_NAME: 'transactionhub',
                    CX_SCAN_TYPE        : 'AsyncScan',
                ]
            ],
        ],
        [
            packagingType     : 'kaniko',
            branchPattern     : branchPattern,
            dockerRegistry    : "${kanikoRegistry}",
            dockerfile        : 'Dockerfile',
            conftestValidation: false,
            image             : [
                org : 'transactionhub',
                name: applicationName,
                tags: [
                    [
                        tag   : '$APPLICATION_VERSION-$GIT_COMMIT_SHORT',
                        expire: true
                    ],
                    [
                        tag   : '$APPLICATION_VERSION',
                        expire: true
                    ],
                    [
                        tag   : '$GIT_COMMIT_SHORT',
                        expire: true
                    ]
                ]
            ],
            quay              : [
                credentialsId: "${kanikoCred}"
            ]
        ],

        [
            deploymentType        : 'helm',
            branchPattern         : 'feature.*',
            sdlcEnvironment       : 'dev',
            isProductionDeployment: false,
            vault                 : [
                credentialsId: 'cms-processor-vault-nonprod',
                files        : [
                    "$applicationName-pkg/values/dev/dmom-cms-secrets.yaml.enc",
                ]
            ],
            helm                  : [
                serverUrl     : 'https://api.gp-1-nonprod.openshift.cignacloud.com:6443',
                namespace     : 'transaction-hub-dev',
                credentialsId : "$helmCred",
                oscpLDAPAuth  : true,

                chart         : "$applicationName-pkg",
                deploymentName: applicationName,
                setValues     : [
                    'applicationName': applicationName,
                    'imageRepository': "$kanikoRegistry/$registryOrg/$applicationName",
                    'imageTag'       : '$APPLICATION_VERSION-$GIT_COMMIT_SHORT'
                ],
                values        : [
                    "$applicationName-pkg/values/values.yaml",
                     "$applicationName-pkg/values/dev/dmom-cms-secrets.yaml",
                    "$applicationName-pkg/values/dev/cmsFileConfig.yaml",
                    "$applicationName-pkg/values/dev/common.yaml",
                ],
                historyMax    : 1
             ]
        ],

        [
                    deploymentType        : 'helm',
                    branchPattern         : 'develop',
                    sdlcEnvironment       : 'sys',
                    isProductionDeployment: false,
                    vault                 : [
                        credentialsId: 'cms-processor-vault-nonprod',
                        files        : [
                            "$applicationName-pkg/values/sys/dmom-cms-secrets.yaml.enc",
                        ]
                    ],
                    helm                  : [
                        serverUrl     : 'https://api.gp-1-nonprod.openshift.cignacloud.com:6443',
                        namespace     : 'transaction-hub-sys',
                        credentialsId : "$helmCred",
                        oscpLDAPAuth  : true,

                        chart         : "$applicationName-pkg",
                        deploymentName: applicationName,
                        setValues     : [
                            'applicationName': applicationName,
                            'imageRepository': "$kanikoRegistry/$registryOrg/$applicationName",
                            'imageTag'       : '$APPLICATION_VERSION-$GIT_COMMIT_SHORT'
                        ],
                        values        : [
                            "$applicationName-pkg/values/values.yaml",
                             "$applicationName-pkg/values/sys/dmom-cms-secrets.yaml",
                            "$applicationName-pkg/values/sys/cmsFileConfig.yaml",
                            "$applicationName-pkg/values/sys/common.yaml",
                        ],
                        historyMax    : 1
                     ]
        ],

                [
                            deploymentType        : 'helm',
                            branchPattern         : 'pvs',
                            sdlcEnvironment       : 'pvs',
                            isProductionDeployment: false,
                            vault                 : [
                                credentialsId: 'cms-processor-vault-nonprod',
                                files        : [
                                    "$applicationName-pkg/values/pvs/dmom-cms-secrets.yaml.enc",
                                ]
                            ],
                            helm                  : [
                                serverUrl     : 'https://api.gp-1-pvs.openshift.cignacloud.com:6443',
                                namespace     : 'transaction-hub-pvs',

                                credentialsId : "$helmCred",
                                oscpLDAPAuth  : true,

                                chart         : "$applicationName-pkg",
                                deploymentName: applicationName,
                                setValues     : [
                                    'applicationName': applicationName,
                                    'imageRepository': "$kanikoRegistry/$registryOrg/$applicationName",
                                    'imageTag'       : '$APPLICATION_VERSION-$GIT_COMMIT_SHORT'
                                ],
                                values        : [
                                    "$applicationName-pkg/values/values.yaml",
                                     "$applicationName-pkg/values/pvs/dmom-cms-secrets.yaml",
                                    "$applicationName-pkg/values/pvs/cmsFileConfig.yaml",
                                    "$applicationName-pkg/values/pvs/common.yaml",
                                ],
                                historyMax    : 1
                             ]
                ],
                [
                            deploymentType        : 'helm',
                            branchPattern         : 'release.*',
                            sdlcEnvironment       : 'rlse',
                            isProductionDeployment: false,
                            vault                 : [
                                credentialsId: 'cms-processor-vault-nonprod',
                                files        : [
                                    "$applicationName-pkg/values/rlse/dmom-cms-secrets.yaml.enc",
                                ]
                            ],
                            helm                  : [
                                serverUrl     : 'https://api.gp-1-rlse.openshift.cignacloud.com:6443',
                                namespace     : 'transaction-hub-rlse',

                                credentialsId : "$helmCred",
                                oscpLDAPAuth  : true,

                                chart         : "$applicationName-pkg",
                                deploymentName: applicationName,
                                setValues     : [
                                    'applicationName': applicationName,
                                    'imageRepository': "$kanikoRegistry/$registryOrg/$applicationName",
                                    'imageTag'       : '$APPLICATION_VERSION-$GIT_COMMIT_SHORT'
                                ],
                                values        : [
                                    "$applicationName-pkg/values/values.yaml",
                                    "$applicationName-pkg/values/rlse/dmom-cms-secrets.yaml",
                                    "$applicationName-pkg/values/rlse/cmsFileConfig.yaml",
                                    "$applicationName-pkg/values/rlse/common.yaml",
                                ],
                                historyMax    : 1
                             ]
                ],

                [
                            deploymentType        : 'helm',
                            branchPattern         : 'master',
                            sdlcEnvironment       : 'prod',
                            isProductionDeployment: false,
                            vault                 : [
                                credentialsId: 'cms-processor-vault-prod',
                                files        : [
                                    "$applicationName-pkg/values/prod/dmom-cms-secrets.yaml.enc",
                                ]
                            ],
                            helm                  : [
                                serverUrl     : 'https://api.gp-1-prod.openshift.cignacloud.com:6443',
                                namespace     : 'transaction-hub-prod',

                                credentialsId : "$helmCred",
                                oscpLDAPAuth  : true,

                                chart         : "$applicationName-pkg",
                                deploymentName: applicationName,
                                setValues     : [
                                    'applicationName': applicationName,
                                    'imageRepository': "$kanikoRegistry/$registryOrg/$applicationName",
                                    'imageTag'       : '$APPLICATION_VERSION-$GIT_COMMIT_SHORT'
                                ],
                                values        : [
                                    "$applicationName-pkg/values/values.yaml",
                                    "$applicationName-pkg/values/prod/dmom-cms-secrets.yaml",
                                    "$applicationName-pkg/values/prod/cmsFileConfig.yaml",
                                    "$applicationName-pkg/values/prod/common.yaml",
                                ],
                                historyMax    : 1
                             ]
                ]
    ]
}

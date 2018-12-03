@Library("islog-helper") _

pipeline {
    agent none

    options {
        gitLabConnection('Gitlab Pontos')
        disableConcurrentBuilds()
    }

    environment {
        LINUX_DOCKER_IMAGE_NAME = 'docker-registry.islog.com:5000/conan-recipes-support:cis-latest'
        ANDROID_DOCKER_IMAGE_NAME = 'docker-registry.islog.com:5000/conan-recipes-support-android:cis-latest'
        PACKAGE_NAME = "LogicalAccess/2.1.0@islog/${BRANCH_NAME}"

        // This is needed because MSBuild fails spuriously quiet often
        // on the build machine.
        MSBUILDDISABLENODEREUSE = 1
    }

    parameters {
        booleanParam(name: 'BUILD_COMMON_VARIANTS',
                defaultValue: true,
                description: 'Build common variants.')
        booleanParam(name: 'BUILD_ALL_VARIANTS',
                defaultValue: false,
                description: 'Build all variants of options.')
        booleanParam(name: 'BUILD_DEBUG',
                defaultValue: true,
                description: 'Perform DEBUG build')
        booleanParam(name: 'BUILD_RELEASE',
                defaultValue: true,
                description: 'Perform RELEASE build')
        booleanParam(name: 'BUILD_WINDOWS',
                defaultValue: true,
                description: 'Perform Windows build')
        booleanParam(name: 'BUILD_LINUX',
                defaultValue: true,
                description: 'Perform Linux build')
        booleanParam(name: 'BUILD_ANDROID',
                defaultValue: true,
                description: 'Perform Android build')
    }

    /**
     * This Jenkinsfile driven build will generate Conan package for multiple
     * combination of settings and options.
     */

    stages {
        stage('Android ARM Debug') {
            // Build with no additional option, simply using a Conan profile file.
            // Also for now we assume any profile is android, which wont always be true
            when { expression { params.BUILD_ANDROID } }
            steps {
                script {
                    node('linux') {
                        docker.image(ANDROID_DOCKER_IMAGE_NAME).inside {
                            checkout scm
                            lla.conanPerformAllWithProfile('../profiles/arm_clang_android_21_debug')
                        }
                    }
                }
            }
        }
        stage('Android ARM') {
            // Build with no additional option, simply using a Conan profile file.
            // Also for now we assume any profile is android, which wont always be true
            when { expression { params.BUILD_ANDROID } }
            steps {
                script {
                    node('linux') {
                        docker.image(ANDROID_DOCKER_IMAGE_NAME).inside {
                            checkout scm
                            lla.conanPerformAllWithProfile('../profiles/arm_clang_android_21')
                        }
                    }
                }
            }
        }
        stage('Android x86') {
            // Build with no additional option, simply using a Conan profile file.
            // Also for now we assume any profile is android, which wont always be true
            when { expression { params.BUILD_ANDROID } }
            steps {
                script {
                    node('linux') {
                        docker.image(ANDROID_DOCKER_IMAGE_NAME).inside {
                            checkout scm
                            lla.conanPerformAllWithProfile('../profiles/x86_clang_android_21')
                        }
                    }
                }
            }
        }

        stage('Minimal Feature Build') {
            when { expression { params.BUILD_COMMON_VARIANTS } }
            steps {
                buildCommonVariants(true)
            }
        }

        stage('Complete Feature Build') {
            when { expression { params.BUILD_COMMON_VARIANTS } }
            steps {
                buildCommonVariants(false)
            }
        }

        stage('All Variants Build') {
            when { expression { params.BUILD_ALL_VARIANTS } }
            steps {
                testDynamic()
            }
        }
    }

    post {
        changed {
            script {
                if (currentBuild.currentResult == 'FAILURE' || currentBuild.currentResult == 'SUCCESS') {
                    // Other values: SUCCESS, UNSTABLE
                    // Send an email only if the build status has changed from green/unstable to red
                    emailext subject: '$DEFAULT_SUBJECT',
                            body: '$DEFAULT_CONTENT',
                            recipientProviders: [
                                    [$class: 'CulpritsRecipientProvider'],
                                    [$class: 'DevelopersRecipientProvider'],
                                    [$class: 'RequesterRecipientProvider']
                            ],
                            replyTo: 'cis@islog.com',
                            to: 'reports@islog.com'
                }
            }
        }
    }
}

/**
 * Generate stages to build common variants of LogicalAccess.
 *
 * If minimalFeatures is false, build everything we can with FALSE, otherwise
 * build with TRUE.
 */
def buildCommonVariants(minimalFeatures) {
    def boolBuildOptions = ['LLA_BUILD_UNITTEST']
    def hardcodedOptions = [:]

    if (minimalFeatures) {
        hardcodedOptions = ['LLA_BUILD_PKCS'   : 'False',
                            'LLA_BUILD_RFIDEAS': 'False',
                            'LLA_BUILD_IKS'    : 'False']

    } else {
        hardcodedOptions = ['LLA_BUILD_PKCS'   : 'True',
                            'LLA_BUILD_RFIDEAS': 'True',
                            'LLA_BUILD_IKS'    : 'True']
    }

    def combinations = utils.generateOptionsCombinations(boolBuildOptions, hardcodedOptions)
    lla.startJobForCombinations(combinations)
}

def testDynamic() {
    def boolBuildOptions = ['LLA_BUILD_IKS',
                            'LLA_BUILD_UNITTEST',
                            'LLA_BUILD_PKCS',
                            'LLA_BUILD_RFIDEAS']
    def combinations = utils.generateOptionsCombinations(boolBuildOptions, [:])
    lla.startJobForCombinations(combinations)
}

def debPackageBuild() {
    sh './scripts/debian/git-buildpackage.sh'
    sh 'cd /home/jenkins/liblogicalaccess-debian && dupload --nomail ../*.changes'
}

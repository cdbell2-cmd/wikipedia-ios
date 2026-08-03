#!/usr/bin/env groovy
// iOS MultiBranch Pipeline
// Modeled after Virgo_iOS (Adobe DCM) on CloudBees CI — dcmmercury controller
// Open source reference: https://github.com/wikimedia/wikipedia-ios

// Branch type helpers — mirrors Virgo_iOS branch naming conventions
def isReleaseBranch  = env.BRANCH_NAME ==~ /release\/.*/
def isDevelopBranch  = env.BRANCH_NAME == 'develop'
def isMainBranch     = env.BRANCH_NAME == 'master'
def isHotfixBranch   = env.BRANCH_NAME ==~ /hotfix\/.*/
def isBetaBranch     = env.BRANCH_NAME ==~ /beta-.*/
def isFeatureBranch  = env.BRANCH_NAME ==~ /feature\/.*/

pipeline {
    agent {
        // Targets the macOS build agents (dcm-csbm-mac-175..184 on dcmmercury)
        label 'mac'
    }

    options {
        // develop: 7 days / 10 builds  |  all other branches: 7 days / 5 builds
        // Mirrors LogRotator from branch config.xml; artifacts kept indefinitely
        buildDiscarder(logRotator(
            daysToKeepStr:         '7',
            numToKeepStr:          isDevelopBranch ? '10' : '5',
            artifactDaysToKeepStr: '-1',
            artifactNumToKeepStr:  '-1'
        ))
        timeout(time: 2, unit: 'HOURS')
    }

    // These variables are injected by the MultiBranch folder
    // (EnvVarsFolderProperty in config.xml — exact names from bundle)
    // KEYCHAIN_PASSWORD, HockeyAppToken, dcmbuildAPIToken
    // are already in scope as environment variables; no credentials() binding needed.

    environment {
        XCODE_WORKSPACE = 'Wikipedia.xcworkspace'
        XCODE_SCHEME    = 'Wikipedia'
        OUTPUT_DIR      = "${WORKSPACE}/build/output"
    }

    stages {

        // ── 1. CHECKOUT ──────────────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // ── 2. INSTALL DEPENDENCIES ──────────────────────────────────────────
        stage('Install Dependencies') {
            steps {
                sh '[ -f Podfile ] && pod install --repo-update || echo "No Podfile found — skipping CocoaPods install"'
            }
        }

        // ── 3. BUILD ─────────────────────────────────────────────────────────
        stage('Build') {
            steps {
                // Uses xcode-plugin (v2.0.17) installed on dcmmercury controller
                xcodeBuild(
                    cleanBeforeBuild:    true,
                    configuration:       'Debug',
                    target:              "${XCODE_SCHEME}",
                    xcodeWorkspaceFile:  "${XCODE_WORKSPACE}",
                    sdk:                 'iphonesimulator',
                    buildDir:            "${OUTPUT_DIR}",
                    developmentTeamName: '',
                    signingMethod:       'nosign'
                )
            }
        }

        // ── 4. UNIT TEST ─────────────────────────────────────────────────────
        stage('Unit Test') {
            steps {
                xcodeBuild(
                    cleanBeforeBuild:    false,
                    configuration:       'Debug',
                    target:              "${XCODE_SCHEME}",
                    xcodeWorkspaceFile:  "${XCODE_WORKSPACE}",
                    sdk:                 'iphonesimulator',
                    buildDir:            "${OUTPUT_DIR}",
                    developmentTeamName: '',
                    signingMethod:       'nosign',
                    xcodebuildArguments: "test -destination 'platform=iOS Simulator,name=iPhone 16,OS=latest' -enableCodeCoverage YES"
                )
            }
            post {
                always {
                    junit testResults: 'build/output/**/*.xml', allowEmptyResults: true
                }
            }
        }

        // ── 5. ARCHIVE ────────────────────────────────────────────────────────
        // Only for release/*, master, hotfix/*, and beta-* branches
        stage('Archive') {
            when {
                anyOf {
                    expression { isReleaseBranch }
                    expression { isMainBranch }
                    expression { isHotfixBranch }
                    expression { isBetaBranch }
                }
            }
            steps {
                // Unlock keychain using KEYCHAIN_PASSWORD injected by folder env vars
                sh "security unlock-keychain -p '${env.KEYCHAIN_PASSWORD}' ~/Library/Keychains/login.keychain-db"

                xcodeBuild(
                    cleanBeforeBuild:    true,
                    configuration:       'Release',
                    target:              "${XCODE_SCHEME}",
                    xcodeWorkspaceFile:  "${XCODE_WORKSPACE}",
                    sdk:                 'iphoneos',
                    buildDir:            "${OUTPUT_DIR}",
                    signingMethod:       'manual',
                    developmentTeamName: 'Wikimedia Foundation',
                    xcodebuildArguments: "-archivePath ${OUTPUT_DIR}/Wikipedia.xcarchive archive"
                )
            }
        }

        // ── 6. EXPORT IPA ─────────────────────────────────────────────────────
        stage('Export IPA') {
            when {
                anyOf {
                    expression { isReleaseBranch }
                    expression { isMainBranch }
                    expression { isHotfixBranch }
                    expression { isBetaBranch }
                }
            }
            steps {
                // ExportOptions.plist must be committed to the repository root
                exportIpa(
                    archiveDir:          "${OUTPUT_DIR}",
                    keychainPath:        "${OUTPUT_DIR}",
                    ipaExportMethod:     'ad-hoc'
                )
                archiveArtifacts artifacts: 'build/output/*.ipa', fingerprint: true
            }
        }

        // ── 7. DISTRIBUTE TO HOCKEYAPP ────────────────────────────────────────
        // release/* and hotfix/* branches only; uses HockeyAppToken folder env var
        stage('Distribute') {
            when {
                anyOf {
                    expression { isReleaseBranch }
                    expression { isHotfixBranch }
                }
            }
            steps {
                sh """
                    IPA=\$(find "${OUTPUT_DIR}" -name '*.ipa' | head -1)
                    curl \
                        -F "status=2" \
                        -F "notify=1" \
                        -F "notes=Branch: ${env.BRANCH_NAME} | Build: ${env.BUILD_NUMBER}" \
                        -F "notes_type=0" \
                        -F "ipa=@\${IPA}" \
                        -H "X-HockeyAppToken: ${env.HockeyAppToken}" \
                        https://rink.hockeyapp.net/api/2/apps/upload
                """
            }
        }

    } // end stages

    post {
        always {
            // Re-lock keychain regardless of build outcome
            sh 'security lock-keychain ~/Library/Keychains/login.keychain-db || true'
            deleteDir()
        }
        failure {
            echo "Build FAILED — branch: ${env.BRANCH_NAME}, build: ${env.BUILD_NUMBER}"
        }
        unstable {
            echo "Build UNSTABLE (test failures) — branch: ${env.BRANCH_NAME}"
        }
    }
}

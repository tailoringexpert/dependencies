pipeline {
    agent {
        docker {
            image 'ghcr.io/tailoringexpert/maven:3.9-eclipse-25'
            // KORREKTUR: """ erlaubt die Variablenauflösung über ${...}
            args """
                --entrypoint='' \
                -u 501:1000 \
                -v "${WORKSPACE}":/data \
                -v "${env.M2_VOLUME}":/home/maven
            """
            reuseNode true
        }
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '3', numToKeepStr: '3'))
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(
            name: 'RELEASE_BUILD',
            defaultValue: false,
            description: 'Create a release')
        booleanParam(
            name: 'DEPLOY',
            defaultValue: false,
            description: 'Execute deployment step')
        gitParameter(
            name: 'BRANCH',
            branch: '',
            defaultValue: 'develop',
            branchFilter: 'origin/(.*)',
            selectedValue: 'NONE',
            sortMode: 'NONE',
            type: 'PT_BRANCH',
            description: 'Select target branch')
    }

    environment {
        GIT_REPOSITORY = 'tailoringexpert/dependencies.git' 
        
        GIT_CREDENTIALS_ID = 'TAILORINGEXPERT_GITHUB_CREDENTIALS'
        GIT_CREDENTIALS    = credentials('TAILORINGEXPERT_GITHUB_CREDENTIALS')

        MAVEN_CREDENTIALS_USR = "${env.MAVEN_SONATYPE_CREDENTIALS_USR}"
        MAVEN_CREDENTIALS_PSW = "${env.MAVEN_SONATYPE_CREDENTIALS_PSW}"
        MAVEN_GPG_SIGNKEY_BASE64 = credentials('MAVEN_SONATYPE_GPG_SIGNKEY')
        MAVEN_GPG_PASSPHRASE  = credentials('MAVEN_SONATYPE_GPG_PASSPHRASE')
        
        GNUPGHOME = "/tmp/gpg-home"
        PATH = "/usr/local/bin:${env.PATH}"
        M2_VOLUME 
    }

    stages {
        stage('checkout') {
             steps {
                script {
                    deploymentExtension = params.DEPLOY ? " (deploy)" : ""
                    checkoutBranch = params.RELEASE_BUILD ? 'develop' : params.BRANCH
                    currentBuild.displayName = "#${env.BUILD_ID} | " + (params.RELEASE_BUILD ? "RELEASE " : "${checkoutBranch}${deploymentExtension}")
                }
                
                sh "echo checking out ${checkoutBranch}"             
                cleanWs()
                
                git branch: "${checkoutBranch}", url: env.GIT_URL, credentialsId: "${env.GIT_CREDENTIALS_ID}"
             }
        }

        stage('install') {
            steps {
                sh "mvn --settings .jenkins/settings.xml -Dmaven.repo.local=${M2_VOLUME}/repository -DskipTests clean install -X"
            }
        }

        stage('release') {
            when {
                expression { params.RELEASE_BUILD }
            }
            steps {
                sh '''
                    setup-git.sh            
				    mvn --settings .jenkins/settings.xml -Dmaven.repo.local=${M2_VOLUME}/repository -B -Dresume=false -DargLine='-DprocessAllModules --settings .jenkins/settings.xml -Dmaven.repo.local=/home/maven/.m2 --settings .jenkins/settings.xml' -DskipTestProject=true  -DgpgSignTag=false -DgpgSignCommit=false -DpostReleaseGoals=deploy gitflow:release
                '''                    
            }
        }

        stage('deploy') {   
            when {
                anyOf {
                    expression { params.RELEASE_BUILD }
                    expression { params.DEPLOY }
                }
            }
            steps {
                script {
                    if (params.DEPLOY) {
                        sh '''
                           # setup-maven-gpg.sh

                            set -e
                            
                            export MAVEN_GPG_SIGNKEY=$(echo "$MAVEN_GPG_SIGNKEY_BASE64" | base64 --decode)
                            mkdir -p "$GNUPGHOME"
                            chmod 700 "$GNUPGHOME"

                            echo "pinentry-mode loopback" > "$GNUPGHOME/gpg.conf"
                            echo "allow-loopback-pinentry" > "$GNUPGHOME/gpg-agent.conf"
                            chmod 600 "$GNUPGHOME"/*

                            if [ -z "$MAVEN_GPG_SIGNKEY" ]; then
                              echo "Fehler: MAVEN_GPG_SIGNKEY ist leer!"
                              exit 1
                            fi

                            echo "$MAVEN_GPG_SIGNKEY" | gpg --batch --import
                            MAVEN_GPG_KEY_ID=$(gpg --batch --list-secret-keys --with-colons | awk -F: '/^sec:/ {print $5}')

                            # Falls 'sec' leer ist, versuchen wir es mit dem Subkey ('ssb')
                            if [ -z "$MAVEN_GPG_KEY_ID" ]; then
                                MAVEN_GPG_KEY_ID=$(gpg --batch --list-secret-keys --with-colons | awk -F: '/^ssb:/ {print $5}')
                            fi

                            echo "Exakte Maven GPG-ID: $MAVEN_GPG_KEY_ID"

                            if [ -z "$MAVEN_GPG_KEY_ID" ]; then
                              echo "Fehler: Der Maven-Schlüssel konnte nicht im GPG-Speicher gefunden werden!"
                              # Zeige zur Fehlersuche an, was GPG überhaupt sieht:
                              gpg --list-secret-keys
                              exit 1
                            fi

						    mvn --settings .jenkins/settings.xml -Dmaven.repo.local=${M2_VOLUME}/repository -DskipTests deploy
                        '''
                    } else {
                        echo "Deployment bereits via gitflow:release erledigt."
                    }
                }
            }
        }
    }
}

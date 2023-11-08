properties([
    parameters([
        booleanParam(
            name: 'RELEASE_BUILD',
            defaultValue: false),
		booleanParam(
            name: 'DEPLOY',
            defaultValue: false),
        gitParameter(
            name: 'BRANCH',
            branch: '',
            defaultValue: 'develop',
            branchFilter: 'origin/(.*)',
            selectedValue: 'NONE',
            sortMode: 'NONE',
            type: 'PT_BRANCH')
    ])
])

pipeline {

    environment {
        GIT_CREDENTIALS_ID = 'TAILORINGEXPERT_GITHUB_CREDENTIALS'
		GIT_CREDENTIALS = credentials('TAILORINGEXPERT_GITHUB_CREDENTIALS')
		GPG_SIGNKEY = credentials('GITHUB_GPG_SIGNKEY')
        OSSRH_CREDENTIALS = credentials('OSSRH_CREDENTIALS')
        OSSRH_GPG = credentials('OSSRH_GPG')
		GIT_REPOSITORY = 'tailoringexpert/dependencies.git'	
         
        // other (external) defined env vars
        // M2_VOLUME maven      repoository volume
        // GPG_VOLUME           gpg key volume
        // GIT_COMMITTER_NAME   name of the git committer
        // GIT_COMMITTER_EMAIL  mail of the git committer

    }

    agent {
		docker {
			image 'tailoringexpert/mvn-openjdk-17'
			args '''
                -v $M2_VOLUME:/home/.m2 \
                -v $GPG_VOLUME:/home/.gnupg \
                -e GIT_CREDENTIALS=$GIT_CREDENTIALS \
                -e GIT_COMMITTER_NAME=$GIT_COMMITTER_NAME \
                -e GIT_COMMITTER_EMAIL=$GIT_COMMITTER_EMAIL \
                -e OSSRH_CREDENTIALS_USR=$OSSRH_CREDENTIALS_USR \
                -e OSSRH_CREDENTIALS_PSW=$OSSRH_CREDENTIALS_PSW \
                -e GPG_SIGNKEY=$GPG_SIGNKEY 
            '''
       }	   
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '3', numToKeepStr: '3'))
    }

    stages {
        stage('checkout') {
             steps {
                script {
                    checkoutBranch = params.RELEASE_BUILD ? 'develop' : params.BRANCH
                    currentBuild.displayName = "#${env.BUILD_ID} | " + (params.RELEASE_BUILD ?  "RELEASE " : ("${checkoutBranch}" + (params.DEPLOY ? " (deploy)" : "")))
                }
                sh "echo checking out ${checkoutBranch}"             
             
				cleanWs()
                git branch: "${checkoutBranch}", url: env.GIT_URL, credentialsId: GIT_CREDENTIALS_ID
             }
        }

        stage('install') {
			steps {
				sh "mvn --settings .jenkins/settings.xml -DskipTests install"
			}
        }

		stage('release') {
            when {
                expression { params.RELEASE_BUILD }
            }

			steps {
				// prepare git signing
				sh('git remote set-url origin https://$GIT_CREDENTIALS@github.com/$GIT_REPOSITORY')
                sh('git config user.name "$GIT_COMMITTER_NAME"')
				sh('git config user.email $GIT_COMMITTER_EMAIL')
				sh('git config commit.gpgsign true')
				sh('git config user.signingkey $GPG_SIGNKEY')
				
				sh "mvn --settings .jenkins/settings.xml -B -Dresume=false -DargLine='-DprocessAllModules --settings .jenkins/settings.xml' -DskipTestProject=true  -DgpgSignTag=true -DgpgSignCommit=true -DpostReleaseGoals=deploy gitflow:release" 

				// remove credentials
				sh('git remote set-url origin $GIT_URL')
			}
		}

		stage('deploy') {	
			when {
				anyOf {
					expression { params.RELEASE_BUILD };
					expression { params.DEPLOY }
				}
			}
					
            steps {
                script {
					if (params.DEPLOY) {
						sh "mvn --settings .jenkins/settings.xml -DskipTests deploy"
					} else {
						sh "exit 0"
					}
				}
			}
		}
    }
}

def initBuild() {
	sh '''#!/bin/bash
	pkill -f app.js -9 || true
	sudo service postgresql restart
	dropdb lisk_test || true
	createdb lisk_test
	'''
	deleteDir()
	checkout scm
}

def buildDependency() {
	try {
		sh '''#!/bin/bash
		# Install Deps
		npm install
		'''
	} catch (err) {
		currentBuild.result = 'FAILURE'
		report()
		error('Stopping build, installation failed')
	}
}

def startLisk() {
	try {
		sh '''#!/bin/bash
		cp test/config.json test/genesisBlock.json .
		export NODE_ENV=test
		JENKINS_NODE_COOKIE=dontKillMe ~/start_lisk.sh
		'''
	} catch (err) {
		currentBuild.result = 'FAILURE'
		report()
		error('Stopping build, Lisk failed')
	}
}

def report(){
	step([
		$class: 'GitHubCommitStatusSetter',
		errorHandlers: [[$class: 'ShallowAnyErrorHandler']],
		contextSource: [$class: 'ManuallyEnteredCommitContextSource', context: 'jenkins-ci/func-unit'],
		statusResultSource: [
			$class: 'ConditionalStatusResultSource',
			results: [
					[$class: 'BetterThanOrEqualBuildResult', result: 'SUCCESS', state: 'SUCCESS', message: 'This commit looks good :)'],
					[$class: 'BetterThanOrEqualBuildResult', result: 'FAILURE', state: 'FAILURE', message: 'This commit failed testing :('],
					[$class: 'AnyBuildResult', state: 'FAILURE', message: 'This build some how escaped evaluation']
			]
		]
	])
}

lock(resource: "Lisk-Core-Nodes", inversePrecedence: true) {
	stage ('Prepare Workspace') {
		parallel(
			"Build Node-01" : {
				node('node-01'){
					initBuild()
				}
			},
			"Build Node-02" : {
				node('node-02'){
					initBuild()
				}
			},
			"Build Node-03" : {
				node('node-03'){
					initBuild()
				}
			},
			"Build Node-04" : {
				node('node-04'){
					initBuild()
				}
			},
			"Initialize Master Workspace" : {
				node('master-01'){
					sh '''
					cd /var/lib/jenkins/coverage/
					rm -rf node-0*
					rm -rf *.zip
					rm -rf coverage-unit/*
					rm -rf lisk/*
					rm -f merged-lcov.info
					'''
					deleteDir()
					checkout scm
				}
			}
		)
	}

	stage ('Build Dependencies') {
		parallel(
			"Build Dependencies Node-01" : {
				node('node-01'){
					buildDependency()
				}
			},
			"Build Dependencies Node-02" : {
				node('node-02'){
					buildDependency()
				}
			},
			"Build Dependencies Node-03" : {
				node('node-03'){
					buildDependency()
				}
			},
			"Build Dependencies Node-04" : {
				node('node-04'){
					buildDependency()
				}
			}
		)
	}

	stage ('Start Lisk') {
		parallel(
			"Start Lisk Node-01" : {
				node('node-01'){
					startLisk()
				}
			},
			"Start Lisk Node-02" : {
				node('node-02'){
					startLisk()
				}
			},
			"Start Lisk Node-03" : {
				node('node-03'){
					startLisk()
				}
			},
			"Start Lisk Node-04" : {
				node('node-04'){
					startLisk()
				}
			}
		)
	}

	stage ('Parallel Tests') {
		parallel(
			"ESLint" : {
				node('node-01'){
					sh '''
					cd "$(echo $WORKSPACE | cut -f 1 -d '@')"
					npm run eslint
					'''
				}
			},
			"Functional http get Tests" : {
				node('node-01'){
					sh '''
					export TEST_TYPE='UNIT' NODE_ENV='TEST'
					cd "$(echo $WORKSPACE | cut -f 1 -d '@')"
					npm run test-functional-http-get
					'''
				}
			},  // End node-01
			"Functional ws Tests" : {
				node('node-02'){
					sh '''
					export TEST_TYPE='UNIT' NODE_ENV='TEST'
					cd "$(echo $WORKSPACE | cut -f 1 -d '@')"
					npm run test-functional-ws
					'''
				}
			},
			"Functional http post Tests" : {
				node('node-02'){
					sh '''
					export TEST_TYPE='UNIT' NODE_ENV='TEST'
					cd "$(echo $WORKSPACE | cut -f 1 -d '@')"
					npm run test-functional-http-post
					'''
				}
			}, // End Node-02
			"Unit Tests" : {
				node('node-03'){
					sh '''
					export TEST_TYPE='UNIT' NODE_ENV='TEST'
					cd "$(echo $WORKSPACE | cut -f 1 -d '@')"
					npm run test-unit
					'''
				}
			}, // End Node-03 unit tests
			"Functional Stress - Transactions" : {
				node('node-04'){
					sh '''
					export TEST=test/functional/ws/transport.transactions.stress.js TEST_TYPE='FUNC' NODE_ENV='TEST'
					cd "$(echo $WORKSPACE | cut -f 1 -d '@')"
					npm run jenkins
					'''
				}
			} // End Node-04
		) // End Parallel
	}

	stage ('Gather Coverage') {
		parallel(
			"Gather Coverage Node-01" : {
				node('node-01'){
					sh '''#!/bin/bash
					export HOST=127.0.0.1:4000
					npm run fetchCoverage
					# Submit coverage reports to Master
					scp test/.coverage-func.zip jenkins@master-01:/var/lib/jenkins/coverage/coverage-func-node-01.zip
					'''
				}
			},
			"Gather Coverage Node-02" : {
				node('node-02'){
					sh '''#!/bin/bash
					export HOST=127.0.0.1:4000
					npm run fetchCoverage
					# Submit coverage reports to Master
					scp test/.coverage-func.zip jenkins@master-01:/var/lib/jenkins/coverage/coverage-func-node-02.zip
					'''
				}
			},
			"Gather Coverage Node-03" : {
				node('node-03'){
					sh '''#!/bin/bash
					export HOST=127.0.0.1:4000
					# Gathers unit test into single lcov.info
					npm run coverageReport
					npm run fetchCoverage
					# Submit coverage reports to Master
					scp test/.coverage-unit/* jenkins@master-01:/var/lib/jenkins/coverage/coverage-unit/
					scp test/.coverage-func.zip jenkins@master-01:/var/lib/jenkins/coverage/coverage-func-node-03.zip
					'''
				}
			},
			"Gather Coverage Node-04" : {
				node('node-04'){
					sh '''#!/bin/bash
					export HOST=127.0.0.1:4000
					npm run fetchCoverage
					# Submit coverage reports to Master
					scp test/.coverage-func.zip jenkins@master-01:/var/lib/jenkins/coverage/coverage-func-node-04.zip
					'''
				}
			}
		)
	}

	stage ('Submit Coverage') {
		node('master-01'){
			sh '''
			cd /var/lib/jenkins/coverage/
			unzip coverage-func-node-01.zip -d node-01
			unzip coverage-func-node-02.zip -d node-02
			unzip coverage-func-node-03.zip -d node-03
			unzip coverage-func-node-04.zip -d node-04
			bash merge_lcov.sh . merged-lcov.info
			cp merged-lcov.info $WORKSPACE/merged-lcov.info
			cp .coveralls.yml $WORKSPACE/.coveralls.yml
			cd $WORKSPACE
			cat merged-lcov.info | coveralls -v
			'''
		}
	}

	stage ('Cleanup') {
		parallel(
			"Cleanup Node-01" : {
				node('node-01'){
					sh '''
					pkill -f app.js -9
					'''
				}
			},
			"Cleanup Node-02" : {
				node('node-02'){
					sh '''
					pkill -f app.js -9
					'''
				}
			},
			"Cleanup Node-03" : {
				node('node-03'){
					sh '''
					pkill -f app.js -9
					'''
				}
			},
			"Cleanup Node-04" : {
				node('node-04'){
					sh '''
					pkill -f app.js -9
					'''
				}
			},
			"Cleanup Master" : {
				node('master-01'){
					sh '''
					cd /var/lib/jenkins/coverage/
					rm -rf node-0*
					rm -rf *.zip
					rm -rf coverage-unit/*
					rm -f merged-lcov.info
					rm -rf lisk/*
					rm -f coverage.json
					rm -f lcov.info
					'''
				}
			}
		)
	}

	stage ('Set milestone') {
		node('master-01'){
			milestone 1
			currentBuild.result = 'SUCCESS'
			report()
		}
	}
}

pipeline {

    // This agent label is mandatory for running on Theta
    agent {
        label 'Datascience-Theta' 
    }

    /* Global environment variables can be set here
    * They are visible to the shells and accessible 
    * in the pipline as "${env.BUILD_ROOT}"
    * WARNING! Must use double quotes or else variables will not be passed
    * WARNING! If you are using dollar sign ($) inside a sh command, you must 
    * escape it as "\$". */
    environment {
        BASE_MODULE = 'cray-python/3.6.1.1'
        BUILD_ROOT = '/projects/datascience/jenkins-test/mpi4py-build'
        RELEASE_ROOT = '/projects/datascience/jenkins-test/mpi4py-release'
        QSTAT_HEADER = 'JobId:User:JobName'
    }

    /* The stages in a pipeline run sequentially; and only if there are no 
    * failures in prior stages.  We can use nonzero shell codes to indicate
    * failure to either retry or abort steps in the pipeline. */
    stages {

        // Create virtualenv based on Cray Python 3.6
        // ------------------------------------------
        stage('Virtualenv Setup') {
            steps {
                sh 'pwd'
                sh 'ls'
                sh '. ./env-setup.sh' // You must use "." instead of "source"! Jenkins doesn't support bash!
            }
        }

        // Build cython
        // ------------
        stage('Build Cython') {
            steps {
                sh '. ./build-cython.sh'
            }
        }

        
        // Build mpi4py
        // ------------
        stage('Build mpi4py') {
            steps {
                sh '. ./build-mpi4py.sh'
            }
        }


        // Submit a 1 node debug job to Cobalt to test the new mpi4py
        // -----------------------------------------------------------
        stage('Quick Test') {
            steps {
                // we will "pass" BUILD_ROOT to the testMPI4Py.sh Cobalt job thru a file
                sh "echo ${env.BUILD_ROOT} > BUILD_ROOT.path"

                // use the "script" step to embed old-fashioned Jenkins procedural scripting
                // this way, we can conveniently "capture" the Cobalt job ID from tail of stdout
                script {
                    cobalt_id = sh(returnStdout: true, script: 'qsub -A datascience -n 1 -t 10 -q debug-cache-quad ./testMPI4Py.sh | tail -n 1').trim()
                }

                // Keep checking until a ${cobalt_id}.finished file appears
                echo "Submitted Job to cobalt (ID ${cobalt_id}). Polling on completion..."
                timeout(time: 24, unit: 'HOURS') {
                   sh "while [ ! -f ${cobalt_id}.finished ]; do sleep 5; done"
                }
                echo "Job completed; checking output..."
                sh "cat ${cobalt_id}.output"

                // grep -q will return 0 (success) only if there is a match:
                sh "grep -q 'Success: aprun -n2 gave rank0 and rank1' ${cobalt_id}.output"
            }
        }

        // On success, deploy env to a permanent location
        stage('Deploy and Benchmark') {
            steps {
                sh ". ./deploy-benchmark.sh"
            }
        }
    }
    post {
        success {
            mail to: 'msalim@anl.gov',
             subject: "Success!  Pipeline: ${currentBuild.fullDisplayName} completed.",
             body: "The build was a success in ${env.BUILD_URL}"
        }
        failure {
            mail to: 'msalim@anl.gov',
             subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
             body: "Something is wrong with ${env.BUILD_URL}"
        }
        // Always clean up after yourself
        always {
            sh "rm -rf $BUILD_ROOT"
            sh 'rm -rf ./cython'
            sh 'rm -rf ./mpi4py'
            sh '''
            rm printRank.out
            rm *.output
            rm *.error
            rm *.cobaltlog
            '''
            deleteDir()
        }
    }
}

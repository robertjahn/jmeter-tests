pipeline {
    parameters {
        string(name: 'SCRIPT_NAME',         defaultValue: 'basiccheck.jmx', description: 'The script you want to execute', trim: true)
        string(name: 'SERVER_URL',          defaultValue: 'user.{APP_STAGING_DOMAIN}', description: 'Please enter the URI or the IP of your service you want to run your test against', trim: true)
        string(name: 'SERVER_PORT',         defaultValue: '80', description: 'Please enter the port of the endpoint', trim: true)
        string(name: 'CHECK_PATH',          defaultValue: '/health', description: 'This parameter is only good for scripts that use this parameter, e.g: basiccheck.jmx', trim: true)
        string(name: 'VUCount',             defaultValue: '1', description: 'Number of Virtual Users to be executed. ', trim: true)
        string(name: 'LoopCount',           defaultValue: '1', description: 'Number of iterations every virtual user executes', trim: true)
        string(name: 'ThinkTime',           defaultValue: '250', description: 'Default Thinktime between load testing steps')
        string(name: 'DT_LTN',              defaultValue: 'DTLoadTest', description: 'For scripts that have been setup to pass x-dynatrace-test this will pass the LTN Request Attribute', trim: true)
        choice(name: 'FUNC_VALIDATION',     choices: 'yes\nno', description: 'BREAK the Pipeline if there is a functional issue?' )
        string(name: 'AVG_RT_VALIDATION',   defaultValue: '0', description: 'BREAK the Pipeline if the average response time exceeds the passed value. 0 means NO VALIDATION')
        string(name: 'RETRY_ON_ERROR',      defaultValue: '0', description: 'How many times to retry on error? Especially useful for the initial health check as it will take a while until the app is up and running')
        string(name: 'RETRY_WAIT',          defaultValue: '5000', description: 'How long to wait between retries in milliseconds')
     }

    agent {
        label "jenkins-jmeter"
    }
    environment {
        ORG               = 'acm-workshop'
        APP_NAME          = 'jmeter-as-container'
        GIT_PROVIDER      = 'github.com'        
    }    

    stages {
        stage('RunTest') {
            steps
            {
                // check out all our tests and create the results directory!
                checkout scm
                sh "mkdir results"

                // now we run the test within the container!
                container('jmeter') {

                    // lets run the test and put the console output to output.txt
                    sh "echo 'execute the jmeter test and console output goes to output.txt'"
                    sh "/jmeter/bin/jmeter.sh -n -t ./scripts/$SCRIPT_NAME -e -o results -l result.tlf -JSERVER_URL='$SERVER_URL' -JDT_LTN='$DT_LTN' -JVUCount='$VUCount' -JLoopCount='$LoopCount' -JCHECK_PATH='$CHECK_PATH' -JSERVER_PORT='$SERVER_PORT' -JThinkTime='$ThinkTime' > output.txt"
                    
                    sh "cat output.txt"
                }

                // parse the result.tlf and archive the artifacts
                perfReport percentiles: '0,50,90,100', sourceDataFiles: 'result.tlf'
                archiveArtifacts artifacts:'*.*/**'                

                // now we validate
                container('jmeter') {
                    // Lets do the functional validation if FUNC_VALIDATION=='yes'
                    sh '''
                        ERROR_COUNT=$(awk '/summary =/ {print $15;}' output.txt)
                        if [ "$FUNC_VALIDATION" == "yes" ] && [ $ERROR_COUNT -gt 0 ]
                        then
                            echo "More than 1 error"
                        exit 1
                        fi
                    '''

                    // Lets do the performance validation if AVG_RT_VALIDATION > 0
                    sh '''
                        AVG_RT=$(awk '/summary =/ {print $9;}' output.txt)
                        echo "AVG_RT = $AVG_RT"
                        if [ $AVG_RT_VALIDATION -gt 0 ] && [ $AVG_RT_VALIDATION -gt $AVG_RT ]
                        then
                            echo "Response Time Threshold Violation: $AVG_RT > $$AVG_RT_VALIDATION"
                            exit 1
                        fi
                    '''
                }
            }
        }
    }
}
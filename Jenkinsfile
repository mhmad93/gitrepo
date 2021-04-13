import static Constants.*

class Constants {
    static final BASE_DIR = "mean"
    static final OUTPUT_DIR = BASE_DIR + "/output"
    static final RESULT_FILENAME = "result.json"
    static final HOST = "localhost"
    static final EXCHANGE = "meanexchange"
    static final QUEUE_NAME = "analyzer-events"
	static final PUBLISHER_IMAGE = "rabbitmq_publisher:latest"
}

@NonCPS
def parseJson(str) {
    return new groovy.json.JsonSlurperClassic().parseText(str)
}
def getEventTypeAndInfo(errorCode, stdErr) {
    if (errorCode == 124)
        return ['timeout', '']
    else if (errorCode != 0)
        return ['error', stdErr]
    else
        return ['result', '']
}
def createAnalyzerEvent(requestId, analyzerName, eventType, sourceContext, info='') {
    analyzerEvent = ['request_id': requestId,
                      'analyzer_name': analyzerName,
                      'event_type': eventType,
                      'source_context': sourceContext]

    if (info != '') {
        analyzerEvent['info'] = info
    }
    if (eventType == 'result') {
        if (fileExists(OUTPUT_DIR + "/" + RESULT_FILENAME)) {
            analyzerEvent['analyzer_result'] = parseJson(readFile(OUTPUT_DIR + "/" + RESULT_FILENAME))
        }
        else {
            analyzerEvent['event_type'] = 'error'
            analyzerEvent['info'] = 'no result file found, check that the analyzer finished correctly'
        }
    }
    return analyzerEvent
}


def req = parseJson(AnalyzeRequest)
def ctx = req.source_context
def seconds = req.containsKey("timeout") ? req.timeout : (1.0d / 0.0d)

node{
    stage('create input files') {
        checkout scm
        sh "rm -rf mean"
        sh "mkdir -p mean/code"
        sh "mkdir -p mean/input"
        sh "mkdir -p mean/output"
        sh 'echo $AnalyzeRequest > mean/input/analyze_request.json'
        analyzerStarted = createAnalyzerEvent(req.request_id, req.analyzer_name, "started", ctx)
        def jsonData = groovy.json.JsonOutput.toJson(analyzerStarted)
        writeFile(file: "analyzer_event.json", text: jsonData, encoding: "UTF-8")
        sh "docker run --network=\"host\" -v ${WORKSPACE}:/temp ${PUBLISHER_IMAGE} ${HOST} ${EXCHANGE} ${QUEUE_NAME} temp/analyzer_event.json"
        sh 'rm analyzer_event.json'
    }
    stage("checkout") {
    def (scheme, host) = ctx.host_uri.split("://", 2)
    sh "git clone ${scheme}://jenkins:N7kFQ8CwqyjNb4WP0HHI@${host}/a/${ctx.project_name}.git mean/code"
    sh """
       cd mean/code
       git fetch ${scheme}://jenkins:N7kFQ8CwqyjNb4WP0HHI@${host}/a/${ctx.project_name}.git ${ctx.ref} && git checkout FETCH_HEAD
       cd -
       """
    }
    stage("run analyzer") {
		def rc = sh(script: """
						  timeout ${seconds} docker run --rm\
                    --network=\"host\" \
									  -v \${WORKSPACE}/mean:/mean \
									  -u \$(id -u \${USER}):\$(id -g \${USER}) \
									  2> stderr.log \
									  ${req.docker_image}
						  """, returnStatus: true)
		def stderr = readFile("stderr.log")
		println("stderr: " + stderr + " END OF stderr")
		sh "rm stderr.log"
		def (eventType, info) = getEventTypeAndInfo(rc, stderr)
		analyzerEvent = createAnalyzerEvent(req.request_id, req.analyzer_name, eventType, ctx, info)
    }
    stage('publish') {
        def jsonData = groovy.json.JsonOutput.toJson(analyzerEvent)
        writeFile(file: "analyzer_event.json", text: jsonData, encoding: "UTF-8")
        sh "docker run --network=\"host\" -v ${WORKSPACE}:/temp ${PUBLISHER_IMAGE} ${HOST} ${EXCHANGE} ${QUEUE_NAME} temp/analyzer_event.json"
        sh 'rm analyzer_event.json'
    }
    stage('cleanup') {
        sh "rm -rf mean"
    }
}

@Library('ecdc-pipeline')
import ecdcpipeline.ContainerBuildNode
import ecdcpipeline.PipelineBuilder

project = "file-writer-sanitizer"

container_build_nodes = [
  'address': ContainerBuildNode.getDefaultContainerBuildNode('centos7-gcc8'),
  'thread': ContainerBuildNode.getDefaultContainerBuildNode('centos7-gcc8'),
]

// Set number of old builds to keep.
properties([
  pipelineTriggers([
      cron('5 3 * * *')
    ]),
  [
  $class: 'BuildDiscarderProperty',
  strategy: [
    $class: 'LogRotator',
    artifactDaysToKeepStr: '',
    artifactNumToKeepStr: '',
    daysToKeepStr: '',
    numToKeepStr: ''
  ]
]]);


pipeline_builder = new PipelineBuilder(this, container_build_nodes)
pipeline_builder.activateEmailFailureNotifications()

builders = pipeline_builder.createBuilders { container ->
  pipeline_builder.stage("${container.key}: Checkout") {
    dir(pipeline_builder.project) {
      checkout([  
                  $class: 'GitSCM', 
                  branches: [[name: 'refs/heads/main']], 
                  userRemoteConfigs: [[url: 'https://github.com/ess-dmsc/kafka-to-nexus.git']]
              ])
    }
    container.copyTo(pipeline_builder.project, pipeline_builder.project)
  }  // stage

  pipeline_builder.stage("${container.key}: Dependencies") {

    def conan_remote = "ess-dmsc-local"
    container.sh """
      mkdir build
      cd build
      conan remote add \
        --insert 0 \
        ${conan_remote} ${local_conan_server}
      conan install --build=outdated ../${pipeline_builder.project}/conanfile.txt
    """
  }  // stage

  pipeline_builder.stage("${container.key}: Configure") {
    container.sh """
      cd build
      . ./activate_run.sh
      cmake -DSANITIZER=${container.key} ../${pipeline_builder.project}
    """
  }  // stage

  pipeline_builder.stage("${container.key}: Build") {
    container.sh """
    cd build
    . ./activate_run.sh
    make UnitTests
    """
  }  // stage

  pipeline_builder.stage("${container.key}: Test") {
    def test_dir
    test_dir = 'bin'

    container.sh """
      cd build
      . ./activate_run.sh
      ./${test_dir}/UnitTests
    """
  }  // stage
}  // createBuilders

node {
  dir("${project}") {
    try {
    } catch (e) {
      failure_function(e, 'Checkout failed')
    }
  }

  try {
    parallel builders
  } catch (e) {
    pipeline_builder.handleFailureMessages()
    throw e
  }

  // Delete workspace when build is done
  cleanWs()
}

def failure_function(exception_obj, failureMessage) {
  def toEmails = [[$class: 'DevelopersRecipientProvider']]
  emailext body: '${DEFAULT_CONTENT}\n\"' + failureMessage + '\"\n\nCheck console output at $BUILD_URL to view the results.', recipientProviders: toEmails, subject: '${DEFAULT_SUBJECT}'
  throw exception_obj
}


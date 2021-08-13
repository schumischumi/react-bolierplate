#!/bin/groovy
import groovy.json.JsonSlurperClassic 


pipeline {
  agent any
  tools {
    nodejs 'recent node'
  }
  stages {
    stage('Prepare') {
      steps {
        script {
          sh 'npm install yarn -g'
          sh 'yarn install'
        }
      }
    }
    stage('Git') {
      steps {
        script {
          git "${gitURL}"
        }
      }
    }
    stage('Test') {
      steps {
        script {
          sh 'yarn test'
        }
      }
    }
    stage('Build') {
      steps {
        script {
          sh 'yarn build'
        }
      }
    }
    stage('Get JWT Token') {
      steps {
        script {
          withCredentials([usernamePassword(credentialsId: 'Portainer', usernameVariable: 'PORTAINER_USERNAME', passwordVariable: 'PORTAINER_PASSWORD')]) {
              def json = """
                  {"Username": "$PORTAINER_USERNAME", "Password": "$PORTAINER_PASSWORD"}
              """
              def jwtResponse = httpRequest acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', validResponseCodes: '200', httpMode: 'POST', ignoreSslErrors: true, consoleLogResponseBody: true, requestBody: json, url: "${portainerURL}/api/auth"
              def jwtObject = new groovy.json.JsonSlurper().parseText(jwtResponse.getContent())
              env.JWTTOKEN = "Bearer ${jwtObject.jwt}"
          }
        }
        echo "${env.JWTTOKEN}"
      }
    }
    stage('Build Docker Image on Portainer') {
      steps {
        script {
          // Get endpoint ID
          String existingEndpointId = ""
          if("true") {
            def endpointResponse = httpRequest httpMode: 'GET', ignoreSslErrors: true, url: "${portainerURL}/api/endpoints", validResponseCodes: '200', consoleLogResponseBody: true, customHeaders:[[name:"Authorization", value: env.JWTTOKEN ], [name: "cache-control", value: "no-cache"]]
            def endpoint = new groovy.json.JsonSlurper().parseText(endpointResponse.getContent())
            //existingEndpointId = endpoint.Id
            env.ENDPOINTID = "${endpoint.Id.join(',')}"
        
            
          }
          echo "${env.ENDPOINTID}"
          // Build the image
              def repoURL = """
                ${portainerURL}/api/endpoints/${env.ENDPOINTID}/docker/build?t=${dockerTag}&remote=${gitURL}&dockerfile=Dockerfile&nocache=true
              """
              def imageResponse = httpRequest httpMode: 'POST', ignoreSslErrors: true, url: repoURL, validResponseCodes: '200', customHeaders:[[name:"Authorization", value: env.JWTTOKEN ], [name: "cache-control", value: "no-cache"]]
              echo "${imageResponse}"
        }
      }
    }
    stage('Delete old Stack') {
      steps {
        script {

          // Get all stacks
          String existingStackId = ""
          if("true") {
            def stackResponse = httpRequest httpMode: 'GET', ignoreSslErrors: true, url: "${portainerURL}/api/stacks", validResponseCodes: '200', consoleLogResponseBody: true, customHeaders:[[name:"Authorization", value: env.JWTTOKEN ], [name: "cache-control", value: "no-cache"]]
            def stacks = new groovy.json.JsonSlurper().parseText(stackResponse.getContent())
            
            stacks.each { stack ->
              if(stack.Name == "${SwarmName}") {
                existingStackId = stack.Id
              }
            }
          }

          if(existingStackId?.trim()) {
            // Delete the stack
            def stackURL = """
              ${portainerURL}/api/stacks/$existingStackId?endpointId=${env.ENDPOINTID}
            """
            httpRequest acceptType: 'APPLICATION_JSON', validResponseCodes: '204', httpMode: 'DELETE', ignoreSslErrors: true, url: stackURL, customHeaders:[[name:"Authorization", value: env.JWTTOKEN ], [name: "cache-control", value: "no-cache"]]

          }

        }
      }
    }
    stage('Deploy new stack to Portainer') {
      steps {
        script {
          @NonCPS
          def jsonParse(def json) {
              new groovy.json.JsonSlurperClassic().parseText(json)
          }
          //import groovy.json.JsonSlurperClassic 
          def createStackJson = ""

          // Stack does not exist
          // Generate JSON for when the stack is created          
          def swarmResponse = httpRequest acceptType: 'APPLICATION_JSON', validResponseCodes: '200', httpMode: 'GET', ignoreSslErrors: true, consoleLogResponseBody: true, url: "${portainerURL}/api/endpoints/${env.ENDPOINTID}/docker/swarm", customHeaders:[[name:"Authorization", value: env.JWTTOKEN ], [name: "cache-control", value: "no-cache"]]
          def swarmInfo = new groovy.json.JsonSlurper().parseText(swarmResponse.getContent())

          createStackJson = """
            {"Name": "${SwarmName}", "SwarmID": "$swarmInfo.ID", "RepositoryURL": "${gitURL}", "ComposeFilePathInRepository": "docker-compose.yml", "RepositoryAuthentication": false}
          """

          if(createStackJson?.trim()) {
            jsonParse(httpRequest acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', validResponseCodes: '200', httpMode: 'POST', ignoreSslErrors: true, consoleLogResponseBody: true, requestBody: createStackJson, url: "${portainerURL}/api/stacks?method=repository&type=1&endpointId=${env.ENDPOINTID}", customHeaders:[[name:"Authorization", value: env.JWTTOKEN ], [name: "cache-control", value: "no-cache"]])
          }

        }
      }
    }
  }
}

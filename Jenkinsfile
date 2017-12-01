import groovy.json.JsonSlurperClassic
import groovy.json.JsonSlurper

env.LC_CTYPE = 'en_US.UTF-8'
env.APPNAME = 'DlightPalier'
env.BUNDLEID = 'com.deveryware.dlightpalier'

node('macosx-1') {
    env.FL_UNLOCK_KEYCHAIN_PATH = "~/Library/Keychains/jenkins.keychain"
    env.FASTLANE_XCODE_LIST_TIMEOUT = 120

    stage ('environment') {
        sh "env"
    }

    stage ('git clone') {
        checkout scm
    }

    stage ('install bundler') {
      sh "~/.rbenv/bin/rbenv install -s && ~/.rbenv/bin/rbenv rehash && ~/.rbenv/shims/gem install bundler"
    }

    stage ('update install gems') {
      sh "~/.rbenv/shims/bundle update && ~/.rbenv/shims/bundle install --path .gem"
    }

    def targets = []

    try {
        if ("${TARGET_PREPROD}" == "true") {
            targets.add('preprod')
        }
    } catch (MissingPropertyException e) {
        println 'TARGET_PREPROD is not defined'
    }

    try {
        if ("${TARGET_SNAPSHOT}" == "true") {
            targets.add('snapshot')
        }
    } catch (MissingPropertyException e) {
        println 'TARGET_SNAPSHOT is not defined'
    }

    dir('.') {
        for (target in targets) {
          URL apiUrl = "https://www.appaloosa-store.com/api/v2/189/mobile_application_updates?api_key=yxxiejejz1rstl17yadvmii1rsjx59&group_name=Notico".toURL()
          List json = new JsonSlurper().parse(apiUrl.newReader())
          echo "jsonparsed: ${json.mobile_application_updates[0].application_id}"




            stage ('ios app') {
                sh "sed \"s/${BUNDLEID}/${BUNDLEID}-${target}/g\" config.xml > config.xml.tmp"
                sh "sed \"s/${APPNAME}/${APPNAME}-${target}/g\" config.xml.tmp > config.xml.tmp2"
                sh "cat config.xml.tmp | head"
                sh "cat config.xml.tmp2 | head"
                sh "cat config.xml | head"

                withCredentials([
                    [$class: 'StringBinding', credentialsId: 'FABRIC_API_SECRET', variable: 'FABRIC_API_SECRET'],
                    [$class: 'StringBinding', credentialsId: 'FABRIC_API_KEY', variable: 'FABRIC_API_KEY']
                ]) {
                    withEnv([
                    "FRONT_SERVICE_URL=https://deverylight-${target}.deveryware.team",
                    "MQTT_SERVICE_URL=wss://deverylight-${target}.deveryware.team/mqtt"
                    ]) {
                        stage ('build ios') {
                            echo "FRONT_SERVICE_URL => ${FRONT_SERVICE_URL}"
                            echo "MQTT_SERVICE_URL => ${MQTT_SERVICE_URL}"
                            sh 'npm install && npm install cordova-custom-config && ionic cordova plugin add cordova-fabric-plugin --variable FABRIC_API_SECRET=$FABRIC_API_SECRET --variable FABRIC_API_KEY=$FABRIC_API_KEY && ionic cordova platform add ios && ionic cordova prepare ios'
                        }
                    }
                }

                withCredentials([
                    [$class: 'StringBinding', credentialsId: 'ITUNES_PASSWORD', variable: 'FASTLANE_PASSWORD'],
                    [$class: 'StringBinding', credentialsId: 'KEYCHAIN_PASSWORD', variable: 'FL_UNLOCK_KEYCHAIN_PASSWORD'],
                    [$class: 'StringBinding', credentialsId: 'APPALOOSA_API_TOKEN', variable: 'FL_APPALOOSA_API_TOKEN'],
                    [$class: 'StringBinding', credentialsId: 'APPALOOSA_STORE_ID', variable: 'FL_APPALOOSA_STORE_ID']
                ]) {
                    stage ('build and deploy ios') {
                        sh "~/.rbenv/shims/bundle exec fastlane ios release build:${APPNAME}-${target} to_appaloosa:${TO_APPALOOSA} to_testflight:${TO_TESTFLIGHT}"
                    }
                }

                stage ('archive ios') {
                    archive '**/*.ipa'
                }
            }
        }
    }
}

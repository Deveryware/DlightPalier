import groovy.json.JsonSlurperClassic

env.APPNAME = 'DlightPalier'
env.BUNDLEID = 'com.deveryware.dlightpalier'
env.APPALOOSA_GROUP_IDS = '16136'

@NonCPS
def getVersionNumberIncremented(def storeId, def apiKey, def groupName, def applicationId) {
    URL apiUrl = "https://www.appaloosa-store.com/api/v2/${storeId}/mobile_application_updates?api_key=${apiKey}&group_name=${groupName}".toURL()
    def json = new groovy.json.JsonSlurperClassic().parse(apiUrl.newReader())

    echo "json : ${json}"

    for (def val:json['mobile_application_updates']) {
      echo "val : ${json['mobile_application_updates']}"
      echo "val['application_id'] : ${val['application_id']}"
      echo "applicationId : ${applicationId}"
      if (val['application_id'].equals(applicationId)) {
        echo "inside for if : ${val['application_id']}"
        return val['version'].toInteger() + 1
      }
    }
    return 1
}

node('macosx-1') {

    env.LC_CTYPE = 'en_US.UTF-8'

    //stage ('android environment') {
    //    sh "env"
    //}

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
            stage ("Android app target ${target}") {
                withCredentials([
                    [$class: 'StringBinding', credentialsId: 'FABRIC_API_SECRET', variable: 'FABRIC_API_SECRET'],
                    [$class: 'StringBinding', credentialsId: 'FABRIC_API_KEY', variable: 'FABRIC_API_KEY'],
                    [$class: 'StringBinding', credentialsId: 'APPALOOSA_API_TOKEN', variable: 'FL_APPALOOSA_API_TOKEN'],
                    [$class: 'StringBinding', credentialsId: 'APPALOOSA_STORE_ID', variable: 'FL_APPALOOSA_STORE_ID']
                ]) {
                    withEnv([
                    "FRONT_SERVICE_URL=https://deverylight-${target}.deveryware.team",
                    "MQTT_SERVICE_URL=wss://deverylight-${target}.deveryware.team/mqtt"
                    ]) {
                        stage ('change config.xml for ios') {

                            def versionNumberIncremented = getVersionNumberIncremented("${FL_APPALOOSA_STORE_ID}", "${FL_APPALOOSA_API_TOKEN}", "${APPNAME}", "${BUNDLEID}-${target}")
                            echo "versionNumberIncremented: ${versionNumberIncremented}"
                            sh "sed \"s/${BUNDLEID}/${BUNDLEID}_${target}/g\" config.xml > config.xml.tmp"
                            sh "sed \"s/${APPNAME}/${APPNAME}-${target}/g\" config.xml.tmp > config.xml.tmp2"
                            sh "sed \"s/xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/android-versionCode=\\\"${versionNumberIncremented}\\\" xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/g\" config.xml.tmp2 > config.xml"
                            sh "cat config.xml.tmp | head"
                            sh "cat config.xml.tmp2 | head"
                            sh "cat config.xml | head"

                            echo "FRONT_SERVICE_URL => ${FRONT_SERVICE_URL}"
                            echo "MQTT_SERVICE_URL => ${MQTT_SERVICE_URL}"
                        }

                        stage ('generate android app code with Ionic Cordova') {
                            sh 'npm install && npm install cordova-custom-config && ionic cordova platform rm android && ionic cordova platform add android@6.1.2'
                            sh 'cordova plugin add cordova-plugin-device'
                            sh 'cordova plugin add cordova-plugin-console'
                            sh 'cordova plugin add cordova-plugin-whitelist'
                            sh 'cordova plugin add cordova-plugin-splashscreen'
                            sh 'cordova plugin add cordova-plugin-statusbar'
                            sh 'cordova plugin add cordova-plugin-geolocation'
                            sh 'cordova plugin add ionic-plugin-keyboard'
                            sh 'cordova plugin add cordova-plugin-ios-disableshaketoedit'
                            sh 'cordova plugin add cordova-plugin-crosswalk-webview'
                            sh 'cordova plugin add cordova-plugin-insomnia'
                            sh 'cordova plugin add cordova-plugin-tts'
                            sh 'cordova plugin add cordova-plugin-device-orientation'
                            sh 'cordova plugin add cordova.plugins.diagnostic'
                            sh 'cordova plugin add cordova-open-native-settings'
                            sh 'cordova plugin add cordova-plugin-camera'
                            sh 'cordova plugin add cordova-plugin-file'
                            sh 'cordova plugin add cordova-plugin-add-swift-support'
                            sh 'cordova plugin add cordova-plugin-photo-library'
                            sh 'cordova plugin add cordova-fabric-plugin --variable FABRIC_API_SECRET=$FABRIC_API_SECRET --variable FABRIC_API_KEY=$FABRIC_API_KEY'
                            sh 'cordova plugin add cordova-custom-config --fetch '
                        }

                        stage ('build android with Cordova and Gradle') {
                            sh 'ionic cordova build android --release'
                        }
                    }
                }

                withCredentials([
                     [$class: 'FileBinding', credentialsId: 'KEYSTORE_DEVERYWARE', variable: 'KEYSTORE_PATH'],
                     [$class: 'StringBinding', credentialsId: 'APPALOOSA_API_TOKEN', variable: 'FL_APPALOOSA_API_TOKEN'],
                     [$class: 'StringBinding', credentialsId: 'APPALOOSA_STORE_ID', variable: 'FL_APPALOOSA_STORE_ID']
                ]) {
                   stage ('deploy android') {
                     sh "~/.rbenv/shims/bundle exec fastlane android release app:${APPNAME}-${target} app_identifier:${BUNDLEID}_${target} appaloosa_group_ids:${APPALOOSA_GROUP_IDS} to_appaloosa:${TO_APPALOOSA} to_testflight:${TO_TESTFLIGHT} to_google_play_beta:${TO_GOOGLE_PLAY_BETA}"
                   }
                }

                stage ('archive android') {
                  archive "**/deverylight-${env.PLATFORM}.apk"
                }
            }
        }
    }
}


node('macosx-1') {

    env.LC_CTYPE = 'en_US.UTF-8'
    env.FL_UNLOCK_KEYCHAIN_PATH = "~/Library/Keychains/jenkins.keychain"
    env.FASTLANE_XCODE_LIST_TIMEOUT = 120

    //stage ('ios environment') {
    //    sh "env"
    //}

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
            stage ("ios app target ${target}") {
                withCredentials([
                    [$class: 'StringBinding', credentialsId: 'FABRIC_API_SECRET', variable: 'FABRIC_API_SECRET'],
                    [$class: 'StringBinding', credentialsId: 'FABRIC_API_KEY', variable: 'FABRIC_API_KEY'],
                    [$class: 'StringBinding', credentialsId: 'ITUNES_PASSWORD', variable: 'FASTLANE_PASSWORD'],
                    [$class: 'StringBinding', credentialsId: 'KEYCHAIN_PASSWORD', variable: 'FL_UNLOCK_KEYCHAIN_PASSWORD'],
                    [$class: 'StringBinding', credentialsId: 'APPALOOSA_API_TOKEN', variable: 'FL_APPALOOSA_API_TOKEN'],
                    [$class: 'StringBinding', credentialsId: 'APPALOOSA_STORE_ID', variable: 'FL_APPALOOSA_STORE_ID']

                ]) {
                    withEnv([
                    "FRONT_SERVICE_URL=https://deverylight-${target}.deveryware.team",
                    "MQTT_SERVICE_URL=wss://deverylight-${target}.deveryware.team/mqtt"
                    ]) {
                        stage ('change config.xml for ios') {

                            def versionNumberIncremented = getVersionNumberIncremented("${FL_APPALOOSA_STORE_ID}", "${FL_APPALOOSA_API_TOKEN}", "${APPNAME}", "${BUNDLEID}-${target}")

                            sh "sed \"s/${BUNDLEID}/${BUNDLEID}-${target}/g\" config.xml > config.xml.tmp"
                            sh "sed \"s/${APPNAME}/${APPNAME}-${target}/g\" config.xml.tmp > config.xml.tmp2"
                            sh "sed \"s/xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/ios-CFBundleVersion=\\\"${versionNumberIncremented}\\\" xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/g\" config.xml.tmp2 > config.xml"
                            sh "cat config.xml.tmp | head"
                            sh "cat config.xml.tmp2 | head"
                            sh "cat config.xml | head"

                            echo "FRONT_SERVICE_URL => ${FRONT_SERVICE_URL}"
                            echo "MQTT_SERVICE_URL => ${MQTT_SERVICE_URL}"
                        }

                        stage ('generate ios app code with Ionic Cordova') {
                            sh 'npm install && npm install cordova-custom-config && ionic cordova plugin add cordova-fabric-plugin --variable FABRIC_API_SECRET=$FABRIC_API_SECRET --variable FABRIC_API_KEY=$FABRIC_API_KEY && ionic cordova platform add ios && ionic cordova prepare ios'
                        }

                        stage ('build, sign and deploy ios with Fastlane') {
                            sh "~/.rbenv/shims/bundle exec fastlane ios release app:${APPNAME}-${target} app_identifier:${BUNDLEID}-${target} appaloosa_group_ids:${APPALOOSA_GROUP_IDS} to_appaloosa:${TO_APPALOOSA} to_testflight:${TO_TESTFLIGHT}"
                        }
                    }
                }

                stage ('archive ios') {
                    archive '**/*.ipa'
                }
            }
        }
    }
}


import groovy.json.JsonSlurperClassic

env.APPNAME = 'DlightPalier'
env.BUNDLEID = 'com.deveryware.dlightpalier'
env.APPALOOSA_GROUP_IDS = '16136'
env.MOBILE_DIRECTORY = '.'

@NonCPS
def getVersionNumberIncremented(def storeId, def apiKey, def groupName, def applicationId, def android) {
    URL apiUrl = "https://www.appaloosa-store.com/api/v2/${storeId}/mobile_application_updates?api_key=${apiKey}&group_name=${groupName}".toURL()
    def json = new groovy.json.JsonSlurperClassic().parse(apiUrl.newReader())

     for (def val:json['mobile_application_updates']) {
      if (val['application_id'] == applicationId) {
        def existingVersion = val['version']
        if ("${android}" == "true") {
          def existingVersionTruncated = existingVersion.substring(0, existingVersion.length() - 1)
          return existingVersionTruncated.toInteger() + 1
        }
        return existingVersion.toInteger() + 1
      }
    }
    return 1
}

// BUILD ANDROID

node('macosx-1') {

    env.LC_CTYPE = 'en_US.UTF-8'

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

    if ("${TO_APPALOOSA}" == "true") {
        for (target in targets) {

            stage ('git clone') {
                checkout scm
            }

            stage ('install bundler') {
              sh "~/.rbenv/bin/rbenv install -s && ~/.rbenv/bin/rbenv rehash && ~/.rbenv/shims/gem install bundler"
            }

            stage ('update install gems') {
              sh "~/.rbenv/shims/bundle update && ~/.rbenv/shims/bundle install --path .gem"
            }

            dir("${MOBILE_DIRECTORY}") {
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
                        stage ('change config.xml for android') {
                          if ("${TO_APPALOOSA}" == "true") {
                             def versionNumberIncremented = getVersionNumberIncremented("${FL_APPALOOSA_STORE_ID}", "${FL_APPALOOSA_API_TOKEN}", "${APPNAME}", "${BUNDLEID}_${target}", "true")
                             echo "versionNumberIncremented: ${versionNumberIncremented}"

                             sh "sed \"s/${BUNDLEID}/${BUNDLEID}_${target}/g\" config.xml > config.xml.tmp"
                             sh "sed \"s/${APPNAME}/${APPNAME}-${target}/g\" config.xml.tmp > config.xml.tmp2"
                             sh "sed \"s/xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/android-versionCode=\\\"${versionNumberIncremented}\\\" xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/g\" config.xml.tmp2 > config.xml"
                             sh "cat config.xml | head"
                          }

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
                     sh "~/.rbenv/shims/bundle exec fastlane android to_appaloosa app:${APPNAME}-${target} appaloosa_group_ids:${APPALOOSA_GROUP_IDS}"
                   }
                }

                stage ('archive android') {
                  archive "**/${APPNAME}-${target}.apk"
                }
            }
        }
    }

    if ("${TO_GOOGLE_PLAY_BETA}" == "true") {
        stage ('git clone') {
            checkout scm
        }

        stage ('install bundler') {
          sh "~/.rbenv/bin/rbenv install -s && ~/.rbenv/bin/rbenv rehash && ~/.rbenv/shims/gem install bundler"
        }

        stage ('update install gems') {
          sh "~/.rbenv/shims/bundle update && ~/.rbenv/shims/bundle install --path .gem"
        }


        withCredentials([
            [$class: 'FileBinding', credentialsId: 'GOOGLE_PLAYSTORE_JSON', variable: 'SUPPLY_JSON_KEY']
        ]) {
           stage ('deploy android') {
              sh '~/.rbenv/shims/bundle exec fastlane run google_play_track_version_codes package_name:com.deveryware.deverylight track:beta'
           }
        }


        dir("${MOBILE_DIRECTORY}") {
            withCredentials([
                [$class: 'StringBinding', credentialsId: 'FABRIC_API_SECRET', variable: 'FABRIC_API_SECRET'],
                [$class: 'StringBinding', credentialsId: 'FABRIC_API_KEY', variable: 'FABRIC_API_KEY'],
                [$class: 'StringBinding', credentialsId: 'APPALOOSA_API_TOKEN', variable: 'FL_APPALOOSA_API_TOKEN'],
                [$class: 'StringBinding', credentialsId: 'APPALOOSA_STORE_ID', variable: 'FL_APPALOOSA_STORE_ID'],
                [$class: 'FileBinding', credentialsId: 'GOOGLE_PLAYSTORE_JSON', variable: 'SUPPLY_JSON_KEY']
            ]) {
                withEnv([
                  "FRONT_SERVICE_URL=https://deverylight-prod.deveryware.team",
                  "MQTT_SERVICE_URL=wss://deverylight-prod.deveryware.team/mqtt"
                ]) {
                   echo "FRONT_SERVICE_URL => ${FRONT_SERVICE_URL}"
                   echo "MQTT_SERVICE_URL => ${MQTT_SERVICE_URL}"

                   sh "~/.rbenv/shims/bundle exec fastlane run google_play_track_version_codes package_name:com.deveryware.deverylight track:beta  | grep 'Result: ' | sed 's/.*Result: \\[\\([0-9]*\\).*/\\1/' > build_number_google_play.txt"
                   def build_number_google_play = readFile('build_number_google_play.txt').trim()
                   echo "build_number_google_play: ${build_number_google_play}"
                   def build_number_incremented = build_number_google_play.toInteger() + 1
                   echo "build_number_incremented: ${build_number_incremented}"

                   sh "sed \"s/xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/android-versionCode=\\\"${build_number_incremented}\\\" xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/g\" config.xml > config.xml.tmp"
                   sh "cat config.xml.tmp > config.xml"


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
                  sh "~/.rbenv/shims/bundle exec fastlane android to_google_play_beta app:${APPNAME}"
               }
            }

            stage ('archive android') {
              archive "**/${APPNAME}.apk"
            }
        }

    }
}

// BUILD iOS

node('macosx-1') {


    env.LC_CTYPE = 'en_US.UTF-8'
    env.FL_UNLOCK_KEYCHAIN_PATH = "~/Library/Keychains/jenkins.keychain"
    env.FASTLANE_XCODE_LIST_TIMEOUT = 120

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

    if ("${TO_APPALOOSA}" == "true") {
        for (target in targets) {

            stage ('git clone') {
                checkout scm
            }

            stage ('install bundler') {
              sh "~/.rbenv/bin/rbenv install -s && ~/.rbenv/bin/rbenv rehash && ~/.rbenv/shims/gem install bundler"
            }

            stage ('update install gems') {
              sh "~/.rbenv/shims/bundle update && ~/.rbenv/shims/bundle install --path .gem"
            }

            dir("${MOBILE_DIRECTORY}") {
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
                            if ("${TO_APPALOOSA}" == "true") {
                              def versionNumberIncremented = getVersionNumberIncremented("${FL_APPALOOSA_STORE_ID}", "${FL_APPALOOSA_API_TOKEN}", "${APPNAME}", "${BUNDLEID}-${target}", false)

                              sh "sed \"s/${BUNDLEID}/${BUNDLEID}-${target}/g\" config.xml > config.xml.tmp"
                              sh "sed \"s/${APPNAME}/${APPNAME}-${target}/g\" config.xml.tmp > config.xml.tmp2"
                              sh "sed \"s/xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/ios-CFBundleVersion=\\\"${versionNumberIncremented}\\\" xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/g\" config.xml.tmp2 > config.xml"
                              sh "cat config.xml | head"
                            }

                            echo "FRONT_SERVICE_URL => ${FRONT_SERVICE_URL}"
                            echo "MQTT_SERVICE_URL => ${MQTT_SERVICE_URL}"
                        }

                        stage ('generate ios app code with Ionic Cordova') {
                            sh "npm install && npm install cordova-custom-config && ionic cordova plugin add cordova-fabric-plugin --variable FABRIC_API_SECRET=$FABRIC_API_SECRET --variable FABRIC_API_KEY=$FABRIC_API_KEY && ionic cordova platform add ios && ionic cordova prepare ios"
                        }

                        stage ('build, sign and deploy ios with Fastlane') {
                            sh "~/.rbenv/shims/bundle exec fastlane ios to_appaloosa app:${APPNAME}-${target} app_identifier:${BUNDLEID}-${target} appaloosa_group_ids:${APPALOOSA_GROUP_IDS}"
                        }

                        stage ('archive ios') {
                            archive '**/${APPNAME}-${target}.ipa'
                        }
                    }
                }
            }
        }
    }

    if ("${TO_TESTFLIGHT}" == "true") {
        stage ('git clone') {
            checkout scm
        }

        stage ('install bundler') {
          sh "~/.rbenv/bin/rbenv install -s && ~/.rbenv/bin/rbenv rehash && ~/.rbenv/shims/gem install bundler"
        }

        stage ('update install gems') {
          sh "~/.rbenv/shims/bundle update && ~/.rbenv/shims/bundle install --path .gem"
        }


        withCredentials([
            [$class: 'StringBinding', credentialsId: 'ITUNES_PASSWORD', variable: 'FASTLANE_PASSWORD']
        ]) {

        }

        dir("${MOBILE_DIRECTORY}") {
            withCredentials([
                [$class: 'StringBinding', credentialsId: 'FABRIC_API_SECRET', variable: 'FABRIC_API_SECRET'],
                [$class: 'StringBinding', credentialsId: 'FABRIC_API_KEY', variable: 'FABRIC_API_KEY'],
                [$class: 'StringBinding', credentialsId: 'ITUNES_PASSWORD', variable: 'FASTLANE_PASSWORD'],
                [$class: 'StringBinding', credentialsId: 'KEYCHAIN_PASSWORD', variable: 'FL_UNLOCK_KEYCHAIN_PASSWORD'],
                [$class: 'StringBinding', credentialsId: 'APPALOOSA_API_TOKEN', variable: 'FL_APPALOOSA_API_TOKEN'],
                [$class: 'StringBinding', credentialsId: 'APPALOOSA_STORE_ID', variable: 'FL_APPALOOSA_STORE_ID']

            ]) {
                withEnv([
                  "FRONT_SERVICE_URL=https://deverylight-prod.deveryware.net",
                  "MQTT_SERVICE_URL=wss://deverylight-prod.deveryware.net/mqtt"
                ]) {

                    echo "FRONT_SERVICE_URL => ${FRONT_SERVICE_URL}"
                    echo "MQTT_SERVICE_URL => ${MQTT_SERVICE_URL}"

                    //sh "cat config.xml | grep 'version=\"' | sed 's/.*version=\"\\(.*\\)\" xmlns=.*/\\1/' > version_number.txt"
                    //def version_number = readFile('version_number.txt').trim()
                    //echo "version_number: ${version_number}"

                    sh "~/.rbenv/shims/bundle exec fastlane run latest_testflight_build_number version:get_version_number | grep 'Result: ' | sed 's/.*Result: \\([0-9]*\\).*/\\1/' > build_number_itunesconnect.txt"
                    def build_number_itunesconnect = readFile('build_number_itunesconnect.txt').trim()
                    echo "build_number_itunesconnect: ${build_number_itunesconnect}"
                    def build_number_incremented = build_number_itunesconnect.toInteger() + 1
                    echo "build_number_incremented: ${build_number_incremented}"

                    sh "sed \"s/xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/ios-CFBundleVersion=\\\"${build_number_incremented}\\\" xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/g\" config.xml > config.xml.tmp"
                    sh "cat config.xml.tmp > config.xml"


                    stage ('generate ios app code with Ionic Cordova') {
                        sh "npm install && npm install cordova-custom-config && ionic cordova plugin add cordova-fabric-plugin --variable FABRIC_API_SECRET=$FABRIC_API_SECRET --variable FABRIC_API_KEY=$FABRIC_API_KEY && ionic cordova platform add ios && ionic cordova prepare ios"
                    }

                    stage ('build, sign and deploy ios with Fastlane') {
                        sh "~/.rbenv/shims/bundle exec fastlane ios to_testflight app:${APPNAME} app_identifier:${BUNDLEID}"
                    }

                    stage ('archive ios') {
                        archive '**/${APPNAME}.ipa'
                    }
                }
            }
        }
    }
}












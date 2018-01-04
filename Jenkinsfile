import groovy.json.JsonSlurperClassic

def APPNAME_DEV = 'DlightPalier'
def APPNAME_STORE = 'DlightPalier'
def BUNDLEID = 'com.deveryware.dlightpalier'
def APPALOOSA_GROUP_IDS = '16136'
def MOBILE_DIRECTORY = '.'

def TARGET_PREPROD = "${params.DEPLOY_ENV}"
def TARGET_SNAPSHOT = "${TARGET_SNAPSHOT}"
def TO_APPALOOSA = "${TO_APPALOOSA}"
def TO_TESTFLIGHT = "${TO_TESTFLIGHT}"
def TO_GOOGLE_PLAY_BETA = "${TO_GOOGLE_PLAY_BETA}"

@NonCPS
def getAppaloosaBuildNumberIncremented(def storeId, def apiKey, def groupName, def applicationId, def android) {
    URL apiUrl = "https://www.appaloosa-store.com/api/v2/$storeId/mobile_application_updates?api_key=$apiKey&group_name=$groupName".toURL()
    def json = new groovy.json.JsonSlurperClassic().parse(apiUrl.newReader())
        for (def val:json['mobile_application_updates']) {
        if (val['application_id'] == applicationId) {
          def existingBuildNumber = val['version']
          if (android == "true") {
            def existingBuildNumberTruncated = existingBuildNumber.substring(0, existingBuildNumber.length() - 1)
            return existingBuildNumberTruncated.toInteger() + 1
          }
          return existingBuildNumber.toInteger() + 1
        }
    }
    return 1
}

def getTestFlightBuildNumberIncremented(def applicationId, def version) {

    sh "~/.rbenv/shims/bundle exec fastlane run latest_testflight_build_number version:$version | grep 'Result: ' | sed 's/.*Result: \\([0-9]*\\).*/\\1/' > build_number_itunesconnect.txt"
    def build_number_itunesconnect = readFile('build_number_itunesconnect.txt').trim()
    if (build_number_itunesconnect?.trim()) {
        def build_number_incremented = build_number_itunesconnect.toInteger() + 1
        return build_number_incremented
    }
    return 1
}

def getGooglePlayBuildNumberIncremented(def applicationId) {
    sh "~/.rbenv/shims/bundle exec fastlane run google_play_track_version_codes package_name:$applicationId track:beta  | grep 'Result: ' | sed 's/.*Result: \\[\\([0-9]*\\).*/\\1/' > build_number_google_play.txt"
    def build_number_google_play = readFile('build_number_google_play.txt').trim()
    if (build_number_google_play?.trim()) {
        def existingBuildNumberTruncated = build_number_google_play.substring(0, build_number_google_play.length() - 1)
        return existingBuildNumberTruncated.toInteger() + 1
    }
    return 1
}


// BUILD iOS

node('macosx-1') {

    env.LC_ALL = 'en_US.UTF-8'
    env.LANG = 'en_US.UTF-8'
    env.LC_CTYPE = 'en_US.UTF-8'
    env.FL_UNLOCK_KEYCHAIN_PATH = "~/Library/Keychains/jenkins.keychain"
    env.FASTLANE_XCODE_LIST_TIMEOUT = 120

    def stores = []

    if (TO_APPALOOSA == "true") {
        stores.add('to_appaloosa')
    }

    if (TO_TESTFLIGHT == "true") {
        stores.add('to_testflight')
    }

    for (store in stores) {

        def targets = []

        switch (store) {
            case "to_appaloosa":
                if (TARGET_PREPROD == "true") {
                    targets.add('preprod')
                }

                if (TARGET_SNAPSHOT == "true") {
                    targets.add('snapshot')
                }
                break
            case "to_testflight":
                targets.add('prod')
                break
            default:
                break
        }

        for (target in targets) {

            stage ("iOS: $target - $store") {

                echo "### iOS $target $store ###"

                checkout scm
                sh "~/.rbenv/bin/rbenv install -s && ~/.rbenv/bin/rbenv rehash && ~/.rbenv/shims/gem install bundler"
                sh "~/.rbenv/shims/bundle update && ~/.rbenv/shims/bundle install --path .gem"

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

                            echo "FRONT_SERVICE_URL => $FRONT_SERVICE_URL"
                            echo "MQTT_SERVICE_URL => $MQTT_SERVICE_URL"

                            switch (store) {
                                case "to_appaloosa":
                                    def buildNumberIncremented = getAppaloosaBuildNumberIncremented(FL_APPALOOSA_STORE_ID, FL_APPALOOSA_API_TOKEN, APPNAME_DEV, "${BUNDLEID}-${target}", false)
                                    sh "sed \"s/$BUNDLEID/$BUNDLEID-$target/g\" config.xml > config.xml.tmp"
                                    sh "sed \"s/$APPNAME_DEV/$APPNAME_DEV-$target/g\" config.xml.tmp > config.xml.tmp2"
                                    sh "sed \"s/xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/ios-CFBundleVersion=\\\"$buildNumberIncremented\\\" xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/g\" config.xml.tmp2 > config.xml"
                                    sh "cat config.xml | head"
                                    sh "npm install && npm install cordova-custom-config && ionic cordova plugin add cordova-fabric-plugin --variable FABRIC_API_SECRET=$FABRIC_API_SECRET --variable FABRIC_API_KEY=$FABRIC_API_KEY && ionic cordova platform add ios && ionic cordova prepare ios"
                                    sh "~/.rbenv/shims/bundle exec fastlane ios to_appaloosa app:$APPNAME_DEV-$target app_identifier:$BUNDLEID-$target appaloosa_group_ids:$APPALOOSA_GROUP_IDS"
                                    archive '**/${APPNAME}-${target}.ipa'
                                    break
                                case "to_testflight":
                                    sh "cat config.xml | grep 'version=\"' | sed 's/.*version=\"\\(.*\\)\" xmlns=.*/\\1/' > version_number.txt"
                                    def version_number = readFile('version_number.txt').trim()
                                    def build_number_incremented = getTestFlightBuildNumberIncremented(BUNDLEID, version_number)
                                    sh "sed \"s/xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/ios-CFBundleVersion=\\\"$build_number_incremented\\\" xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/g\" config.xml > config.xml.tmp"
                                    sh "cat config.xml.tmp > config.xml"
                                    sh "cat config.xml | head"
                                    sh "npm install && npm install cordova-custom-config && ionic cordova plugin add cordova-fabric-plugin --variable FABRIC_API_SECRET=$FABRIC_API_SECRET --variable FABRIC_API_KEY=$FABRIC_API_KEY && ionic cordova platform add ios && ionic cordova prepare ios"
                                    sh "~/.rbenv/shims/bundle exec fastlane ios to_testflight app:$APPNAME_STORE app_identifier:$BUNDLEID"
                                    archive '**/${APPNAME_STORE}.ipa'
                                    break
                                default:
                                    break
                            }
                        }
                    }
                }
            }
        }
    }
}


// BUILD ANDROID

node('macosx-1') {

    env.LC_ALL = 'en_US.UTF-8'
    env.LANG = 'en_US.UTF-8'
    env.LC_CTYPE = 'en_US.UTF-8'

    def stores = []

    if (TO_APPALOOSA == "true") {
        stores.add('to_appaloosa')
    }

    if (TO_GOOGLE_PLAY_BETA == "true") {
        stores.add('to_google_play_beta')
    }

    for (store in stores) {

        def targets = []

        switch (store) {
            case "to_appaloosa":
                if (TARGET_PREPROD == "true") {
                    targets.add('preprod')
                }

                if ("${TARGET_SNAPSHOT}" == "true") {
                    targets.add('snapshot')
                }
                break
            case "to_google_play_beta":
                targets.add('prod')
                break
            default:
                break
        }

        for (target in targets) {

            stage ("ANDROID: $target - $store") {

                echo "### ANDROID $target $store ###"

                checkout scm
                sh "~/.rbenv/bin/rbenv install -s && ~/.rbenv/bin/rbenv rehash && ~/.rbenv/shims/gem install bundler"
                sh "~/.rbenv/shims/bundle update && ~/.rbenv/shims/bundle install --path .gem"

                dir("${MOBILE_DIRECTORY}") {
                    withCredentials([
                        [$class: 'FileBinding', credentialsId: 'KEYSTORE_DEVERYWARE', variable: 'KEYSTORE_PATH'],
                        [$class: 'StringBinding', credentialsId: 'FABRIC_API_SECRET', variable: 'FABRIC_API_SECRET'],
                        [$class: 'StringBinding', credentialsId: 'FABRIC_API_KEY', variable: 'FABRIC_API_KEY'],
                        [$class: 'StringBinding', credentialsId: 'APPALOOSA_API_TOKEN', variable: 'FL_APPALOOSA_API_TOKEN'],
                        [$class: 'StringBinding', credentialsId: 'APPALOOSA_STORE_ID', variable: 'FL_APPALOOSA_STORE_ID'],
                        [$class: 'FileBinding', credentialsId: 'GOOGLE_PLAYSTORE_JSON', variable: 'SUPPLY_JSON_KEY']
                    ]) {
                        withEnv([
                            "FRONT_SERVICE_URL=https://deverylight-${target}.deveryware.team",
                            "MQTT_SERVICE_URL=wss://deverylight-${target}.deveryware.team/mqtt"
                        ]) {

                            echo "FRONT_SERVICE_URL => $FRONT_SERVICE_URL"
                            echo "MQTT_SERVICE_URL => $MQTT_SERVICE_URL"

                            switch (store) {
                                case "to_appaloosa":
                                    def buildNumberIncremented = getAppaloosaBuildNumberIncremented(FL_APPALOOSA_STORE_ID, FL_APPALOOSA_API_TOKEN, APPNAME_DEV, "${BUNDLEID}_${target}", "true")
                                    sh "sed \"s/$BUNDLEID/${BUNDLEID}_${target}/g\" config.xml > config.xml.tmp"
                                    sh "sed \"s/$APPNAME_DEV/$APPNAME_DEV-$target/g\" config.xml.tmp > config.xml.tmp2"
                                    sh "sed \"s/xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/android-versionCode=\\\"$buildNumberIncremented\\\" xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/g\" config.xml.tmp2 > config.xml"
                                    sh "cat config.xml | head"

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
                                    sh 'ionic cordova build android --release'

                                    sh "~/.rbenv/shims/bundle exec fastlane android to_appaloosa app:$APPNAME_DEV-$target appaloosa_group_ids:$APPALOOSA_GROUP_IDS"
                                    archive "**/${APPNAME_DEV}-${target}.apk"
                                    break
                                case "to_google_play_beta":
                                    def buildNumberIncremented = getGooglePlayBuildNumberIncremented(BUNDLEID)
                                    sh "sed -i '' \"s/xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/android-versionCode=\\\"$buildNumberIncremented\\\" xmlns=\\\"http:\\/\\/www.w3.org\\/ns\\/widgets\\\"/g\" config.xml"

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
                                    sh 'ionic cordova build android --release'

                                    sh "~/.rbenv/shims/bundle exec fastlane android to_google_play_beta app:$APPNAME_STORE"
                                    archive "**/${APPNAME_STORE}.apk"
                                    break
                                default:
                                    break
                            }
                        }
                    }
                }
            }
        }
    }
}

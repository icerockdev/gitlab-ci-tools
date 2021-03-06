fastlane_version "2.70.0"

default_platform :ios

# Constants
K_NAME = "temp_keychain"
K_PASS = "temp_password"

platform :ios do

  lane :fill_keychain do |options|
        # создаем временный кейчейн
    create_keychain(
      name: K_NAME,
      password: K_PASS,
      default_keychain: false,
      unlock: true,
      timeout: 3600,
      lock_when_sleeps: false
    )

    # загружаем WWDR сертификаты
    sh("curl -o wwdr_2023.cer 'https://developer.apple.com/certificationauthority/AppleWWDRCA.cer'")
    sh("security add-certificates -k #{K_NAME} wwdr_2023.cer || true")

    sh("curl -o wwdr_2030.cer 'https://www.apple.com/certificateauthority/AppleWWDRCAG3.cer'")
    sh("security add-certificates -k #{K_NAME} wwdr_2030.cer || true")

    # загружаем данные для подписи
    match(app_identifier: ENV["BUILD_APP_IDENTIFIER"], 
      type: options[:type], 
      keychain_name: K_NAME,
      keychain_password: K_PASS,
      verbose: true)

    sh("security set-key-partition-list -S apple-tool:,apple:,codesign: -s -k #{K_PASS} #{K_NAME} 1> /dev/null")
  end

  lane :build do
    # если есть, устанавливаем номер билда в настройки проекта
    build_number = ENV["BUILD_NUMBER"]
    if !(build_number.nil?)
      # добавляем в иконку для тестовых билдов
      is_appstore = ENV["IS_APPSTORE"]
      if (is_appstore.nil?)
        ci_commit_hash = ENV["CI_COMMIT_SHORT_SHA"]
        sh("badge --shield '#{build_number}-#{ci_commit_hash}-green' --no_badge --verbose --glob '/../**/*.appiconset/*.{png,PNG}'")
      end

      increment_build_number(build_number: ENV["BUILD_NUMBER"])
    end
    # убираем автоподпись
    disable_automatic_code_signing

    fill_keychain(type: "development")

    # ставим поды
    cocoapods(try_repo_update_on_error: true)
    # собираем билд
    gym({
      scheme: ENV["BUILD_SCHEME"], 
      export_method: 'development', 
      skip_package_ipa: true, 
      clean: true, 
      build_path: 'archive'
    })
    
    # вытаскиваем архив
    sh("cd .. && mv '#{lane_context[SharedValues::XCODEBUILD_ARCHIVE]}' #{ENV["XCARCHIVE_NAME"]}")
    # если указано куда - пакуем
    zip_name = ENV["XCZIP_NAME"]
    if !(zip_name.nil?)
      sh("cd .. && zip -r #{ENV["XCZIP_NAME"]} #{ENV["XCARCHIVE_NAME"]}")
    end
  end

  lane :deploy_appstore do
    zip_name = ENV["XCZIP_NAME"]
    if !(zip_name.nil?)
      sh("cd .. && unzip #{ENV["XCZIP_NAME"]}")
    end

    fill_keychain(type: "appstore")

    # экспортируем приложение
    gym({
      scheme: ENV["BUILD_SCHEME"], 
      skip_build_archive: true,
      export_method: 'app-store', 
      archive_path: ENV["XCARCHIVE_NAME"],
      output_name: ENV["IPA_NAME"]
    })

    # отправляем в appstore
    upload_to_testflight(
      skip_submission: true,
      skip_waiting_for_build_processing: true
    )
  end

  lane :deploy_firebase do
    zip_name = ENV["XCZIP_NAME"]
    if !(zip_name.nil?)
      sh("cd .. && unzip #{ENV["XCZIP_NAME"]}")
    end
    match_type = ENV["BUILD_EXPORT_METHOD"].tr('-', '')
    notes_path = ENV["CHANGELOG_PATH"]

    fill_keychain(type: match_type)

    # экспортируем приложение
    gym({
      scheme: ENV["BUILD_SCHEME"], 
      skip_build_archive: true,
      export_method: ENV["BUILD_EXPORT_METHOD"], 
      archive_path: ENV["XCARCHIVE_NAME"],
      output_name: ENV["IPA_NAME"]
    })


    sh("firebase appdistribution:distribute #{ENV["IPA_OUTPUT_PATH"]}  \
      --app #{ENV["FIREBASE_APP_ID"]} \
      --groups #{ENV["BUILD_CRASHLYTICS_TEST_GROUPS"]}")
    
    # Выгрузка пока закомментирована, т.к. FirebaseCrashlytics пока отдаёт ошибку по этой команде, не видит upload-symbols
    # upload_symbols_to_crashlytics(binary_path: "./Pods/FirebaseCrashlytics/upload-symbols", gsp_path: "./GoogleService-Info.plist")
  end

  after_all do |lane, options|
    delete_keychain(name: K_NAME)
  end

  error do |lane, exception, options|
    delete_keychain(name: K_NAME)
  end

  lane :register_new_devices do
    register_devices(devices_file: ENV["DEVICES_FILE"])    
  end
  
end
# はじめに

とあるプロジェクトに参画した時に、テストアプリ配布を全て手動で行なっており、テストフェーズに入った時に、非効率すぎてめちゃめちゃストレスになりました...
自動化したいなと思い、とりあえず無料で手っ取り早くできるものを模索した結果、fastlane と firebase app distribution を使用してテストアプリを配布を効率化することにしました。

# 事前準備

- テストアプリ配布用の Bundle Identifier
- テストアプリ配布用(AdHock)の証明書
- Firebase プロジェクトに app distribution を追加
- Firebase のアプリ ID と CLI TOKEN の発行

# fastlane インストール

### fastlane を homebrew でインストール

https://formulae.brew.sh/formula/fastlane

```
brew install fastlane
```

# iOS のセットアップ

### ios ディレクトリで下記を実行

```
cd ios

/// fastlaneを初期化
/// fastlaneディレクトリが生成される
fastlane init

/// distriutionのプラグインを追加
fastlane add_plugin firebase_app_distribution
```

### .env ファイルに必要な情報を設定

```
### iOSテストアプリ配布用設定
FIREBASE_APP_ID="" // Firebaseで登録されるアプリID
FIREBASE_CLI_TOKEN="" // FirebaseのCLI TOKEN
APP_IDENTIFIER="" // iosアプリのバンドルID
APPLE_ID="" // xcodeに設定のapple id
PROVISIONING='' // Provisioning Profile名
TEAM_ID = '' // Signing Certificate チームID
GROUPS='' // distributionに登録のグループ名
```

### Appfile に下記を追加

```ruby
require 'dotenv'
Dotenv.load

app_identifier(ENV['APP_IDENTIFIER'])
apple_id(ENV['APPLE_ID'])
team_id(ENV['TEAM_ID'])
```

### FastFile に下記を追加

基本的には.env ファイルの情報を入力しておけばコピペで実行できる

```ruby
require 'dotenv'
Dotenv.load

default_platform(:ios)

platform :ios do
  desc "iosテストアプリ配布"
  lane :deploy do

    project_name = "Runner.xcodeproj"
    workspace_name = "Runner.xcworkspace"
    scheme_name = "Runner"
    output_directory = "build"
    ipa_path = "#{output_directory}/#{scheme_name}.ipa"
    release_notes_file = ENV['PWD'] + "/fastlane/release_note.txt"

    # flutter build
    Dir.chdir "../.." do
      sh("flutter", "clean")
      sh("flutter", "pub", "get")
      sh("flutter", "build", "ios", "--release", "--no-codesign")
    end

    # xcode archive
    build_app(
      workspace: workspace_name,
      export_options: {
        method: "ad-hoc",
        provisioningProfiles: {
          ENV['APP_IDENTIFIER'] => ENV['PROVISIONING']
        },
        signingStyle: "manual"
      },
      clean: true,
      export_xcargs: "-allowProvisioningUpdates",
      output_name: "#{scheme_name}.ipa",
      output_directory: "#{output_directory}"
    )

    # deploy
    firebase_app_distribution(
      app: ENV['FIREBASE_APP_ID'],
      firebase_cli_token: ENV['FIREBASE_CLI_TOKEN'],
      ipa_path: ipa_path,
      groups: ENV['GROUPS'],
      release_notes_file: release_notes_file,
    )
  end
end
```

### release_note.txt を作成

リリースノートの作成も fastlane から可能です。
txt ファイルで作成する必要があります。

```
- XXXの実装を行いました。
- XXXの修正を行いました。
```

### android のセットアップ

### android ディレクトリで下記を実行

```
cd android

/// fastlaneを初期化
/// fastlaneディレクトリが生成される
fastlane init

/// distriutionのプラグインを追加
fastlane add_plugin firebase_app_distribution
```

### .env ファイルに必要な情報を設定

```
### androidテストアプリ配布用設定
FIREBASE_APP_ID=""
FIREBASE_CLI_TOKEN=""
PACKAGE_NAME="" // androidのapplicationId
GROUPS=''
```

### Appfileに下記を追加
```ruby
require 'dotenv'
Dotenv.load

json_key_file("")
package_name(ENV['PACKAGE_NAME'])
```

### FastFile に下記を追加

基本的には.env ファイルの情報を入力しておけばコピペで実行できる

```ruby
default_platform(:android)
platform :android do

desc "androidテストアプリ配布"
lane :deploy do

  apk_path = "../build/app/outputs/apk/release/app-release.apk"
  release_notes_file = ENV['PWD'] + "/fastlane/release_note.txt"
  
  # flutter build
  Dir.chdir "../.." do
    sh("flutter", "clean")
    sh("flutter", "pub", "get")
    sh("flutter", "build", "apk", "--release")
  end

  # deploy
  firebase_app_distribution(
          app: ENV['FIREBASE_APP_ID'], 
          firebase_cli_token: ENV['FIREBASE_CLI_TOKEN'], 
          android_artifact_type: "APK",
          android_artifact_path: apk_path,
          groups: ENV['GROUPS'],
          release_notes_file: release_notes_file,
      )
  end
end
```

### release_note.txt を作成
リリースノートの作成も fastlane から可能です。
txt ファイルで作成する必要があります。

```
- XXXの実装を行いました。
- XXXの修正を行いました。
```

### 実行する

### iOS
```
cd ios
fastlane ios deploy
```

### android
```
cd android
fastlane android deploy
```

# 最後に
コマンドからテストアプリの自動化をしてみたい方はぜひやって見てください！
また、build numberを自動でインクリメントしたりなど、色々カスタマイズができるので、好みの形に実装するのがいいかと思います。

# fastlane_deploy_to_app_distribution

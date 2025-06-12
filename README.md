# KMP Publish iOS App on Firebase Action

This GitHub Action automates the process of building and publishing iOS applications to Firebase App
Distribution. It handles the build process, signing, and deployment using Fastlane.

## Features

- Automated Firebase App Distribution deployment
- iOS IPA build and signing
- Build artifact archiving
- Gradle and Ruby dependency caching
- Firebase tester group management
- SSH key authentication via base64 secrets

## Prerequisites

Before using this action, ensure you have:

1. A Firebase project with App Distribution enabled
2. Firebase service account credentials
3. An iOS app set up in Firebase
4. Xcode project configured with proper signing
5. A private GitHub repo with certificates and provisioning profiles for Fastlane Match
6. SSH key added as a deploy key to the certificates repo

## Setup

### SSH Key Setup

1. Generate SSH Key Locally<br>
   In your terminal, run:

```yaml
ssh-keygen -t ed25519 -C "your_email@example.com"
```

It will ask for a file path. Press enter a custom path like:

```yaml
~/.ssh/match_ci_key
```

You can skip setting a passphrase when prompted (just hit enter twice).

This generates two files:

- `~/.ssh/match_ci_key` → Private key

- `~/.ssh/match_ci_key.pub` → Public key

2. Add the Private Key to the SSH Agent (optional but helpful)
   This step ensures the key is used during local development.

```yaml
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/match_ci_key
```

3. Add the Public Key to Your Certificates Repo (GitHub)

- Go to your certificates repo on GitHub (e.g., openMF/ios-provisioning-profile).
- Go to Settings → Deploy Keys.
- Click “Add deploy key”.
- Set title as `CI Match SSH Key` and paste the content of:

```yaml
cat ~/.ssh/match_ci_key.pub
```

- Check **Allow write access**.
- Click Add key.

4. Convert the Private Key to Base64
   This is how we pass it to GitHub Actions securely.

```yaml
base64 -i ~/.ssh/match_ci_key | pbcopy
```

This command copies the base64-encoded private key to your clipboard (macOS).

5. Save the Private Key as a GitHub Secret

- Go to the repo with your Fastfile (the project repo, not the certs repo).
- Navigate to Settings → Secrets and variables → Actions → New repository secret.
- Add the following:
    - Name: MATCH_SSH_PRIVATE_KEY
    - Value: Paste the base64-encoded private key

### Fastlane Setup

Create a `Gemfile` in your project root:

```ruby
source "https://rubygems.org"

gem "fastlane"
gem "firebase_app_distribution"
```

Create a `fastlane/Fastfile` with the following content:

```ruby
default_platform(:ios)

platform :ios do
    desc "Deploy iOS app to Firebase App Distribution"
    lane :deploy_on_firebase do |options|
        # === Variables at top ===
        firebase_app_id = options[:firebase_app_id] || "1:xxxxxx:ios:xxxxxxxx"
        match_type = options[:match_type] || "adhoc"
        app_identifier = options[:app_identifier] || "com.example.myapp"
        git_url = options[:git_url] || "git@github.com:openMF/ios-provisioning-profile.git"
        git_branch = options[:git_branch] || "master"
        match_password = options[:match_password] || "your-default-match-password"
        git_private_key = options[:git_private_key] || "./secrets/match_ci_key"
        groups = options[:tester_groups] || "qa-team,beta-testers"
        serviceCredsFile = options[:serviceCredsFile] || "secrets/firebaseAppDistributionServiceCredentialsFile.json"
        provisioning_profile_name = options[:provisioning_profile_name] || "match AdHoc com.example.myapp"
        appstore_key_id = options[:appstore_key_id] || "DEFAULT_KEY_ID"
        appstore_issuer_id = options[:appstore_issuer_id] || "DEFAULT_ISSUER_ID"
        key_filepath = options[:key_filepath] || "secrets/Auth_key.p8"
    
        # === Setup CI ===
        unless ENV['CI']
            UI.message("🖥️ Running locally, skipping CI-specific setup.")
        else
            setup_ci(provider: "circleci")
        end
    
        # === Load API Key ===
        app_store_connect_api_key(
            key_id: appstore_key_id,
            issuer_id: appstore_issuer_id,
            key_filepath: key_filepath,
            duration: 1200
        )
    
        # === Fetch Match certs ===
        match(
            type: match_type,
            app_identifier: app_identifier,
            readonly: false,
            git_url: git_url,
            git_branch: git_branch,
            git_private_key: git_private_key,
            force_for_new_devices: true,
            api_key: Actions.lane_context[SharedValues::APP_STORE_CONNECT_API_KEY]
        )
    
        # === Increment build number from Firebase ===
        latest_release = firebase_app_distribution_get_latest_release(
            app: firebase_app_id,
            service_credentials_file: serviceCredsFile
        )
    
        if latest_release
            increment_build_number(
                xcodeproj: "iosApp/iosApp.xcodeproj",
                build_number: latest_release[:buildVersion].to_i + 1
            )
        else
            UI.important("⚠️ No existing Firebase release found. Skipping build number increment.")
        end
    
        # === Build signed IPA ===
        build_ios_app(
            scheme: "iosApp",
            project: "iosApp/iosApp.xcodeproj",
            output_name: "DeployIosApp.ipa",
            output_directory: "iosApp/build",
            export_options: {
                provisioningProfiles: {
                    app_identifier => provisioning_profile_name
                }
            },
            xcargs: "CODE_SIGN_STYLE=Manual CODE_SIGN_IDENTITY=\"Apple Distribution\" DEVELOPMENT_TEAM=L432S2FZP5 PROVISIONING_PROFILE_SPECIFIER=\"#{provisioning_profile_name}\""
        )
    
        # === Generate release notes ===
        releaseNotes = changelog_from_git_commits(
            commits_count: 1
        )
    
        # === Upload to Firebase ===
        firebase_app_distribution(
            app: firebase_app_id,
            service_credentials_file: serviceCredsFile,
            release_notes: releaseNotes,
            groups: groups
        )
    end
end
```

## Usage

Add the following workflow to your GitHub Actions:

```yaml
name: Deploy iOS to Firebase

on:
  push:
    branches: [ main ]
  # or:
  workflow_dispatch:

jobs:
  deploy_ios:
    runs-on: macos-latest
    steps:
      - name: Set Xcode version 16.2
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: latest-stable

      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Publish iOS App to Firebase
        uses: openMF/kmp-publish-ios-on-firebase-action@v1.0.2
        with:
          app_identifier: 'com.example.myapp'
          git_url: 'git@github.com:openMF/ios-provisioning-profile.git'
          git_branch: 'master'
          match_type: 'adhoc'
          provisioning_profile_name: 'match AdHoc com.example.myapp'
          match_password: ${{ secrets.MATCH_PASSWORD }}
          match_ssh_private_key: ${{ secrets.MATCH_SSH_PRIVATE_KEY }}
          appstore_key_id: ${{ secrets.APPSTORE_KEY_ID }}
          appstore_issuer_id: ${{ secrets.APPSTORE_ISSUER_ID }}
          appstore_auth_key: ${{ secrets.APPSTORE_AUTH_KEY }}
          ios_package_name: 'iosApp'
          firebase_app_id: '1:xxxxxx:ios:xxxxxxxx'
          firebase_creds: ${{ secrets.FIREBASE_CREDS }}
          tester_groups: 'qa-team,beta-testers'
```

## Inputs and Secrets

| Input                       | Description                                         | Required |
|-----------------------------|-----------------------------------------------------|----------|
| `app_identifier`            | The unique bundle identifier for the iOS app        | ✅        |
| `git_url`                   | Git URL to certificates/profiles repo               | ✅        |
| `git_branch`                | Branch in certificates repo                         | ✅        |
| `match_type`                | Profile type (adhoc, appstore, development)         | ✅        |
| `provisioning_profile_name` | Name of provisioning profile                        | ✅        |
| `match_password`            | Password to decrypt Match certs                     | ✅        |
| `match_ssh_private_key`     | Base64-encoded SSH private key for Match            | ✅        |
| `appstore_key_id`           | App Store Connect API key ID                        | ✅        |
| `appstore_issuer_id`        | App Store Connect issuer ID                         | ✅        |
| `appstore_auth_key`         | Base64-encoded `.p8` API key                        | ✅        |
| `ios_package_name`          | Name of the iOS package/module (Xcode scheme)       | ✅        |
| `firebase_app_id`           | Firebase App ID                                     | ✅        |
| `firebase_creds`            | Base64 encoded Firebase service account credentials | ✅        |
| `tester_groups`             | Firebase tester groups (comma-separated)            | ✅        |

## Setting up Secrets

1. Encode your Firebase credentials file to base64:

```bash
base64 -i path/to/firebase-credentials.json -o firebase-creds.txt
```

2. Encode you r `.p8` API key to base64:
```bash
base64 -i path/to/AuthKey_XXXXXXXXXX.p8 | pbcopy 
```

3. Add the following secret to your GitHub repository:

- `FIREBASE_CREDS`: Content of firebase-creds.txt
- `APPSTORE_AUTH_KEY`: Content of AuthKey_XXXXXXXXXX.p8

## Build Artifacts

The action uploads the built IPA file as an artifact with:

- Name: 'ios-app'
- Retention period: 1 day
- Maximum compression (level 9)

You can find the IPA file in your GitHub Actions run artifacts.

This helps reduce build times in subsequent runs.

## Firebase App Setup

1. Go to the Firebase Console
2. Navigate to App Distribution
3. Create a new iOS app or select an existing one
4. Set up tester groups under App Distribution
5. Create a service account with appropriate permissions
6. Download the service account JSON key file

## Troubleshooting

Common issues and solutions:

1. Build fails due to signing
    - Ensure your Xcode project has proper signing configuration
    - Verify the export method matches your provisioning profile

2. Firebase deployment fails
    - Check if the service account has sufficient permissions
    - Verify the Firebase app ID is correct
    - Ensure tester groups exist in Firebase console
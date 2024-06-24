# CI/CD example for iOS 

A simple example of CI/CD for iOS. The example is basic and is made only to show how you can automate the **launch of unit tests** and sending build to **Test Flight** every time you **commit** to the **main** branch. 
The CI/CD of the project is built on [GitHub Actions](https://github.com/features/actions) and [Fastlane](https://fastlane.tools/).

> [!TIP]
> For more convenient work with files in the project, I recommend using [Visual Studio Code](https://code.visualstudio.com/).
>
> For CI/CD, it is advisable to create a **new account** for **github** and a new account for **appstore**.

### What is CI/CD?

Continuous Integration and Continuous Deployment (CI/CD) is a set of practices that aim to streamline the software development lifecycle.

### What is GitHub Actions?

GitHub Actions provides create custom workflows that automate various tasks, such as building, testing, and deploying code, directly from their GitHub repositories.

### Requirement for our CI/CD project:
1. Run Github Actions on every branch push.
2. Github Actions should run unit tests.
3. Github Actions should send the build to Test Flight after successfully passing unit tests.


### 1. Installing Fastlane

Fastlane can be installed in various other ways. For installation **alternatives**, consult the [official Fastlane documentation](https://docs.fastlane.tools/).

```ruby
brew install fastlane
```

After installation, **navigate** to the **iOS folder** of your project and initialize Fastlane:

```ruby
fastlane init
```

During the initialization process, when asked about how you want to configure Fastlane, choose **'Manual setup'**.
Upon completion, the initial structure of Fastlane will be created with the following files:

```
├── fastlane
  ├── Appfile
  └── Fastfile
└── Gemfile
```
> [!TIP]
> For more convenient work with files in the project, I recommend using [Visual Studio Code](https://code.visualstudio.com/).

- `Gemfile` - used by Bundler to manage **Ruby dependencies.**
- `Appfile` - contains **global settings** for your app.
- `Fastfile` - defines the **'lanes**' that automate **specific tasks**.

Let's create a file of **constants**. Then we can replace them ["Using secrets in GitHub Actions"](https://docs.github.com/ru/actions/security-guides/using-secrets-in-github-actions).

Create a new file named `.env.default` in "fastlane" folder. You should have something like this:

```
├── fastlane
  ├── Appfile
  ├── .env.default
  └── Fastfile
└── Gemfile
```
and add new value to the `env.default`
```ruby
APP_IDENTIFIER="nesterchuk.oleksii.CiCdExample"  # The bundle identifier of your app
APPLE_ID="nesalexy@gmail.com" # Your Apple email address
TEAM_NAME="My team name" # Appstore Team Name
TEAM_ID="L4******3D" # Appstore Team ID
```
Edit the `'Appfile'` to configure the bundle of your application:
```ruby
app_identifier "#{ENV["APP_IDENTIFIER"]}"
apple_id "#{ENV["APPLE_ID"]}"

team_name "#{ENV["TEAM_NAME"]}"
team_id "#{ENV["TEAM_ID"]}"
```
Update the `'Gemfile'` to specify the version of Fastlane used in the project:

### 2. Configuring Fastlane Match

Match is the implementation of the codesigning.guide concept. Match creates all required **certificates** & **provisioning profiles** and **store**s them in a separate **git repository**, **Google Cloud**, or **Amazon S3**.

**A lot of mistakes arise at this step**. Please read the [Fastlane Documentation](https://docs.fastlane.tools/actions/match/) or [Code Signing](https://codesigning.guide/) if you get errors.

**Before next step**:
1. Create a new app in [appstoreconnect]( https://appstoreconnect.apple.com/) with your app ID. For current project **nesterchuk.oleksii.CiCdExample**
2. **Сreate a separate git repository** where **certificates** & **provisioning profiles** will be stored. 
3. For **Match** to interact with the **GitHub repository**, you will need a **'Personal Access Token'**. [Here's how to create this token](https://docs.github.com/ru/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-personal-access-token-classic). After **creating the token**, convert it to **Base64** and copy the result as follows: 
```ruby
echo -n 'your_github_username:your_personal_access_token' | base64 | pbcopy
```
Add the **generated** key to `env.default`. It goes something like this
```ruby
MATCH_GIT_BASIC_AUTHORIZATION="bmVzY***eHk6Z2hwX3N0WWWZMV******KOFp0T2ZYVzZhV0d**********jFPTGh6aA=="
```

### Okay, now we can move on to installing Match

Make sure you are in the **root folder** of the project and write the command:

```ruby
fastlane match init
```

Choose the `'git'` option and provide the URL of the **newly created repository**, for example: 
```
https://github.com/iOS-CI-CD-Example/secrets.git
```

`'Matchfile'` will be created in the `'fastlane'` folder
```
├── fastlane
  ├── Appfile
  ├── .env.default
  ├── Matchfile
  └── Fastfile
└── Gemfile
```

**Generating Certificates and Provisioning Profiles**

Run the Match commands to generate the certificates:

- For **development**: `fastlane match development`
- For **distribution**: `fastlane match appstore`

Remember to create a password for the keychain when prompted and **keep it in a secure location**.
Check the 'secrets' repository to see if the certificates have been successfully created.

Add the **Your match keychain** key to `env.default`. It goes something like this
```ruby
MATCH_PASSWORD="cn123321"
```

**Toward the end of step "2. Configuring Fastlane Match"**, your **Matchfile** should look something like this
```ruby
git_url("https://github.com/iOS-CI-CD-Example/secrets.git")
storage_mode("git")
type("appstore") # The default type, can be: appstore, adhoc, enterprise or development
app_identifier(["#{ENV["APP_IDENTIFIER"]}"])
username "#{ENV["APPLE_ID"]}"
```
Your **env.default** should look something like this
```ruby
APP_IDENTIFIER="nesterchuk.oleksii.CiCdExample"  # The bundle identifier of your app
APPLE_ID="nesalexy@gmail.com" # Your Apple email address
TEAM_NAME="My team name" # Appstore Team Name
TEAM_ID="L4******3D" # Appstore Team ID
MATCH_GIT_BASIC_AUTHORIZATION="bmVzY***eHk6Z2hwX3N0WWZMV******KOFp0T2ZYVzZhV0d**********jFPTGh6aA=="
MATCH_PASSWORD="cn123321"
```

### 3. Configuring Fastfile

To be able to **run unit tests** and send the build to the **Test Flight**, we need to configure the commands - **lane(s)** in the **'Fastfile'**.

**Before next step, we need Generating the API Keys in App Store Connect**
1. Access your account on [App Store Connect](https://appstoreconnect.apple.com/).
2. Navigate to **'Users and Access'**.
3. Select the **'Keys'** tab and click on **'Generate API Key'**.
4. Download the **AuthKey_W********.p8** key and put to **fastlane folder**
```
├── fastlane
  ├── Appfile
  ├── AuthKey_W********A.p8
  ├── .env.default
  ├── Matchfile
  └── Fastfile
└── Gemfile
```

We need to modify file **env.default** and add constants.
```ruby
APP_STORE_API_PRIVATE_KEY="W********A"
APP_STORE_API_KEY_ISSUER_ID="caXXXX8d-XXXX-44ce-XXXX-63b7XXXX063a"
APP_AUTH_KEY_FILEPATH="./Fastlane/AuthKey_$APP_STORE_API_PRIVATE_KEY.p8"
APP_STORE_CONNECT_DURATION=1200
```
add additional constants
```ruby
APP_XCODEPROJ="nesterchuk.oleksii.CiCdExample.xcodeproj"
APP_STORE_CONNECT_DURATION=1200
APP_SCHEME="nesterchuk.oleksii.CiCdExample"
```
Modify `Fastfile` and add the code
```ruby
fastlane_version '2.221.0'
default_platform :ios

platform :ios do
  before_all do
    setup_ci
  end

  desc 'Builds project and executes unit tests'
  lane :tests do
    run_tests(workspace: "#{ENV['APP_XCODEPROJ']}/project.xcworkspace",
              devices: ["iPhone 15 Pro"],
              scheme: "#{ENV['APP_SCHEME']}")
  end

  lane :test_flight do
    api_key = app_store_connect_api_key(
      key_id: "#{ENV['APP_STORE_API_PRIVATE_KEY']}",
      issuer_id: "#{ENV['APP_STORE_API_KEY_ISSUER_ID']}",
      key_filepath: "#{ENV['APP_AUTH_KEY_FILEPATH']}",
      duration: "#{ENV['APP_STORE_CONNECT_DURATION']}" 
    )
    previous_build_number = latest_testflight_build_number(
      app_identifier:"#{ENV['APP_IDENTIFIER']}",
      api_key: api_key,
    )
    
    current_build_number = previous_build_number + 1
    increment_build_number(
      xcodeproj: "#{ENV['APP_XCODEPROJ']}",
      build_number: current_build_number
    )

    sync_code_signing(
      type: "appstore",
      readonly: true,
    )

    build_app(
      scheme: "#{ENV['APP_SCHEME']}",
      xcargs: "-allowProvisioningUpdates"
    )

    upload_to_testflight(
      skip_submission: true,
      skip_waiting_for_build_processing: true
    ) 
  end
end
```
### 4. Configuring Xcode

To prepare the automation of our app launch process for **TestFlight**, we performed some configurations in **Xcode**:

1. Automatically manage signing: In the **'Signing & Capabilities'** tab, we **disabled the 'Automatically manage signing'** option and selected the **'match AppStore nesterchuk.oleksii.CiCdExample'** provisioning profile.

### 5. Testing 

To continue, we need to test if everything is set up correctly. To do this, let's test locally. Make sure you are in the root folder of the ios project and run unit test 
```ruby
fastlane tests
```
If all went well, try submitting the build locally to Test Flight
```ruby
fastlane test_flight
```

### 6. Github Actions

Before next step, we need add 
1. `.gitignore` to root folder
2. `.ruby-version` to root folder
3. a new folder `.github` -> add a new folder `workflows` -> pullRequest.yml

```
├──.github
  ├── workflows
    ├── pullRequest.yml
├── fastlane
  ├── Appfile
  ├── AuthKey_W********A.p8
  ├── .env.default
  ├── Matchfile
  └── Fastfile
├──.gitignore
├──Gemfile
└──.ruby-version
```
Add code to `.gitignore`
```ruby
Build

# fastlane
Fastlane/report.xml
Fastlane/test_output

nesterchuk.oleksii.CiCdExample.app.dSYM.zip
nesterchuk.oleksii.CiCdExample.ipa
```
Add code to `.ruby-version`
```ruby
3.3
```
Add code to `pullRequest.yml`
```ruby
name: Pull Request

on:
  push:
    branches:
      - main
  workflow_dispatch:
  
jobs:
  CI:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cancel Previous Runs
        uses: styfle/cancel-workflow-action@0.9.1
        with:
          access_token: ${{ secrets.KEY }} # see https://docs.github.com/ru/actions/security-guides/using-secrets-in-github-actions
        
      - uses: maxim-lobanov/setup-xcode@v1
        id: setup-xcode
        with:
          xcode-version: latest-stable
        
      - name: Set up Ruby
        id: setup-ruby
        uses: ruby/setup-ruby@v1

      - name: Install Bundler
        id: install-bundler
        run: gem install bundler

      - name: Install gems
        id: install-gems
        run: bundle install

      - name: Swift Packages Cache
        id: swift-packages-cache
        uses: actions/cache@v2
        with:
          path: |
            Build/SourcePackages
            Build/Build/Products
          key: ${{ runner.os }}-deps-v1-${{ hashFiles('BILDsolid.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved') }}
          restore-keys: ${{ runner.os }}-deps-v1-

      - name: Launch Simulator
        uses: futureware-tech/simulator-action@v3
        with:
          model: 'iPhone 15 Pro'
          os: 'iOS'
          os_version: '17.2'

      - name: Run Tests (No Cache)
        if: steps.setup.outputs.cache-hit != 'true'
        run: bundle exec fastlane tests
      
      - name: Run Tests (Cache)
        if: steps.setup.outputs.cache-hit == 'true'
        run: bundle exec fastlane tests skip_package_dependencies_resolution:true
  CD:
    needs: CI
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4

      - name: Cancel Previous Runs
        uses: styfle/cancel-workflow-action@0.9.1
        with:
          access_token: ${{ secrets.KEY }}
        
      - uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: latest-stable
        
      - uses: ruby/setup-ruby@v1

      - name: Install Bundler
        run: gem install bundler

      - name: Install gems
        run: bundle install
      
      - name: Send to TF
        run: bundle exec fastlane test_flight
```

If you have configured everything correctly, then when you push any changes to the main branch, the tests will be run, then the build will be assembled and sent to Test Flight.

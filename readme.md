# Building Android Applications with AWS Amplify

In this workshop we'll learn how to build cloud-enabled mobile applications with Andriod and [AWS Amplify]().

### Topics we'll be covering:

- [Authentication](#adding-authentication)
- [GraphQL API and AWS AppSync](#adding-a-graphql-api)
- [Adding Analytics](#adding-analytics)
- [Testing Applications with Device Farm](#testing-your-app)

## Getting Started - Creating the Android Studio Project

To get started, we first need to create a new Andriod project.

If you already have a project set up, skip to the next step. Otherwise, in Android Studio:

- select `File > New > New Project`
- choose `Empty Activity`
- choose `Java` for application type and take note of where the project will be saved to (we'll need this information in the next step
- choose `Finish`

Now it's time to build and run the application.

## Installing the CLI & Initializing a new AWS Amplify Project

### Installing the CLI

Next, we'll install the AWS Amplify CLI:

```bash
npm install -g @aws-amplify/cli
```

Now we need to configure the CLI with our credentials:

```js
amplify configure
```

> If you'd like to see a video walkthrough of this configuration process, click [here](https://www.youtube.com/watch?v=fWbM5DLh25U).

Here we'll walk through the `amplify configure` setup. Once you've signed in to the AWS console, continue:

- Specify the AWS Region: **us-east-1**
- Specify the username of the new IAM user: **amplify-workshop-user**
  > In the AWS Console, click **Next: Permissions**, **Next: Tags**, **Next: Review**, & **Create User** to create the new IAM user. Then, return to the command line & press Enter.
- Enter the access key of the newly created user:  
  ? accessKeyId: **(<YOUR_ACCESS_KEY_ID>)**  
  ? secretAccessKey: **(<YOUR_SECRET_ACCESS_KEY>)**
- Profile Name: **amplify-workshop-user**

### Initializing A New Project

```bash
amplify init
```

- Enter a name for the project: **amplifyandroidapp**
- Enter a name for the environment: **master**
- Choose your default editor: **Visual Studio Code (or your default editor)**
- Please choose the type of app that you're building: **android**
- Where is your Res directory? **(app/src/main/res)**
- Do you want to use an AWS profile? **Y**
- Please choose the profile you want to use: **amplify-workshop-user**

```bash
amplify push
```

Now an `awsconfiguration.json` file has be created with your configuration and will be updated as features get added to your project by the Amplify CLI. The file is placed in the `./app/src/main/res/raw` directory of your Android Studio project and automatically used by the SDKs at runtime.

To view the status of the amplify project at any time, you can run the Amplify `status` command:

```sh
amplify status
```

## Adding Authentication

### Adding a Cognito User Pool

```sh
amplify add auth #accept default configuration
amplify push
```

### Setting Up the Dependencies

First add the following dependencies to your App `build.gradle`

```gradle
//For AWSMobileClient only:
implementation 'com.amazonaws:aws-android-sdk-mobile-client:2.13.+'

//For the drop-in UI also:
implementation 'com.amazonaws:aws-android-sdk-auth-userpools:2.13.+'
implementation 'com.amazonaws:aws-android-sdk-auth-ui:2.13.+'
```

You will also need to set the `minSdkVersion` to `23` for the drop-in UI.

```gradle
minSdkVersion 23
```

Make sure the following permissions are enabled in `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

### Updating the Main Activity

`AWSMobileClient` ships with a drop-in Activity for handling sign in and sign up functionality. Replace `MainActivity.java` with the following:

```java
package com.example.amplifyworkshopnyc;

import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;

import com.amazonaws.mobile.client.AWSMobileClient;
import com.amazonaws.mobile.client.Callback;
import com.amazonaws.mobile.client.SignInUIOptions;
import com.amazonaws.mobile.client.UserStateDetails;

public class MainActivity extends AppCompatActivity {

    // tag for logs
    private static final String TAG = "MainActivity";

    // instance of mobile client
    private final AWSMobileClient CLIENT = AWSMobileClient.getInstance();

    // launches auth UI
    protected void showSignIn() {
        CLIENT.showSignIn(
                this,
                SignInUIOptions.builder()
                        .nextActivity(MainActivity.class)
                        .build(),
                new Callback<UserStateDetails>() {
                    @Override
                    public void onResult(UserStateDetails result) {
                        Log.d(TAG, "onResult: " + result.getUserState());
                        switch (result.getUserState()) {
                            case SIGNED_IN:
                                Log.i("INIT", "logged in!");
                                break;
                            case SIGNED_OUT:
                                Log.i(TAG, "onResult: User did not choose to sign-in");
                                break;
                            default:
                                AWSMobileClient.getInstance().signOut();
                                break;
                        }
                    }

                    @Override
                    public void onError(Exception e) {
                        Log.e(TAG, "onError: ", e);
                    }
                }
        );
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        CLIENT.initialize(getApplicationContext(), new Callback<UserStateDetails>() {
            @Override
            public void onResult(UserStateDetails userStateDetails) {
                switch (userStateDetails.getUserState()) {
                    case SIGNED_IN:
                        Log.d(TAG, "user signed in: " + userStateDetails.getUserState());
                        break;
                    case SIGNED_OUT:
                        showSignIn();
                        break;
                    default:
                        CLIENT.signOut();
                        break;
                }
            }

            @Override
            public void onError(Exception e) {
                Log.e("INIT", "Initialization error.", e);
            }
        });
    }
}

```

Build and run the app and you should see the drop-in UI for authentication.

## Adding a GraphQL API

### Setting Up AppSync

```bash
amplify add api
```

- Please select from one of the below mentioned services: **GraphQL**
- Provide API name: **nycworkshop**
- Choose an authorization type for the API: **Amazon Cognito User Pool**
- Do you have an annotated GraphQL schema?: **N**
- Do you want a guided schema creation? **Y**
- What best describes your project: **Single object with fields (e.g., “Todo” with ID, name, description)**
- Do you want to edit the schema now?: **Y**

Update `schema.graphql` with the following:

```graphql
type Todo @model @auth(rules: [{ allow: owner }]) {
  id: ID!
  name: String!
  description: String
}
```

```bash
amplify push
```

At this point the CLI will prompt you with the following:

- Do you want to generate code for your newly created GraphQL API: **Y**
- Enter the file name pattern of graphql queries, mutations and subscriptions: **(app/src/main/graphql/\*\*/\*.graphql)**
- Do you want to generate/update all possible GraphQL operations - queries, mutations and subscriptions: **Y**
- Enter maximum statement depth [increase from default if your schema is deeply nested]: **2**

### Adding Dependencies

Next you'll need to modify your `project/build.gradle` with the following build dependency:

```gradle
classpath 'com.amazonaws:aws-android-sdk-appsync-gradle-plugin:2.8.+'
```

Next, add dependencies to your `app/build.gradle`, and then choose `Sync Now` on the upper-right side of Android Studio.

```gradle
apply plugin: 'com.amazonaws.appsync'

dependencies {
    //Base SDK
    implementation 'com.amazonaws:aws-android-sdk-core:2.13.+'
    //AppSync SDK
    implementation 'com.amazonaws:aws-android-sdk-appsync:2.8.+'
    implementation 'org.eclipse.paho:org.eclipse.paho.client.mqttv3:1.2.0'
    implementation 'org.eclipse.paho:org.eclipse.paho.android.service:1.1.1'
}

```

Finally, update your `AndroidManifest.xml` with the following:

```xml
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>

        <!--other code-->

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">

        <!--add this service-->
        <service android:name="org.eclipse.paho.android.service.MqttService" />

        <!--other code-->
    </application>
```

The next step is to initialize the AppSync mobile client in the `SIGN_IN` switch case in `MainActivity.java`.

```java
// create AppSync client
mAWSAppSyncClient = AWSAppSyncClient.builder()
        .context(getApplicationContext())
        .awsConfiguration(new AWSConfiguration(getApplicationContext()))
        // add our user pool provider
        .cognitoUserPoolsAuthProvider(new CognitoUserPoolsAuthProvider() {
            @Override
            public String getLatestAuthToken() {
                try {
                    return mAWSMobileClient.getTokens().getIdToken().getTokenString();
                } catch (Exception e){
                    Log.e("APPSYNC_ERROR", e.getLocalizedMessage());
                    return e.getLocalizedMessage();
                }
            }
        }).build();
```

Next add the following class methods and properties:

```java
// used for subscriptions
private AppSyncSubscriptionCall subscriptionWatcher;

// query, mutation, and subscription methods and callbacks
public void query(){
    mAWSAppSyncClient.query(ListTodosQuery.builder().build())
        .responseFetcher(AppSyncResponseFetchers.CACHE_AND_NETWORK)
        .enqueue(todosCallback);
}

private GraphQLCall.Callback<ListTodosQuery.Data> todosCallback = new GraphQLCall.Callback<ListTodosQuery.Data>() {
    @Override
    public void onResponse(@Nonnull Response<ListTodosQuery.Data> response) {
        Log.i("API", response.data().listTodos().items().toString());
    }

    @Override
    public void onFailure(@Nonnull ApolloException e) {
        Log.e("ERROR", e.toString());
    }
};

public void mutation(){
    CreateTodoInput createTodoInput = CreateTodoInput.builder()
        .name("Use AppSync")
        .description("Realtime and Offline")
        .build();

    mAWSAppSyncClient.mutate(CreateTodoMutation.builder().input(createTodoInput).build())
        .enqueue(mutationCallback);
}

private GraphQLCall.Callback<CreateTodoMutation.Data> mutationCallback = new GraphQLCall.Callback<CreateTodoMutation.Data>() {
    @Override
    public void onResponse(@Nonnull Response<CreateTodoMutation.Data> response) {
        Log.i("API", "Added Todo - ", response.data.toString());
        query();
    }

    @Override
    public void onFailure(@Nonnull ApolloException e) {
        Log.e("Error", e.toString());
    }
};

```

Next, call the `mutation` method inside of the successful login callback:

```java
case SIGNED_IN:
Log.d(TAG, "user signed in: " + userStateDetails.getUserState());

// create AppSync client
mAWSAppSyncClient = AWSAppSyncClient.builder()
        .context(getApplicationContext())
        .awsConfiguration(new AWSConfiguration(getApplicationContext()))
        .cognitoUserPoolsAuthProvider(new CognitoUserPoolsAuthProvider() {
            @Override
            public String getLatestAuthToken() {
                try {
                    return mAWSMobileClient.getTokens().getIdToken().getTokenString();
                } catch (Exception e){
                    Log.e("APPSYNC_ERROR", e.getLocalizedMessage());
                    return e.getLocalizedMessage();
                }
            }
        }).build();

mutation();
break;
```

## Adding Analytics

### Setting Up Pinpoint

```bash
amplify add analytics
```

- Provide your pinpoint resource name: **(nycworkhop)**
- Apps need authorization to send analytics events. Do you want to allow guests and unauthenticated users to send analytics events? (we recommend you allow this when getting started): **Y**

```bash
amplify push
```

### Adding Dependencies

Add the following to your App `build.gradle`

```gradle
implementation 'com.amazonaws:aws-android-sdk-pinpoint:2.13.+'
implementation ('com.amazonaws:aws-android-sdk-mobile-client:2.13.+@aar') { transitive = true }
```

Next add the following properties to `MainActivity.java`:

```java
public static PinpointManager pinpointManager;

public static PinpointManager getPinpointManager(final Context applicationContext) {
    if (pinpointManager == null) {
        // Initialize the AWS Mobile Client
        final AWSConfiguration awsConfig = new AWSConfiguration(applicationContext);
        AWSMobileClient.getInstance().initialize(applicationContext, awsConfig, new Callback<UserStateDetails>() {
            @Override
            public void onResult(UserStateDetails userStateDetails) {
                Log.i("PINPOINT", userStateDetails.getUserState());
            }

            @Override
            public void onError(Exception e) {
                Log.e("PINPOINT", "Initialization error.", e);
            }
        });

        PinpointConfiguration pinpointConfig = new PinpointConfiguration(
                applicationContext,
                AWSMobileClient.getInstance(),
                awsConfig);

        pinpointManager = new PinpointManager(pinpointConfig);
    }
    return pinpointManager;
}
```

Then in `onCreate` add the following:

```java
final PinpointManager pinpointManager = getPinpointManager(getApplicationContext());
pinpointManager.getSessionClient().startSession();
```

Next we need to record a our session ending in the `onDestroy` method, add the following to method to `MainActivity.java`:

```java
@Override
protected void onDestroy() {
    super.onDestroy();

    pinpointManager.getSessionClient().stopSession();
    pinpointManager.getAnalyticsClient().submitEvents();
}
```

## Testing Your App

If you're curious about how to test your Android apps without having to write any code. [Follow this tutorial](https://dev.to/kkemple/how-to-set-up-end-to-end-tests-for-android-with-zero-code-1ka).

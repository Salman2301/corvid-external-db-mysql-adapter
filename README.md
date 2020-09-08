<h2 align="center">
  Corvid by Wix DB Connectors: External Database Adapter for MySQL
</h1>

This project is a Node.js based adapter that lets you integrate an external MySQL database with your Corvid enabled Wix site.

You can use this project as a basis for deploying your own adapter to the Google Cloud AppEngine. We also provide a ready to deploy public Docker Image which can be hosted on [Google Cloud Run](https://cloud.google.com/run/), [Amazon Fargate](https://aws.amazon.com/fargate/) or [Microsoft Azure App Services](https://azure.microsoft.com/en-us/free/apps/). This project contains a basic implementation of the external database collection SPI that has filtering, authorization, and error handling support.

Learn more about working with external database collections in Corvid [here](https://support.wix.com/en/corvid-by-wix/external-database-collections-1023416).

See our [SPI documentation](https://www.wix.com/corvid/reference/spis/getting-started/overview).

# Getting started

This project assumes you have a Google Cloud Project with billing enabled. If you don't have one, follow this [free trial guide](https://cloud.google.com/free/).

Follow this three steps to connect MySQL DB with Wix External Database
  1. Create a Cloud SQL instance and database
  2. Deploy adapter
      1. Deploying to Cloud Run using Cloud Console
      2. Deploying to Google Cloud Run using the gcloud CLI
      3. Deploying to Google App Engine using gcloud CLI
  3. Connecting MySQL to your Corvid enabled Wix site

## 1. Create a Cloud SQL instance and database
Go to https://cloud.google.com/sql/docs/mysql/quickstart and create a database and tables with your required schema.

### Configuration

The configuration values are exposed via environment variables. There are three required configuration values:

- `SECRET_KEY`: The secret key that you will use when connecting the adapter in the Wix Editor. Each request to your adapter will contain this value as `secretKey` in the _requestContext_ key inside the payload.
- `ALLOWED_OPERATIONS`: The list of all the operations that this adapter will be allowed to perform. For example, if you want to create an adapter that allows read-only access, you can limit these operations to `["get", "find", "count"]`.
- `SQL_CONFIG`: The configuration that will be used to connect to your SQL instance. This is JSON string that will be passed the mysql.createConnection() function. All available configuration options are documented in the [mysqljs/mysql](https://github.com/mysqljs/mysql#connection-options) driver repository. SQL_CONFIG variable must be in form of 
``` {"socketPath":"/cloudsql/[YOUR_SQL_INSTANCE]", "user": "[YOUR_USER]", "password":"[YOUR_PASSWORD]", "database":"DATABASE_YOU_CREATED"} ```
Cloud SQL instance can be obtained as described [here](https://cloud.google.com/sql/docs/mysql/connect-run#nodejs).


## 2.i Deploying to Cloud Run using Cloud Console

Follow the instructions in the Google Cloud Run [Quickstart Guide](https://cloud.google.com/run/docs/quickstarts/prebuilt-deploy).

In the **Create service form** use the following settings:

1. **Container Image**: Use **gcr.io/corvid-api/mysql-connector-node**.
2. **Deployment Platform**: Select **Cloud Run (fully managed)**.
3. **Location**: Select **us-east1** region.
4. **Authentication**: Select **Allow unauthenticated invocations**. This enables access to the connector from Corvid.
5. **Show Optional Settings**: Add an **Environment variables** as described in the [Configuration](#configuration) section.
6. Click **Create** to deploy the image to Google Cloud Run and wait for the deployment to finish.

Click the displayed URL link to test the deployed connector.
In the browser you should see the following, which indicates that the connector is running:
> {"message":"Missing request context"}

Copy the service URL. You will need it to [connect MySQL to your Corvid enabled Wix site](#connecting-mysql-to-your-corvid-enabled-wix-site).

## 2.ii Deploying to Google Cloud Run using the gcloud CLI

1. Install the [Google Cloud SDK](https://cloud.google.com/sdk/docs/quickstarts) for your operating system.
2. [Acquire new user credentials to use for Application Default Credentials](https://cloud.google.com/sdk/gcloud/reference/auth/application-default/login):

    ```bash
    gcloud auth application-default login
    ```

3. [Set your project as the default](https://cloud.google.com/sdk/gcloud/reference/config/set):

    ```bash
    gcloud config set project PROJECTID
    ```

4. [Deploy the Corvid connector container](https://cloud.google.com/sdk/gcloud/reference/run/deploy):

    ```bash
    gcloud run deploy --image gcr.io/corvid-api/mysql-connector-node --platform managed --region us-east1 --set-env-vars SECRET_KEY=[YOUR SECRET KEY],SQL_CONFIG={YOUR MYSQL CONNECTION STRING},ALLOWED_OPERATIONS=["get", "find", "count", ...]
    ```

    a. At the prompt: **Allow unauthenticated invocations to [mysql-connector-node] (Y/N)?** Choose "Y".

    b. When deployment is successful, the command line output will contain something similar to this:

    > Service [mysql-connector-node] revision [mysql-connector-node-00001-nep] has been deployed and is serving 100 percent of traffic at <https://mysql-connector-node-[autogenerated].run.app>

Copy the service URL. You will need it to [connect MySQL to your Corvid enabled Wix site](#connecting-mysql-to-your-corvid-enabled-wix-site).

## 2.iii Deploying to Google App Engine using gcloud CLI

1. Check out the source code:

  ```bash
   git clone https://github.com/wix/corvid-external-db-mysql-adapter.git
   cd corvid-external-db-mysql-adapter
  ```

2. [Deploy the local code and/or configuration of your app to App Engine](https://cloud.google.com/sdk/gcloud/reference/app/deploy):

  ```bash
  gcloud app deploy --project <your project id>
  ```

3. After deployment, access your service at `https://<project id>.appspot.com/`

### Instance Size

The default [AppEngine instance class](https://cloud.google.com/appengine/docs/standard/#instance_classes) is F1. It works well for small tables of several gigabytes. If your application requires a larger capacity and executing complex and large queries, you can adjust the [instance size](https://cloud.google.com/appengine/docs/standard/#instance_classes) in the app.yaml file located at the root of the project. Please follow the instructions [here](https://cloud.google.com/appengine/docs/standard/nodejs/config/appref). Also, check the pricing in the [Google AppEngine Pricing](https://cloud.google.com/appengine/pricing) page.

## 3. Connecting MySQL to your Corvid enabled Wix site

Follow the [instructions here](https://support.wix.com/en/article/corvid-adding-and-deleting-an-external-database-collection).

In the connection Dialog settings use the following:

* **Add an endpoint URL**: Use the **connector service URL** from steps above.
* **Configuration**: Use: {"secretKey":<**Your secret key from the deployment step**>}

You should now see MySQL tables as collections in the Databases sections of the sidebar in your site.

# Developing and extending the adapter

The following steps describe setting up the environment for developing the adapter for Google Cloud SQL.

1. Check out the source code:

  ```bash
   git clone https://github.com/wix/corvid-external-db-mysql-adapter.git
   cd corvid-external-db-mysql-adapter
  ```

2. Create a [GCP Service account](https://cloud.google.com/iam/docs/service-accounts):

  ```bash
  gcloud iam service-accounts create corvid --description Corvid Dev account --display-name corvid-dev
  ```

3. Run the following command and copy its output, which you will use in step 4:

  ```bash
  gcloud iam service-accounts list
  ```

4. Generate the Service Account key and save it in gcp-sa-key.json:

  ```bash
  gcloud iam service-accounts keys create gcp-sa-key.json --iam-account <Service account name from Step 3>
  ```

5. Set an environment variable for GCP client libraries:

  ```bash
  export GOOGLE_APPLICATION_CREDENTIALS=$PWD/gcp-sa-key.json
  ```

6. Set up local MySQL proxy as described [here](https://github.com/GoogleCloudPlatform/nodejs-docs-samples/tree/master/cloud-sql/mysql/mysql#running-locally)

7. Use the following command to start the Connector. It is a NodeJS express application:

  ```bash
  npm install

  npm start
  ```

## Extensions

### Schemas

The schemas are loaded dynamically from the configured database.

Currently, the driver supports these basic MySQL datatypes:
* `varchar`,
* `text`,
* `decimal`,
* `bigint`,
* `int`,
* `tinyint`,
* `time`,
* `date`,
* `datetime`,
* `json`.

For all other datatypes, it defaults to the Wix Data `object` datatype.

This support can be extended by implementing additional handlers for required datatypes in the `service/support/table-converter` module.

### Authentication

Currently, the driver has authentication in the form of a _secret key_ (described in [Configuration](#configuration), above).

The secret key gets deployed together with the adapter to Google AppEngine. Every request made to the adapter is then verified against the secret key.

The authentication functionality can be further extended by modifying the `utils/auth-middleware` module.

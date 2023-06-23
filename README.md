# Development & Deployment

Collections Hub Platform

## Structure

`.
├── README.md
├── api # Contains lambda handlers used by the api gateway
├── cantaloupe # Contains a cantaloupe docker image, along with configuration files for cantaloupe
├── elasticsearch # Contains an elasticsearch docker image, along with configuration and bootstrap script
├── frontend # contains the Next.js frontend application, consuming the API
└── infra # contains the CDK stacks used for deploying the API and NexjJS application`

## API

The API subdirectory contains the AWS lambda handlers used by the API.

Each lambda handler is contained in a script at the top level of the directory.

### Testing lambda handlers

Tests are contained within `api/__tests__`.

To test the lambda handlers:

`cd api
npm run test`

## Cantaloupe

[Cantaloupe](https://cantaloupe-project.github.io/)
 image server currently runs on an ec2 instance as it does not appear to
 be possible to access cross region s3 buckets when running it in a 
container.
However, a Docker image is included in the directory for future 
reference.

### Configuration

Cantaloupe is automatically installed and configured as a 
service upon creation of the ec2 instance. The Cantaloupe service can be
 configured by altering the following files prior to deployment:

- cantaloupe/cantaloupe.service - the Cantaloupe service definition file
- cantaloupe/cantaloupe.sh - the script that launches the Cantaloupe jar

Cantaloupe can be configured by altering the following file prior to deployment:

- cantaloupe/cantaloupe.properties

### Accessing the ec2 instance

Access to the Cantaloupe ec2 instance is by SSM only. No access is provided by ssh.

## Elasticsearch

Elasticsearch runs on a docker container inside an AWS 
Fargate service. The image for the docker container is included in the 
elasticsearch directory.
This container will restore the latest snapshot to the cluster. A script
 called restoreSnapshot.sh is also available to restore an alternative 
snapshot.
Access the container to do this as described below.

### Configuration

The elasticsearch instance on the docker container can be configured by altering the elasticsearch.yml configuration file.

### Accessing the container

After the pipeline has finished running (see below) a 
container will be created within the Fargate ECS. You can use SSM in 
order to start a shell session on the container.

In order to do this, locate the task id of the task which 
can be found under ECS in the AWS console, and replace it in the 
following command. (authentication credentials may need to be entered 
first)

`aws ecs execute-command --cluster <name-based-on-branch>-cluster --task <task id> --container <name-based-on-branch>-ElasticsearchContainer --interactive --command "/bin/sh"`

### Restoring data

A data bootstrapping script is included with the 
elastisearch image. This will run automatically to load the latest 
snapshot.
The SNAPSHOT_BUCKET and SNAPSHOT_FOLDER values at the start of the 
script can be changed depending on the location of the snapshot
repository you wish to use.

The script will automatically be run as part of the 
creation of the container, so there is no need to manually intervene 
with data restoration.

In order to to restore a different snapshot, create a shell session on the container as described above then run the following:

`bash ./bootstrap.sh`

### Data permanence

The elasticsearch instances are stored on EFS volumes. 
This means that they will persist even if the Fargate cluster stack is 
destroyed. Destroying the EFS stack will mean the data is lost.

### Data and local setup files

Data for the elasticsearch cluster is stored in the `elasticsearch/data/`
 folder to allow for persistance. If an index exists in the data/ 
folder, the elasticsearch cluster will run with this when started. If 
there is no data, the latest snapshot from the bucket and snapshot 
folder defined at the top of the bootstrap.sh script will be loaded. *These values should be changed if you are loading from a different repository*.

If you wish to overwrite an existing index, either delete 
the contents of the data folder, or set the DROP_EXISTING_INDEX 
environment variable in the docker-compose.yml file to 'true'.

The docker-compose yml config in the elasticsearch 
directory also supplies the correct options for running the database 
including creating a bridge network, "local-testing-bridge", that must 
be supplied to SAM when invoking lambdas or the API.

## Frontend

### Setup Frontend

`cd frontend
npm install`

### Setting up environment variables in the frontend

Before using SAM api, please make sure to:

- Go to the frontend directory with: `cd frontend`
- Create two new files, .env.development and .env.production.
- For .env.development, add in: NEXT_PUBLIC_API_LINK=[http://localhost:3000/](http://localhost:3000/)
- For .env.production, add in : NEXT_PUBLIC_API_LINK=/

Note - Creating these files are a one off that will not need to be repeated again.

### Test Frontend

`npm run test`

Mock data within the frontend/pages/api directory is used for these tests, ensure that these are up to date when changes to the app are made.

### Run Frontend Locally

To run the frontend using the mock data in the directory frontend/pages/api, simply run:

`npm run build`

`npm run dev` (this works on localhost:3000)

You can also run the frontend locally using real data by running elasticsearch in a docker container and using SAM to emulate the API. To do this, see section on **local development** below.

### Note

You do not need to build the frontend if you run [set-local-testing.sh](http://set-local-testing.sh) or [predeploy.sh](http://predeploy.sh) in the infra folder and deploying the app, as these scripts will do the build for you.

## Infrastructure

### Setup

`cd infra
npm install`

### CI/CD

### Deploying the pipeline/ stacks

Code in infra/bin/infra.ts uses the branch name that you 
are currently using to decide whether to deploy a pipeline or the stacks
 outlined in that file.

If a pipeline already exists for the git branch you are 
in, you will prompt a redeployment by completing a pull request into the
 branch, or pushing into the branch.

Otherwise:

`aws configure sso` (if you haven’t already)

`cd infra
source ./predeploy.sh
cdk synth
cdk deploy --all --require-approval never --profile <profile name>`

Notes:

- Remove the --require-approval-never flag if you want to
 go over the security changes.
- If you aren't in a branch that would trigger a pipeline and you want 
to deploy specific stacks only, synth and then replace the --all flag 
with the specific stack names when you deploy
- For all but the main branch, a subdomain for your branch 
is prefixed to the domain name and this can be seen in CloudWatch under 
alternate domain names.

### Destroying CDK stacks

To destroy the stacks:

`cd infra
cdk destroy --all`

### Destroying CDK pipeline & resources

If you are on a branch which triggered a pipeline, you can run `cdk destroy` to destroy the pipeline stack but destroying the pipeline will not destroy the other stacks and resources that have been created by the pipeline; these must be deleted one by one in the AWS console.

# Local development

When you are developing locally there are ways to emulate different parts of the system.

1. The frontend only, with mocked data.
2. Elasticsearch can be run within a docker container.
3. The API can be emulated with SAM, with the data served by elasticsearch in the container.
4. The frontend can be run locally with real data by pointing it at the SAM emulated API.
5. Some aspects will not work such as the retrieval of the images in the viewer page, but search results and metadata should be able to be retrieved.

### Run the Frontend against mocked data

This is using the mock data in the directory frontend/pages/api. From the frontend directory simply run:

`npm run build`

`npm run dev` (this works on localhost:3000)

### Running Elasticsearch in a container

1. You need to set up the environmental variables for authorisation. 
    
    The PLATFORM variable can be set by running the following command:
    
    `cd elasticsearch
    source bin/set-env-variables.sh`
    
    To set the PLATFORM value manually:
    
    `export PLATFORM=linux/x86-64 # for linux or
    export PLATFORM=linux/arm64/v8 # for MacOS`
    
    Set the AWS tokens manually by logging into the AWS console and on the first page select the 'Command line or programmatic access' link.
     Use Option 1 to place the environment variables into your clipboard and
     paste into the console you are using.
    
    These can be checked with :
    
    `echo $AWS_ACCESS_KEY_ID
    echo $AWS_SECRET_ACCESS_KEY
    echo $AWS_SESSION_TOKEN
    echo $PLATFORM`
    
2. Initiate the container: 
    
    `cd /elasticsearch
    docker-compose up --build`
    
    or use
    
    `docker-compose up --build --detach`
    
    to run in the background.
    
3. Stop the container with a Ctrl-C or with
    
    `docker-compose down`
    

**Note** - See the Elasticsearch section above for more information about the data stored locally and the container setup files, especially useful to restore a different data snapshot.

### Using SAM for the API

1. The SAM CLI needs installing globally if it isn’t already.
2. Use `aws configure sso` if you haven’t already.
3. Setup and synth the CDK app (use the SSO profile name):
    
    `cd infra`
    
    `source./set-local-testing.sh`
    
    `cdk synth --profile <profilename>`
    
4. Now you can start up SAM (ensure you enter the formatted branch name e.g. feature-hnp-390 used for the stack naming):
    
    `sam local start-api -t cdk.out/<branch name>-CollectionsHubApiStack.template.json -n env.json --container-host-interface "0.0.0.0" --docker-network "local-bridge" --warm-containers "EAGER”`
    
    (If necessary use '--force-image-build' switch with above commands when making changes to an API lambda)
    

### Run the frontend against SAM

The above should have set up the api running via sam on [localhost:3000](http://localhost:3000) with the CORS headers allowing cross origin requests, and now frontend can be run on 3001 and it should call the API running on SAM.

- `cd frontend`
- `PORT=3001 npm run dev`

### Running other tests against SAM:

The cross search interface can then be called with e.g.:

`curl -XGET "http://localhost:3000/api/xsearch?field=title&terms=history&from=0"`

The BL archive search can be called with e.g.:

`curl -XGET "http://localhost:3000/api/search/bl?field=title&terms=history&from=0"`

The manifest interface can be called with e.g.:

`curl -XGET "http://localhost:3000/api/metadata/iiif/3/manifest/bl-001748638" --ignore-content-length`

'--ignore-content-length' seems necessary when used with curl.

The media interface uses a 307 redirect with a 'temporary signed URL' for the S3 bucket object. SAM instead supplies a mocked url.
The curl switch -L would normally be required to follow the redirect, but this will fail locally since the url is mocked.
The supplied url is displayed in the console output together with the value of the image filepath, however.

The media interface can be called in two ways: with a 
single volume identifier where the media is associated with a volume,
or with a volume and page identifier where the media is associated with a
 page and where the volume id is that of the page parent.

`curl -XGET "http://localhost:3000/api/media/thumbnail/bl-001748638" -L
curl -XGET "http://localhost:3000/api/media/thumbnail/bl-000268823-01/bl-000268823-01-619461-15" -L`

The zoomable image API is not yet operational until a cantaloupe server is configured in a local Docker container

## Issues

Issue templates are available for submitting bug and feature requests.

## Making pull requests

A template is provided when creating new pull requests. 
Please, fill out all information where possible. If not all of the 
sections are relevant to the change, or where there is other pertinent 
information, please add supporting information to 'notes'.

## Authentication

Authentication is currently handled using a Lamda@Edge 
function mediating access to the application endpoints and a 
pre-configured AWS Cognito User Pool.

The user pool can be managed within the AWS Console, and 
can be used for multiple deployments. In order to enable the userpool 
for a new deployment it is necessary to update the user pool app client 
callback urls, to add the subdomain for the new deployment (e.g. 
feature-hnp-xxx.htja.co.uk). It will be necessarily to manually delete 
these once a feature or release has been completed. Unfortunately it is 
not possible to add pattern matched URLs, however some domains and sub 
domains will be permanently configured (e.g. htja.co.uk, 
develop.htja.co.uk).

In the future this system will be replaced by Jisc Login.

To add an extra layer of security during development a WAF
 is also used which includes an IP access list, meaning it is necessary 
to be logged into the Jisc VPN in order to access the application.
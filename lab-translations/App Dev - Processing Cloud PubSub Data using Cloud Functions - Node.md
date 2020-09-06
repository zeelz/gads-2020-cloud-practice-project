# App Dev - Processing Cloud Pub/Sub Data using Cloud Functions: Node.js


## Sign in to the Google Cloud Platform (GCP) Console

On the Google Cloud Platform top-right menu, click Activate Cloud Shell icon to open Cloud Shell.

*Since we would be using only Cloud Shell for this lab you might as well click the Open in new window icon at the top right of Cloud Shell window to open it in a new tab.*


## Preparing the case study application

*In this section, you access Cloud Shell, clone the git repository containing the Quiz application, configure environment variables, and run the application.*

### Clone source code in Cloud Shell

* To clone the repository for the class, enter the following command:

```
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```

* Create a soft link as a shortcut to your working directory:

```
ln -s ~/training-data-analyst/courses/developingapps/v1.2/nodejs/cloudfunctions ~/cloudfunctions
```


### Configure and run the case study application

* To change the working directory, enter the following command:

`cd ~/cloudfunctions/start`

* To configure the Quiz application, enter the following command:

`. prepare_environment.sh`


* **This script file:**
 * Creates an App Engine application.
 * Exports the environment variables GCLOUD_PROJECT and GCLOUD_BUCKET.
 * Runs npm install.
 * Creates entities in Cloud Datastore.
 * Creates a Cloud Pub/Sub topic.
 * Creates a Cloud Spanner Instance, Database, and Table.
 * Prints out the Google Cloud Platform Project ID.


* To run the web application, enter the following command:

` npm start `


* Preview the web application by clicking on the Web Preview button and selecting ***Preview on port 8080***.

Return to the Cloud Shell and press `CTRL+C` to stop running the app.



## Working with Cloud Functions

The `prepare_environment` script created a Pubsub topic called **Feedback**. You can view it with the command below. The last part of that string is the name of the topic.

`gcloud pubsub topics list`

**Example output:**

```
---
name: projects/google1623327_student@qwiklabs.net/topics/feedback
```

For our Cloud Function source code, create the following two files in a new **sample-pubsub** folder in your current directory:

`mkdir sample-pubsub`

`cd sample-pubsub`

`sudo nano index.js` and paste the below code

`index.js`
```
/**
 * Triggered from a message on a Cloud Pub/Sub topic.
 *
 * @param {!Object} event Event payload.
 * @param {!Object} context Metadata for the event.
 */
exports.helloPubSub = (event, context) => {
  const message = event.data
    ? Buffer.from(event.data, 'base64').toString()
    : 'Hello, World';
  console.log(message);
};

```
* Press Ctrl+O, and then press Enter to save your edited file.
* Press Ctrl+X to exit the nano text editor.

`sudo nano package.json` and paste the below code

`package.json`
```
{
  "name": "sample-pubsub",
  "version": "0.0.1",
  "dependencies": {
    "@google-cloud/pubsub": "^0.18.0"
  }
}
```
* Press Ctrl+O, and then press Enter to save your edited file.
* Press Ctrl+X to exit the nano text editor.

Navigate back to your parent directory

`cd ..`

### Create a Cloud Function

```
gcloud functions deploy process-feedback \
--region us-central1 \
--trigger-topic feedback \
--source sample-pubsub \
--entry-point helloPubSub \
--runtime nodejs10
```
*It might take a minute or two to create the Cloud Function.*


### Run the web application

* To run the web application, enter the following command:

` npm start `

1. Preview the web application by clicking on the Web Preview button and selecting Preview on port 8080.
1. Click Take Test.
1. Click Places.
1. Complete the question.
1. Rate the quiz, enter some feedback, and then click Send Feedback.


### View Cloud Function monitoring and logs

Open a second Cloud Shell tab and run the following command to read the default 20 log from the `process-feedback` function.

You can use the `--limit` to read just fewer number of logs.

```
gcloud functions logs read process-feedback --limit 10
```

*You should see your feedback if you go back to the first Cloud Shell tab.*



## Examining the case study application code


In this section, you use the Cloud Shell text editor to review the case study application code.


### Review the Cloud Function application code structure

* Launch the cloud shell editor and navigate to ***cloudfunctions/start.***


* Select the `index.js` file in the `...function` folder.

*This file contains the same code as the sample from the Cloud Functions window in the Cloud Platform Console, with one change: because the function you will write returns a Promise, the callback argument has been omitted.*

* Select the `package.json` file.

*This file contains the list of dependencies that this function needs to run.
Cloud Functions automatically install the dependencies.*

* Select the `languageapi.js` file.

*This file contains code to process feedback text and return the Natural Language ML API sentiment score.*

* Select the `spanner.js` file.

*This file contains code to insert a record into a Cloud Spanner database.*


## Coding a Cloud Function

*In this section you write the code to create a Cloud Function that retrieves the Cloud Pub/Sub message data, invokes the Natural Language ML API to perform sentiment detection, and inserts a record into Cloud Spanner.*


### Write code to modify a Cloud Function

* Return to the `...function/index.js` file.
* Load the `languageapi` and `spanner` modules. These modules are in the same folder as the `index.js` file.
* In the `subscribe()` method, after the existing code that loads the Cloud Pub/Sub message into a buffer, convert the PubSub message into a feedback object by parsing it as JSON data.
* Return a promise that invokes the `languageapi` module's `analyze` method to analyze the feedback text.
* Chain a `then(...)` method to the end of the return statement.
* Supply an arrow function as the value of the callback.
* In the body of the arrow function, log the `Natural Language API` sentiment score to the console.
* Add a new property called `score` to the `feedback` object.
*Complete the arrow function body by returning the `feedback` object.
* Chain a second `.then(...)` method to the end of the first one. This one uses the `spanner` module to save the feedback.
* Write a third `.then(...)` chained method, including an arrow function with no arguments and an empty body as the value of the callback.
* In the body of this callback, log a message to indicate that the feedback has been saved, and return a success message.
* Attach a `.catch(...)` handler to the end of the chain, which logs the error message to the console.


Here's how your function should look when you're done.

***function/index.js***

```
// TODO: Load the ./languageapi module

const languageAPI = require('./languageapi');

// END TODO

// TODO: Load the ./spanner module

const feedbackStorage = require('./spanner');

// END TODO

exports.subscribe = function subscribe(event) {
  // The Cloud Pub/Sub Message object.
  // TODO: Decode the Cloud Pub/Sub message
  // extracting the feedbackObject data
  // The message received from Pub/Sub is base64 encoded, and
  // the data submitted by students is in a data property     

  const pubsubMessage = Buffer.from(event.data, 'base64').toString();

  let feedbackObject = JSON.parse(pubsubMessage);
  console.log('Feedback object data before Language API:' + JSON.stringify(feedbackObject));

  // END TODO

  // TODO: Run Natural Language API sentiment analysis
  // The analyze(...) method expects to be passed the
  // feedback text from the feedbackObject as an argument,
  // and returns a Promise.

  return languageAPI.analyze(feedbackObject.feedback).then(score => {

    // TODO: Log the sentiment score

    console.log(`Score: ${score}`);

    // END TODO

    // TODO: Add new score property to feedbackObject

    feedbackObject.score = score;

    // END TODO

    // TODO: Pass feedback object to the next handler

    return feedbackObject;

    // END TODO
  })

  // TODO: insert record
  .then(feedbackStorage.saveFeedback).then(() => {
    // TODO: Log and return success

    console.log('feedback saved...');
    return 'success';

    // END TODO

  })
  // END TODO

  // TODO: Catch and Log error
  .catch(console.error);
  // End TODO

};
```

* Save the file.

### Package and deploy the Cloud Function code

* Return to Cloud Shell and stop the web application by pressing Ctrl+C.

* To change the working directory to the Cloud Function code, enter the following command:

`cd function`

* To zip up the files needed to deploy the function, enter the following command:

`zip cf.zip *.js*`

*This generates a zip archive named `cf.zip` that includes all the JavaScript and JSON files in the folder.*

* To stage the zip file into Cloud Storage, enter the following command:

`gsutil cp cf.zip gs://$GCLOUD_BUCKET/`

*This copies the zip archive into the Cloud Storage bucket named after your project ID with a -media suffix.*

* To update the function `process-feedback` to use the zip file in our bucket enter the following command:


```
gcloud functions deploy process-feedback \
--region us-central1 \
--trigger-topic feedback \
--source gs://$GCLOUD_BUCKET/cf.zip \
--entry-point subscribe
```



### Run the web application

1. Change the working folder back to the start folder for the cloudfunctions lab. `cd ..`

1. To start the web application, execute the following command: `npm start`

1. Preview the web application by clicking on the Web Preview button and selecting Preview on port 8080.
1. Click Take Test.
1. Click Places.
1. Complete the question.
1. Rate the quiz, enter some feedback, and then click Send Feedback.

### View Cloud Function monitoring and logs

`gcloud functions logs read process-feedback --limit 10`

*You should see the log entries collected from the Cloud Function.*

*You should now see a score from the Language API in the logs.*

*It may takes a few minutes for the logs to show the invocation of your function.*


### View Cloud Spanner data

```
gcloud spanner databases execute-sql quiz-database \
--instance quiz-instance \
--sql "SELECT * FROM Feedback"
```

*You should see that a new record has been added to the Feedback table.*



### The End.
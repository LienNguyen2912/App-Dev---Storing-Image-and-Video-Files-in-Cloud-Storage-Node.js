# Google Cloud - App Dev - Storing Image and Video Files in Cloud Storage: Node.js
We will
- Create a Cloud Storage bucket.
- Review file upload UI and code changes.
- Write code to store uploaded file data into Cloud Storage.

## Activate Google Cloud Shell
Google Cloud Shell is a virtual machine that is loaded with development tools. It offers a persistent 5GB home directory and runs on the Google Cloud. Google Cloud Shell provides command-line access to your GCP resources.</br>
In GCP console, on the top right toolbar, click the Open Cloud Shell button.</br>
![a](https://user-images.githubusercontent.com/73010204/142588938-f700d742-d45d-46a8-a0e8-d5a89a8d6c01.png)</br>
It takes a few moments to provision and connect to the environment. When you are connected, you are already authenticated, and the project is set to your PROJECT_ID. </br>
**gcloud** is the command-line tool for Google Cloud Platform. It comes pre-installed on Cloud Shell and supports tab-completion.</br>
You can list the active account name with this command:
```sh
gcloud auth list
```
Output:</br>
![b](https://user-images.githubusercontent.com/73010204/142589185-f2aef8e8-0f3f-4bce-bfdd-ca66c2049f1b.PNG)</br>
You can list the project ID with this command:
```sh
gcloud config list project
```
Output:</br>
![c](https://user-images.githubusercontent.com/73010204/142589426-9649e06a-e57f-47ed-9d8a-de2610534f19.PNG)</br>
## Clone source code in Cloud Shell
In Cloud Shell, clone the repository for the class.
```sh
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```
Create a soft link as a shortcut to the working directory.
```sh
ln -s ~/training-data-analyst/courses/developingapps/v1.2/nodejs/cloudstorage cloudstorage
```
## Configure and run the case study application
Navigate to the directory that contains the smaple files for this lab:
```sh
cd ~/cloudstorage/start
```
To configure the application, execute the following command:
```sh
. prepare_environment.sh
```
This script file
- Creates an App Engine application.
- Exports an environment variable, GCLOUD_PROJECT.
- Runs npm install.
- Creates entities in Cloud Datastore.
- Prints out the Google Cloud Platform Project ID.

Here it is
```sh
gcloud app create --region "us-central"
export GCLOUD_PROJECT=$DEVSHELL_PROJECT_ID
npm install -g npm@6.11.3
npm update
export GCLOUD_BUCKET=$DEVSHELL_PROJECT_ID-media
node setup/add_entities.js
```
To run the application, execute the following command:
```sh
npm start
```
## Review the case study application
To view the application, click **Web preview > Preview on port 8080.**</br>
![d](https://user-images.githubusercontent.com/73010204/142590336-cbae670a-f98d-47a4-89d8-1383aa5bac35.PNG)</br>
Output : </br>
![e](https://user-images.githubusercontent.com/73010204/142590625-60bd5314-19e5-4d4e-a70b-a2051bb5ee8a.png)</br>
## Launch the Cloud Shell Editor
In Cloud Shell, click **Open Editor**. Click Open in a new window.</br>
Navigate to the **/cloudstorage/start** folder using the file browser panel on the left side of the editor.</br>
Select the add.pug file in the .../server/web-app/views/questions folder.</br>
- This file contains the pug template for the Create Question form.
- Notice how the form has been modified to use _multipart/form-data_ as the _enc-type_, and there are two new form controls. A file upload control called _image_. A hidden field called _imageUrl_</br>
![f](https://user-images.githubusercontent.com/73010204/142591363-bb3ba4d0-dc62-403e-a4a7-f6a19c2c55d6.png)</br></br>

Select the questions.js file in the .../server/web-app folder. This file has been modified so that the POST handler chains together three distinct middleware calls:</br>
- Configures Multer to accept a single image file from a form field called _image_. Multer is a popular Express middleware package for handling file uploads. The Multer handler consumes the file multipart upload from the browser and holds the file data in memory.
- Consumes the file processed by Multer. This handler is where you will write the Cloud Storage code to upload a file, make it publicly available, and store the public URL for the object.
- Consumes the public URL to store the data into Cloud Datastore. You already wrote the code to store an entity in Cloud Datastore in the previous lab!

Select the **.../server/gcp/cloudstorage.js** file. We will update this file later.
## Create a Cloud Storage bucket
### Create a Cloud Storage bucket
Return to **Cloud Shell**, click **Open Terminal** and press Ctrl+C to stop the application</br>
To create a Cloud Storage bucket named <Project ID>-media, execute the following command:
```sh
gsutil mb gs://$DEVSHELL_PROJECT_ID-media
```
To export the Cloud Storage bucket name as an environment variable named GCLOUD_BUCKET, execute the following command:
```sh
export GCLOUD_BUCKET=$DEVSHELL_PROJECT_ID-media
```
## Adding Objects to Cloud Storage
We will write code to save uploaded files into Cloud Storage by updating **...gcp/cloudstorage.js** as below
```sh
'use strict';
const config = require('../config');
// TODO: Load the module for Cloud Storage
const {Storage} = require('@google-cloud/storage');
// END TODO
// TODO: Create the storage client
// The Storage(...) factory function accepts an options
// object which is used to specify which project's Cloud
// Storage buckets should be used via the projectId
// property.
// The projectId is retrieved from the config module.
// This module retrieves the project ID from the
// GCLOUD_PROJECT environment variable.
const storage = new Storage({
  projectId: config.get('GCLOUD_PROJECT')
});
// END TODO
// TODO: Get the GCLOUD_BUCKET environment variable
// Recall that earlier you exported the bucket name into an
// environment variable.
// The config module provides access to this environment
// variable so you can use it in code
const GCLOUD_BUCKET = config.get('GCLOUD_BUCKET');
// END TODO
// TODO: Get a reference to the Cloud Storage bucket
const bucket = storage.bucket(GCLOUD_BUCKET);
// END TODO
```
Edit **sendUploadToGCS(req, res, next)** function
```sh
function sendUploadToGCS(req, res, next) {
  // The existing code in the handler checks to see if there
  // is a file property on the HTTP request - if a file has
  // been uploaded, then Multer will have created this
  // property in the preceding middleware call.
  if (!req.file) {
    return next();
  }
  // In addition, a unique object name, oname,  has been
  // created based on the file's original name. It has a
  // prefix generated using the current date and time.
  // This should ensure that a new file upload won't
  // overwrite an existing object in the bucket
  const oname = Date.now() + req.file.originalname;
  // TODO: Get a reference to the new object
  const file = bucket.file(oname);
  // END TODO
  // TODO: Create a stream to write the file into
  // Cloud Storage
  // The uploaded file's MIME type can be retrieved using
  // req.file.mimetype.
  // Cloud Storage metadata can be used for many purposes,
  // including establishing the type of an object.
  const stream = file.createWriteStream({
    metadata: {
      contentType: req.file.mimetype
    }
  });
  // END TODO
  // TODO: Attach two event handlers (1) error
  // Event handler if there's an error when uploading
  stream.on('error', err => {
    // TODO: If there's an error move to the next handler
    next(err);
    // END TODO
  });
  // END TODO
  // TODO: Attach two event handlers (2) finish
  // The upload completed successfully
  stream.on('finish', () => {
    // TODO: Make the object publicly accessible  
    file.makePublic().then(() => {
      // TODO: Set a new property on the file for the
      // public URL for the object  
// Cloud Storage public URLs are in the form:
// https://storage.googleapis.com/[BUCKET]/[OBJECT]
// Use an ECMAScript template literal (`https://...`)to
// populate the URL with appropriate values for the bucket
// ${GCLOUD_BUCKET} and object name ${oname}
      req.file.cloudStoragePublicUrl =
`https://storage.googleapis.com/${GCLOUD_BUCKET}/${oname}`;
      // END TODO
      // TODO: Invoke the next middleware handler
      next();
      // END TODO
    });
    // END TODO
  });
  // END TODO
  // TODO: End the stream to upload the file's data
  stream.end(req.file.buffer);
  // END TODO
}
```
## Run the application and create a Cloud Storage object
Save the **...gcp/cloudstorage.js** file, and then return to the Cloud Shell command.</br>
Start the application by typing
```sh
npm start
```
Download a image file to your local PC, you can get one by this link
https://storage.googleapis.com/cloud-training/gcpdev/lab-artifacts/Google-Cloud-Storage-Logo.svg
</br>In Cloud Shell, click **Web preview > Preview on port 8080** to preview the Quiz application.</br>
Click **Create Question.**</br>
Complete the form and then click **Save.**</br>
![8](https://user-images.githubusercontent.com/73010204/142593191-c398ee92-daf8-4dda-9fc4-63ea6773405d.png)</br>
Return to the Cloud Console and click on **Navigation menu > Cloud Storage.**</br>
On the **Cloud Storage > Browser page**, click the correct bucket (named <Project ID>-media).You should see your new object named like this: </br>
![g](https://user-images.githubusercontent.com/73010204/142607822-cab0267e-6d64-4b52-9d5c-9994927ab1b3.png)
## Run the client application and test the Cloud Storage public URL
Add _/api/quizzes/gcp_ to the end of the application's URL. You should see that JSON data has been returned to the client corresponding to the Question you added in the web application.</br>
![h](https://user-images.githubusercontent.com/73010204/142608224-d8e963a3-d510-4e85-894a-2a0626eb55e1.png)</br>
Return to the application home page and click **Take Test.**</br>
Click **GCP**, and answer each question.</br>
![h](https://user-images.githubusercontent.com/73010204/142608491-3662aa2f-10cb-41d9-a2a3-45429a64f9f7.png)
## Congratulations!
You have created a Cloud Storage bucket and written code to store uploaded file data into Cloud Storage.
## Referenence
**Google Cloud Fundamentals: Getting Started With Application Development** course
https://www.coursera.org/learn/getting-started-app-development/home/welcome

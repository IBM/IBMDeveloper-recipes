# Uploading files to web server using AJAX (XHR)
## Understand how XHR and AJAX are used for file uploads

Shalabh Aggarwal

Published on February 26, 2018 / Updated on November 10, 2020

### Overview

Skill Level: Beginner

The purpose of this recipe is to learn how to upload a file to a web server using AJAX in native javascript using XHR. Learn about single file upload here and extend the same to work for multiple files.

### Ingredients

*   HTML
*   Javascript
*   Any web server which can handle the web request which comes in from a HTML form.

This recipe will not talk about the web server needed to handle the upload request. It is assumed that the reader would have the knowledge to handle this part of program.

### Step-by-step

#### 1. Introduction

This recipe will show you how to upload a single file. Note, however, that the reader can extend the same to build a form for multiple file upload as well.

[AJAX](https://en.wikipedia.org/wiki/Ajax_(programming)) is a very popular web development technique which allows the client side applications to make asynchronous requests to the web server. These asynchronous requests can fetch and retrieve data or submit data to the web server without affecting the overall functionality of the other components in the client side application.

[XHR](https://en.wikipedia.org/wiki/XMLHttpRequest) (XMLHttpRequest) is an API which works with AJAX in which objects can be passed between the web client applications and the web server. Despite the name, XHR supports data in formats other than XML, like JSON, HTML or plain text.

Some useful JS libraries can be used to simplify [upload of a file using AJAX](https://cloudinary.com/blog/file_upload_with_ajax). But beforehand, to get started, getting to know the base concept of XHR is helpful.

#### 2. Create a small HTML form

```
<!DOCTYPE html>

<html>

<head>

<title>File Upload using XHR</title>

</head>

<body>

<form id=”myForm” action=”/upload” method=”post” enctype=”multipart/form-data”>

   Upload a File:

   <input type=”file” id=”myFile” name=”myFile”>

   <br/><br/>

   <input type=”submit” id=”submit” name=”submit” value=”Upload File Now” >

</form>

</body>

<script type=”text/javascript”>

<!– The javascript will go here →

</script>

</html>
```

In this form, we have an input field for the file and a submit button. When a file is selected and submit button is clicked, then a POST request would be sent to the url _/upload_ which should be handled by the web server. Make sure to specify the _enctype_ while creating the form element. It tells the POST request that a file is part of the request being made and sets the [Content-Type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Type).

#### 3. Handle the submit event

First the elements for form, the file field and the submit button need to be identified. Then the file need to be fetched from the _myFile_ form field. The click of submit button should handle this. When the submit button is clicked, the form’s _onsubmit_ event is triggered.

```
window.onload = function() {

var myForm = document.getElementById(‘myForm’);

myForm.onsubmit = function(event) {

event.preventDefault;

var myFile = document.getElementById(‘myFile’);

var mySubmit = document.getElementById(‘submit’);

var files = myFile.files;

// Next code comes here

}

};
```

#### 4. Create the form object for submission

Create form data that would be submitted in the POST request to _/upload_. The _formData_ will contain the file object which can be easily appended. Then an AJAX request can be created by using _XMLHttpRequest()_ method. This would be a POST request to _/upload_ which is same as the one we mentioned while creating HTML form in Step 1.

Then the _xhr.onload_ function would handle the event after completion of upload and will display the success or failure accordingly.

```
var formData = new FormData();

formData.append(‘myFile’, files[0], files[0].name);

var xhr = new XMLHttpRequest();

xhr.open(‘POST’, ‘/upload’, true);

xhr.onload = function () {

if (xhr.status === 200) {

alert(‘File successfully uploaded’)

} else {

alert(‘File upload failed!’);

}

};

xhr.send(formData);
```

---
layout: post
date: 2015-05-18
title: Tessel Camera module + Project Oxford Face APIs
---

In this post I will to show you how to use the [Project Oxford](https://www.projectoxford.ai/) Face APIs from a [Tessel](http://tessel.io/) device. The Project Oxford Face APIs are one of several sets of APIs exposed by the Project Oxford, and the most impressive in my opinion. They allow you to make simple REST calls to identify faces in a picture, and even try to guess their gender and age!

I am going to use the [Tessel Camera module](https://tessel.io/modules#module-camera) plugged in port "A" to take a picture when the Config button is pressed. This is very easy with the Tessel and the code looks like this:

```javascript
tessel.button.on('press', function() {
  takePicture();
});
```

As you can see the `tessel` module gives us some convenience objects to easily bind a callback to the button. The function `takePicture` is where we will use the Tessel-provided Camera module to actually take the picture:

```javascript
function takePicture() {
  // Take a picture
  camera.takePicture(function(err, image) {
    if (err) {
      console.log('error taking image', err);
    } else {
	  // Do something with the image!
      var name = 'picture-' + Math.floor(Date.now()*1000) + '.jpg';
      uploadPicture(name, image);
    }	
  });
}
```

The Tessel makes it really easy to use the Camera module. Now we need to upload this picture somewhere so we can use it with the Project Oxford APIs. What a coincidence, I happen to know a very good cloud storage platform!

We are going to upload the image to **Azure Blob Storage** using a Shared Access Signature for authorization. The code looks like this: (some details omitted, please check the [full source code](https://github.com/tomconte/oxford-tessel-demo))

```javascript
function uploadPicture(name, image)
{
  var url = 'http://' + blob_host + '/' + blob_container + '/' + name;
  
  var options = {
    hostname: blob_host,
    port: 80,
    path: '/' + blob_container + '/' + name + blob_sas,
    method: 'PUT',
    headers: {
      'Content-Length' : image.length,
      'x-ms-blob-type' : 'BlockBlob',
      'Content-Type' : 'image/jpeg'
    }
  };

  var req = https.request(options, function(res) {
    
    // Now that the Blob is uploaded,
    // call the Face API
    
    faceDetect(url);
  });

  req.write(image);
  req.end();
}
```

We are simply using the standard Node.JS HTTP module here to upload the picture via a PUT request. The only magic value is `blob_sas`, which is defined like this:

```javascript
var blob_sas = '?sv=2014-02-14&sr=c&sig=xxxx&se=2019-12-31T23%3A00%3A00Z&sp=rwdl';
```

This key gives the bearer (i.e. your Tessel program) some rights on a specific Blob Container. You can use a number of tools to generate this Shared Access Key e.g. Azure Management Studio, or Azure Storage Explorer. You can also use the Azure PowerShell cmdlets:

```
New-AzureStorageContainerSASToken -Name test -Permission rwdl
```

All these tools will output a SAS string that you can just copy/paste into your code.

Finally, we are going to send the resulting picture URL to the Project Oxford Face Detection API in order to detect faces in the picture!

```javascript
function faceDetect(url) {
  var json = JSON.stringify({'url':url});

  var options = {
    hostname: 'api.projectoxford.ai',
    port: 80,
    path: faceapi_request,
    method: 'POST',
    headers: {
		  'Host': 'api.projectoxford.ai',
      'Content-Length' : json.length,
      'Content-Type' : 'application/json',
      'Ocp-Apim-Subscription-Key': oxford_key
    }
  };

  var req = https.request(options, function(res) {
    res.on('data', function(d) {
	  // Parse Face API response
    });
  });
}
```

The result from the Face API call tells us if it found a face in the picture, and if yes it tries to evaluate its gender and age. Here is how you can interpret the response from the API:

```javascript
// Face API response
var face = JSON.parse(d);

if (face.length == 0) {

  // No face detected
  console.log("\nNO FACE DETECTED\n");

} else {

  // Face was detected
  console.log("\nFACE DETECTED\n");
  console.log(face[0].faceId);
  console.log(face[0].attributes.gender);
  console.log(face[0].attributes.age);

}
```

Basically, you parse the JSON returned from the Face API. If it is an empty array, then no faces were detected. If it is an array containing one entry or more, then you will get the following information for each face:

- a Face ID, that you can then use in further API calls for face comparison or identification;
- a bunch of attributes, like the position of face attributes (eyes, nose...)
- a guess at the face gender
- a guess at the face age
- an approximation of the head pose

Some of these attributes must be requested in the API call, for example here is the one I used:

```javascript
var faceapi_request = '/face/v0/detections?analyzesFaceLandmarks=false&analyzesAge=true&analyzesGender=true&analyzesHeadPose=false';
```

You can see here that I requested "face landmarks", "age" and "gender", but did not request "head pose".

Check out the full [Project Oxford](https://www.projectoxford.ai/face) documentation for what you can do with this amazing API!

You can jump to this [Github repository](https://github.com/tomconte/oxford-tessel-demo) where you will find the full working source code for a demo that shows how a LED plugged in GPIO port G3 is turned on if a face is detected.

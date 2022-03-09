### Remove character from weebcam with nextjs


## Introduction

This article demonstrates how to remove a character from webcam in realtime view in next js.



The final github code can be viewed [here](/).


## Codesandbox

The final demo on [Codesandbox](/).

<CodeSandbox
title="webcam-character-remover"
id=" "
/>

## Prerequisites

Entry-level knowledge in both javascript acnd react.

## Project setup

Use the command `npx create-next-app webcam_xter` to create a new Next.js project
You can head to the directory using `cd videomerge` when the project is ready.

We will begin with backend congiguration. The project backend will involve setting up a nextjs handler function that will upload our media files to [Cloudinary](https://cloudinary.com/?ap=em) website for online storage. The

You can start by using this [link](https://cloudinary.com/console) to log in  or create your own cloudinary account. In the website you will be provided a dashboard which will contain environment variables, your `cloudinary name`, `api key` and `api secret` necessary for the upload configuration. To include them in your project , create afile named `.env` in your root directory and paste inside the following

```
".env"

CLOUDINARY_CLOUD_NAME =

CLOUDINARY_API_KEY =

CLOUDINARY_API_SECRET=
```

Replace the blacks above with values from your cloudinary project and restart your project using `npm-run-dev`.

Next move is to create a file named`upload.js` in the projects's `pages/api` directory.
In the `upload` file start by configuring the cloudinary environment keys which will prevent code duplictation.


```
var cloudinary = require("cloudinary").v2;

cloudinary.config({
    cloud_name: process.env.CLOUDINARY_NAME,
    api_key: process.env.CLOUDINARY_API_KEY,
    api_secret: process.env.CLOUDINARY_API_SECRET,
});
```
We can then introduce a handler function which will handle our upload post request.

```
export default async function handler(req, res) {
    if (req.method === "POST") {
        let url = ""
        try {
            let fileStr = req.body.data;
            const uploadedResponse = await cloudinary.uploader.upload_large(
                fileStr,
                {
                    resource_type: "video",
                    chunk_size: 6000000,
                }
            );
            url = uploadedResponse.url
        } catch (error) {
            res.status(500).json({ error: "Something wrong" });
        }

        res.status(200).json({data: url});
    }
}
```

The function abovewill receive our frontend request body and upload it to cloudinary. It will then capture the uploaded media file's cloudinary url and send it back to front end as a response.

This concludes our backend. Let us now proceed to creating our application.

### Front End
To execute our application, we'll need to download  [Tensorflow](https://www.tensorflow.org/js/) .
Tensorflow is a javascript library used for training and deploying machine learning models. You can use this  [link](https://www.tensorflow.org/js/)  to explore its capabilities. Lets begin!


Start by installing the tensorflow dependencies of our application. 

`npm install @tensorflow/tfjs @tensorflow-models/body-pix`

Next we create our UI elements. We will need a video tag element set to `muted`, `loop` and `controls` and also an output canvas to display the processed video. We will also create several buttons to fire our app's respective commands. We will use `useRef` to refference the video elements.

Your code at this point to implement the above should look like this:

```
import React, { useRef, useEffect, useState } from "react";
import * as tf from "@tensorflow/tfjs";
import * as bodyPix from "@tensorflow-models/body-pix";


export default function Home() {
  let ctx_out, video_in, ctx_tmp, c_tmp, c_out;

  const processedVid = useRef();
  const rawVideo = useRef();
  const startBtn = useRef();
  const closeBtn = useRef();
  const videoDownloadRef = useRef();
  const [model, setModel] = useState(null);

  return(
    <>
        <div className="container">
          <div className="header">
            <h1 className="heading">
                Remove character from webcam
            </h1>
          </div>
          <div className="row">
            <div className="column">
              <video
                className="display"
                width={800}
                height={450}
                ref={rawVideo}
                autoPlay
                playsInline
              ></video>
            </div>
            <div className="column">
              <canvas className="display" width={800} height={450} ref={processedVid}></canvas>
            </div>
          </div>
          <div className="buttons">
            <button className="button" onClick={startCamHandler} ref={startBtn}>
              Start Webcam
            </button>
            <button className="button" onClick={stopCamHandler} ref={closeBtn}>
              Close and upload original video
            </button>
            <button className="button">
              <a ref={videoDownloadRef} href={videoUrl}>
                Get Original video
              </a>
            </button>
          </div>
        </div>
      )}
 
    </>
  )
}
```
You can use the github repo provided earlier in the article to duplicate the additional css configurations or use your own prefference.

With the DOM elements ready, let us begin with the neural network architecture. In this article, we will involve `MobileNetV1` model configuration. We use this because despite it's lower accuracy, it involves faster configurations which is preferable for our learning process. Use the following code to create `MobileNetV1` model configuration. 

```
"pages/index.js"

const modelConfig = {
  architecture: "MobileNetV1",
  outputStride: 16,
  multiplier: 1,
  quantBytes: 4,
};

```
In the code above, the `architecture` param determines which bodypix architecture to load. We can choose to use `ResNet`, which is more accurate but in our case, it will be larger and too slow for a learning process. You can use this  [Link](https://github.com/tensorflow/tfjs-models/tree/master/body-pix) to learn more about these configurations.
There are 2 strides supported by MobileNetV1's  `outputStride` param. That is the 8 and 16. They are used to specify the the output stride of the BodyPix model.
The `multiplier` is the float multiplier for the number of channels for all convolution ops. It can be one of 1.0, 0.75 or 0.5
`quantBytes` control bytes used for weight quantitization. Its available option are 1, 2 and 4.

Next we work on our segmentation configuration. 
```
"pages/index.js"

  const segmentationConfig = {
    internalResolution: "full",
    segmentationThreshold: 0.1,
    scoreThreshold: 0.4,
    flipHorizontal: true,
    maxDetections: 1,
  };

```
In the above configuration, we set `internalResolution` to full which will prevent resizing. For a better performance you can downsize the input image before processing.
`segmentationThreshold` reffers to the minimum pixel confident threshold before it identified a human body. 
The `scoreThreshold` represents the minimum confident threshold to recognise an entire human body. The default
We can then load the `bodypix` model with the model configuration inside a `useEffect` hook.
we will also flip our video horizontally set proper orientation for the pose and segmentation.
`maxDetections` detects the maximum number of person to detect per image.


We can the load our `bodypix` model with our model configuration inside a `useEffect` hook.

```
"pages/index.js"


useEffect(() => {
    if (model) return;
    const start_time = Date.now() / 1000;

    bodyPix.load(modelConfig).then((m) => {
      setModel(m);
      const end_time = Date.now() / 1000;
      console.log(`model loaded successfully, ${end_time - start_time}`);
    });
  }, []);

```

Declare the following variables. We'll use them to capture our stream and capture our videos.
```
"pages/index.js"

let recordedChunks = [];
  let localStream = null;
  let options = { mimeType: "video/webm; codecs=vp9" };
  let mediaRecorder = null;
  let videoUrl = null;
``` 

Create a function `startCamHandler` after the above variables.

```
 const startCamHandler = async () => {
    console.log("Starting webcam and mic ..... ");
    localStream = await navigator.mediaDevices.getUserMedia({
      video: true,
      audio: false,
    });

    // console.log(model);

    //populate video element
    rawVideo.current.srcObject = localStream;
    video_in = rawVideo.current;
    rawVideo.current.addEventListener("loadeddata", (ev) => {
      console.log("loaded data.");
      transform();
    });

    mediaRecorder = new MediaRecorder(localStream, options);
    mediaRecorder.ondataavailable = (event) => {
      console.log("data-available");
      if (event.data.size > 0) {
        recordedChunks.push(event.data);
      }
    };
    mediaRecorder.start();
  };
```

Above, we activate user webcam while leaving the audio stream disabled.
We then populate the video element with the webcam stream and run the function named `transform` which will capture the canvas element's context and populate it with current video frame using a `drawImage` method. We then use the `getImageData` method to get the pixel data on the canvas. Use this pixel data with the `segmentPerson` method to begin execute analyzation.
Using a nested loop, iterate through each frame's pixel. We'll use variable 'n' to iterate through each array which are stored in single dimentional format.

Using an if statement, we'll loop each pixel to check if it belongs to a human or not. If it does, the pixel data will be copied. If not, we skip the updating process.

```
 let transform = () => {
    // let ;
    c_out = processedVid.current;
    ctx_out = c_out.getContext("2d");

    c_tmp = document.createElement("canvas");
    c_tmp.setAttribute("width", 800);
    c_tmp.setAttribute("height", 450);

    ctx_tmp = c_tmp.getContext("2d");

    computeFrame();
  };

  let computeFrame = () => {
    ctx_tmp.drawImage(
      video_in,
      0,
      0,
      video_in.videoWidth,
      video_in.videoHeight
    );

    let frame = ctx_tmp.getImageData(
      0,
      0,
      video_in.videoWidth,
      video_in.videoHeight
    );

    model.segmentPerson(frame, segmentationConfig).then((segmentation) => {
      let output_img = ctx_out.getImageData(
        0,
        0,
        video_in.videoWidth,
        video_in.videoHeight
      );

      for (let x = 0; x < video_in.videoWidth; x++) {
        for (let y = 0; y < video_in.videoHeight; y++) {
          let n = x + y * video_in.videoWidth;
          if (segmentation.data[n] == 0) {
            output_img.data[n * 4] = frame.data[n * 4]; // R
            output_img.data[n * 4 + 1] = frame.data[n * 4 + 1]; // G
            output_img.data[n * 4 + 2] = frame.data[n * 4 + 2]; // B
            output_img.data[n * 4 + 3] = frame.data[n * 4 + 3]; // A
          }
        }
      }
      // console.log(segmentation);
      ctx_out.putImageData(output_img, 0, 0);
      setTimeout(computeFrame, 30);
    });
  };
```

...and thats it! We have succesfully achieved the process of removing a human character from webcam view. The final UI experience should be as displayed below

![UI DEMO](https://res.cloudinary.com/dogjmmett/image/upload/v1646817203/final_dofwje.jpg "UI DEMO")

Try out our developing and demo to enjoy the experience.

Happy coding!
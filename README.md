# Twilio Flex Recodings sent to S3

See Lehel's blogpost for most of the following info:
> https://www.twilio.com/blog/encrypt-store-twilio-flex-recordings-site

Prerequisites:
1. Billing setup (cannot enable WFO w/o it)
2. WFO enabled
3. Call Recording enabled in WFO

The flow is configured in three parts:
1. Custum Flex Plugin
    - Replaces action "AcceptTask" to connect with the twilio function
2. Twilio Function
    - Creates a callback URL and sends AWS creds to self-hosted proxy server
3. Self-Hosted Proxy Server for S3
    - Simple node.js server that streams the recording and sends it to the S3 bucket


## Custom Flex Plugin
1. Create a flex plugin (follow: https://www.twilio.com/docs/flex/quickstart/getting-started-plugin)

    `npm init flex-plugin plugin-recordControls --install`
2. Replace the contents of `./plugin-recordControls/src/CallRecordFunctionPlugin.js` with the following:
    ```JavaScript
    import { FlexPlugin } from 'flex-plugin';
    import React from 'react';
    import CustomTaskListComponent from './components/CustomTaskListComponent';

    const PLUGIN_NAME = 'CallRecordFunctionPlugin';

    export default class CallRecordFunctionPlugin extends FlexPlugin {
    constructor() {
        super(PLUGIN_NAME);
    }

    /**
     * This code is run when your plugin is being started
     * Use this to modify any UI components or attach to the actions framework
     *
     * @param flex { typeof import('@twilio/flex-ui') }
     * @param manager { import('@twilio/flex-ui').Manager }
     */
    init(flex, manager) {
        flex.Actions.replaceAction('AcceptTask', (payload, original) => {
        payload.conferenceOptions.record = 'true';
        payload.conferenceOptions.recordingStatusCallback = 'https://<Your Runtime Domain>/recording-status-callback';
        return original(payload);
        });
    }
    }

    ```
3. Replace the `<Your Runtime Domain>` with the domain of your flex (See in your console: https://www.twilio.com/console/runtime/overview)
4. Build (`npm run build`) and upload the asset  `./plugin-recordControls/build/plugin-callRecordFunction.js` to twilio console assest (See in your console: https://www.twilio.com/console/runtime/assets/public) **MAKE SURE TO UPLOAD AS PUBLIC**

## Twilio Function
1. Create a function in console (https://www.twilio.com/console/runtime/functions/manage) by pressing the red plus sign
2. Select blank template
3. Set Function Name to "recording-status-callback" and the Path to "/recording-status-callback"
4. Copy the following into the code for the function:
    ```JavaScript
    const axios = require('axios');
    let AWS = require('aws-sdk');
    const S3UploadStream = require('s3-upload-stream');

    exports.handler = async function(context, event, callback) {
        Object.keys(event).forEach( thisEvent => console.log(`${thisEvent}: ${event[thisEvent]}`));

        // Set the region
        AWS.config.update({region: 'us-east-1'});
        AWS.config.update({ accessKeyId: context.AWSaccessKeyId, secretAccessKey: context.AWSsecretAccessKey });

        const s3Stream = S3UploadStream(new AWS.S3());

        // call S3 to retrieve upload file to specified bucket
        let upload = s3Stream.upload({Bucket: context.AWSbucket, Key: event.RecordingSid, ContentType: 'audio/x-wav'});

        const recordingUpload = await downloadRecording(event.RecordingUrl, upload);

        let client = context.getTwilioClient();
        let workspace = context.TWILIO_WORKSPACE_SID;

        let taskFilter = `conference.participants.worker == '${event.CallSid}'`;

        //search for the task based on the CallSid attribute
        client.taskrouter.workspaces(workspace)
        .tasks
        .list({evaluateTaskAttributes: taskFilter})
        .then(tasks => {

            let taskSid = tasks[0].sid;
            let attributes = {...JSON.parse(tasks[0].attributes)};
            attributes.conversations.segment_link = `https://<your-proxy-server-address>/?recordingSid=${event.RecordingSid}`;

            //update the segment_link
            client.taskrouter.workspaces(workspace)
            .tasks(taskSid)
            .update({
                attributes: JSON.stringify(attributes)
            })
            .then(task => {
                callback(null, null);
            })
            .catch(error => {
                console.log(error);
                callback(error);
            });
        })
        .catch(error => {
            console.log(error);
            callback(error);
        });
    };

    async function downloadRecording (url, upload) {

    const response = await axios({
        url,
        method: 'GET',
        responseType: 'stream'
    })

    response.data.pipe(upload);

    return new Promise((resolve, reject) => {
        upload.on('uploaded', resolve)
        upload.on('error', reject)
    })
    }
    ```
5. Replace the `<your-proxy-server-address>` in the code with the address of your self-hosted proxy server
6. Save.
7. Set the following environment variables for the function (See in your console: https://www.twilio.com/console/runtime/functions/configure)

    | Key | Value |
    | --- | --- |
    | AWSaccessKeyId | from aws |
    | AWSbucket | from aws |
    | AWSsecretAccessKey | from aws |
    | TWILIO_WORKSPACE_SID | see taskrouter (https://www.twilio.com/console/taskrouter/dashboard) |

8. Set the following Dependencies in the same page as the previous step:

    | Name | Version |
    | --- | --- |
    | aws-sdk | (leave blank) |
    | axios | (leave blank) |
    | s3-upload-stream | (leave blank) |

## Self-Hosted Proxy Server for S3
This step is open to your own deployment methods. For proof of concept the following is an example using https://ngrok.com to host your development env with a public endpoint.

1. Spin up a server using the following code:
    ```JavaScript
    const express     = require('express');
    const app         = express();
    const aws         = require('aws-sdk');
    const downloader  = require('s3-download-stream');
    require('dotenv').config();
    require('log-timestamp');

    aws.config.update({ region: 'us-east-1' });
    aws.config.update({ accessKeyId: process.env.accessKeyId, secretAccessKey: process.env.secretAccessKey });

    const s3 = new aws.S3({apiVersion: '2006-03-01'});

    app.listen(8080, function() {
    console.log('[NodeJS] Application Listening on Port 8080');
    });

    app.get('/', function(req, res) {

        const recordingSid = req.query.recordingSid;

        console.log('Received playback request');
        console.log(recordingSid);

        let range = req.headers.range;
        console.log(range);

        let config = {
        client: s3,
        concurrency: 6,
        params: {
            Key: recordingSid,
            Bucket: process.env.bucket
        }
        }

        if (range !== undefined || range !== 'bytes=0-') {
        config.params.Range = range;
        }

        s3.headObject(config.params, (error, data) => {
        if (error) {
            console.log(error);
        }

        console.log(data);

        if (range !== undefined) {

            let contentRange = data.ContentRange;
            if (range === 'bytes=0-') {
            contentRange = `bytes 0-${data.ContentLength - 1}/${data.ContentLength}`;
            config.params.Range = `bytes=0-${data.ContentLength - 1}`;
            }

            res.status(206).header({
            'Accept-Ranges': data.AcceptRanges,
            'Content-Type': 'audio/x-wav',
            'Content-Length': data.ContentLength,
            'Content-Range': contentRange
            });
        } else {
            res.header({
            'Accept-Ranges': data.AcceptRanges,
            'Content-Type': 'audio/x-wav',
            'Content-Length': data.ContentLength
            });
        }

        downloader(config).pipe(res);
        })
    });
    ```
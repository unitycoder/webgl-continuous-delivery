# webgl-continuous-delivery
Reference project on how to automate uploading WebGL artifacts to S3 from Unity Cloud

### Guide

https://discussions.unity.com/t/export-build-from-cloud-build-to-amazon-s3/713648/11

Process breakdown

    Add fastlane to your Unity project. It’s how we’ll do all the syncing with S3
    Create a script to trigger the lanes
    Modify the build configuration in Unity Cloud for:
    3.1. Setting the former script as the post-build script in the Script hooks section
    3.2. Adding the necessary environment variables

Reference project

This is the reference project, where you can find the code to trigger the WebGL sync.
The important parts are:

    fastlane/Fastfile: this has the two lanes that make the job:
        upload_dir_to_s3 lane
        empty_s3_bucket lane
    Scripts/upload_webgl_build_to_s3.sh: the script that calls both lanes; this needs to be added in the build configuration as the post-build script

From the process breakdown, that reference project covers points 1. and 2. You then need to do the 3rd step by customizing the build configuration for calling the script and adding the environment variables. In this reference project we need 4:

    AWS_REGION
    AWS_ACCESS_KEY_ID
    AWS_SECRET_ACCESS_KEY
    WEBGL_BUCKET

The variable UNITY_PLAYER_PATH used in the script is automatically set in Unity Cloud worker.
If you want to test this locally, you can trigger the script by executing:

projectRoot~> AWS_REGION=your-preferred-region \
    AWS_ACCESS_KEY_ID=your-access-key-id \
    AWS_SECRET_ACCESS_KEY=your-secret-access-key \
    WEBGL_BUCKET=the-bucket-name \
    UNITY_PLAYER_PATH=/home/jane-doe/projects/webgl-continuous-delivery/build \
    Scripts/upload_webgl_build_to_s3.sh

Little caveats I had to get through along the way

    don’t use fastlane built-in s3 action, it’s deprecated
    don’t use fastlane plguin aws_s3, it didn’t work for me
    do use ruby’s offical aws-sdk-s3
    you need to create a lane for emptying the S3 bucket
    you need another lane for copying the new build
        you must iterate through the files in the directory and copy them sequentially, given there’s no bulk operation for uploading a whole directory (an alternative to this is to upload the .zip file containing everything and then using a lambda for decompression)
        ruby’s AWS SDK doesn’t set properly the metadata for content-type nor content-encoding when uploading a file (this is handled correctly by AWS if you upload the directory through the web console). This is crucial to get the thing running properly. So I had to use Marcel to infer the mime type, and examined the files extension to determine whether it’s gzipped or brotlied, then explicitly set the content-type and content-encoding on the put_object operation
    the build artifacts are under the folder determined by UNITY_PLAYER_PATH (environment variable provided by unity cloud build). Everything inside here is exactly what we should copy to the bucket
    don’t forget to set the environment variables to provide AWS configuration
    it’s recommended to run a CloudFront invalidation to evict cached resources every time you deploy


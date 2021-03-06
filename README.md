# animl-lambda
Lambda function for processing camera trap images.

## Related repos
- Animl base program      http://github.com/tnc-ca-geo/animl-base
- Animl ML resources      http://github.com/tnc-ca-geo/animl-ml
- Animl desktop app       https://github.com/tnc-ca-geo/animl-desktop
- Animl cloud platform    https://github.com/tnc-ca-geo/animl

## About
The function is written in python and intended for a python 3.6 lambda runtime.

It utilizes a Perl executible called [exiftool](https://exiftool.org/) to 
extract image metadata. Exiftool is used because much of the important image 
metadata for certain manufacturers (including camera serial numbers in Reconyx 
images) are encoded in the MakerNotes field of the exifdata, and exiftool is 
the only metadata extractor that maintains manufacturer-specific decoders to 
decode MakerNotes for a wide variety of cameras.

We are able to 
[run Perl executibles](https://metacpan.org/pod/AWS::Lambda#Use-Prebuild-Public-Lambda-Layer) 
by adding a Perl runtime layer to our lambda function with the following ARN:

```
arn:aws:lambda:us-west-1:445285296882:layer:perl-5-26-runtime:12
```

## Development

### Prerequisits
The instructions below assume you have the following tools installed:
- aws-cli
- aws-vault
- docker

### Create the lambda function
This is only necessary if the function isn't yet created. 
If it is, use update-function (see below)

```sh
aws-vault exec home -- aws lambda create-function --function-name ProcessCamtrapImage \
--zip-file fileb://function.zip --handler lambda_function.handler --runtime python3.6 \
--timeout 15 --memory-size 1024 \
--role arn:aws:iam::719729260530:role/animl-lambda-role \
--layers arn:aws:lambda:us-west-1:445285296882:layer:perl-5-26-runtime:12
```

Grant lambda function permission to fire on s3 object change event:

```sh
aws-vault exec home -- aws lambda add-permission --function-name ProcessCamtrapImage --principal s3.amazonaws.com \
--statement-id s3invoke --action "lambda:InvokeFunction" \
--source-arn arn:aws:s3:::animl-images \
--source-account 719729260530
```

Navigate to the AWS Lambda console to add the s3 ObjectCreated trigger for 
s3://animl-images (not sure how to do this from the aws lambda cli).

### Update the function's dependencies
Python packages that have C bindings need to be built on a computer with the 
same architecture as that which Lambda functions run (i.e., Linux), so to 
update and repackage the python dependences, you'll need to spin up a docker 
container to emulate the amazon linux environment and install the packages 
there.

1. Check if the docker container exists. If you see the ```animl/lambda``` 
image after running the following command, skip to step 3:

```sh
$ docker images
```

2. Build docker container to emulate lambda

```sh
$ docker build -t animl/lambda .
```

3. Run docker container in interactive mode to emulate lambda

```sh
$ docker run --rm -v $(pwd)/output:/output -it animl/lambda bash
```

Note:

The ```-v``` flag makes your output directory available inside the container 
in a directory of the same name.
The ```-it``` flag means you get to interact with this container on launch.
The ```--rm``` flag means Docker will remove the container when you’re finished.
```animl/lambda`` is the name of the container image.


4. Create and activate virtual env within the container

```sh
$ cd output/
$ python3.6 -m venv --copies env
$ source env/bin/activate
```

5. Download exiftool and copy its executable and dependencies into 
`/output/exiftool/`

```sh
$ curl -o Image-ExifTool-12.01.tar.gz https://exiftool.org/Image-ExifTool-12.01.tar.gz
$ tar -zxf Image-ExifTool-12.01.tar.gz
$ mkdir exiftool
$ cp Image-ExifTool-12.01/exiftool exiftool/
$ cp -r Image-ExifTool-12.01/lib exiftool/
```

6. Install dependencies

```sh
$ pip3.6 install --upgrade pip wheel
$ pip3.6 install --no-binary pillow pillow
$ pip3.6 install --no-binary requests requests
$ pip3.6 install --no-binary PyExifTool PyExifTool
```

7. Strip out unnecessary files and zip up the site packages & exiftool executable

```sh
$ base_dir="/output"
$ find $VIRTUAL_ENV/lib/python3.6/site-packages/ -name "*.so" | xargs strip
$ pushd $VIRTUAL_ENV/lib/python3.6/site-packages/ && zip -r -9 -q $base_dir/venv.zip * ; popd
$ zip -g -r venv.zip exiftool
```

8. Exit out of docker container

```sh
$ deactivate
```
Ctrl+D to exit docker.

You will now have access to the zipped dependencies in ```output/venv.zip```

### Update the function and deploy to AWS

```
$ bash scripts/deploy.sh
```

### Testing

1. Upload an image to s3://animl-data-staging
2. View logged output in AWS Cloudwatch

Image Recognition using workload generator, AWS EC2, Queues and Buckets.

Architecture of the project: 
![image](https://user-images.githubusercontent.com/38141800/200202800-e7ee786e-4ebb-4e1b-af7e-99d3598066af.png)

We extensively used 3 Amazon services: Elastic Compute Cloud (EC2), Simple Queue Service
(SQS), Simple Storage Service (S3).
There are 4 components:
1. Web Tier (EC2)
2. Response and Request Queues (Amazon SQS)
3. Application Tier (EC2 and AMI)
4. Input and Output bucket (Amazon S3)

Web Tier:
The Web Tier had the following components:
a) A Flask application (appTier.py) that acted as the host server and received images from the
user. For each image, a unique ID is generated. Image along with its ID is pushed to the
Request queue as a message. Once the message is pushed into the queue this code polls a
folder for a text file with the Unique ID as a part of its name.
b) A Response Queue Listener which will listen to the messages from the response and
generates a text file classifying the image and the unique ID in its name
c) A Scaling logic that keeps track of the request queue size and scales out or scales in by
creating and terminating instances.
Application Tier:
The application tier contains the AMI that has the image recognition model. Along with the
model, it has an image classification and listener file that is instantiated when any instance with
the AMI is created. We passed the necessary Ubuntu commands while creating the instance
with the AMI to trigger the image recognition file that classifies the images. The classified label
along with its unique ID is pushed to the Response queue and the image and the classification
result is put in the input and output S3 buckets respectively. The number of instances generated
of the App Tier is based on the number of requests received from the Web Tier
Response and request Queues:
These queues are used as buffers to accommodate the delay of messages caused by the
processing time of the image recognition logic. Usage of the queues helps us to decouple the
web and the application tier thus providing the ability to scale.
Input and Output Bucket:
This utilizes S3 (Simple Storage Service) which is highly durable. The Input bucket stores the
images sent by the user through the requests and the Output bucket stores the classified label
of the corresponding image.


Code description:

appTier.py ( Web Tier):
● Using the AWS credentials like the access key id, secret access key and input queue url
we get access to the input queue using the BOTO3 library.
● This file contains the code for the flask application and is mainly used to bridge the gap
between user and the input queue.
● We catch the API request sent and get the image that was sent by the user for
classification. If the filename is not empty, it is saved in a folder called requests_files.
● We then generate a unique ID for the input image and encode the image before sending
it to the input queue. Encoding is done because the SQS is incapable of storing images
in JPEG format.
● Once the file is uploaded to the queue, we remove the input image from the
requests_files and wait for the classification results.

recognition.py (App Tier):
● This file consumes messages from the input queue by calling the sqs.receive_message
API constantly to check for any new incoming messages. Each message from the
response of the queue is processed for further classification in the process_message
method
● The process_message method extracts the message ID (UID), image name, and the
actual image. The image is then base64 decoded to save it in JPEG locally in the
appTier instance. Once this image is processed, the path to these locally saved images
is sent to the classifier method.
● The code for the classifier method is adopted from the classification file provided by the
Professor. This predicts the label from the image provided to it and returns the label.
● Once the classification is done, the image file is converted to an Object and saved into
the input S3 bucket. The predicted label along with the fileName of the current image is
saved in the S3 output bucket. The result with the file name, predicted label and unique
ID is returned from the process_message method.
● This result is sent to the output SQS by calling send_message method. Once processing
is complete for an image, the message is deleted from the input and output queues. All
these steps are repeated till there are no messages remaining in the queue.
outputQueueListener.py ( Web Tier):
● Using the AWS credentials like the access key id, secret access key and output queue
url we get access to the output queue using the BOTO3 library.
● This file polls the output queue for any new messages. If it finds a new message(a new
UUID), the code opens the text file with the UUID as the name and writes the predicted
label in the file.
● Once the message is processed successfully, it is deleted from the output queue.

scalingCode.py ( Web Tier):
● Using the AWS credentials like the access key id and the secret access key we get
access to the input queue using the BOTO3 library.
● The get_instance_count method returns the input queue size. It uses the parameters:
ApproximateNumberOfMessages’,‘ApproximateNumberOfMessagesNotVisible’.
● The current variable holds the value of the currently running instances, which also helps
in the naming of the newly created instances.
● The create_apptier_instances creates the apptier instances with ‘apptier-’ prefix and
instances are created with the given AMI id, user data( which contains the commands to
recognition.py) and security group. The scale out logic is briefly explained in section 2.2
● The terminate_apptier_instances method uses the terminate method depending on the
tracker array. If the track array is filled completely with the same number (x) over its
entire size,then x number of instances are going to be deleted.The scale in logic is briefly
explained in section 2.2

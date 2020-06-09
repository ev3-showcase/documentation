# EV3 Car Showcase

## create project 
```
oc new-project yourinitials-sc-carcontrol
```

## Message Broker Setup (https://github.com/ev3-showcase/message-broker)
- type: Catalog Template - HiveMQ

### Overview

This is a modification of the templates provided on Docker Hub for K8s:

* [HiveMQ3](https://hub.docker.com/r/hivemq/hivemq3/)
* [HiveMQ4](https://hub.docker.com/r/hivemq/hivemq4/)

There are a few clean ups to ensure only the correct ports will be exposed, also we
applied some changes to get the Namespace name for service discovery automatically.

To get started, clone this repository locally and change into the folder.
```
git clone https://github.com/ev3-showcase/message-broker
```

### Create Template

In order to use this Template from out of our catalog within OpenShift we need to import the template file first:
```
oc create -f hivemq4-template.yml
```

### Broker Setup

After the Template has been created, we are able to access it via webconsole > catalog or use it directly via cli.
Each of the template has an `app` label, which in this case defaults to `hivemq4-example`. This can be overwritten by using the `NAME` parameter.
```
oc new-app --template=hivemq4-cluster -p NAME=message-broker
```

### Accessing the Control Interface
To get access to the WebUI, expose the service which is responsible for the Control Center (determine by name by running `oc get svc`) with `oc expose svc/<servicename>`

### cleanup
To remove all items created by this template, use
`oc delete all -l app=hivemq4-example`

Change the value of the label to whatever you specified for the `NAME` variable during deployment.


## API-Server Setup (https://github.com/ev3-showcase/api-server)
- type: Python:Flask

### create app
For this application we will use the source to image build process. With Python:36 there is an already existing image referenced, which does contain a predefined runtime environment for Python applications.
In addition it does contain a couple of scripts which helps it to treat the application as it is intended to. The sourcecode is taken from the sourcecoderepository we provided.
The buildprocess now does checkout the git repository, takes it's files and will build a new image with the builderimage as base and the application files within it's final location. In addition, it will execute a pip install for all packages included within the requirements.txt (contained in sourcecode repository)
Finally we only need to tell it, where to find the application file: `-e APP_FILE=api.py`
All information about that specific builderimage can be found under following link: [Python Builderimage](https://github.com/sclorg/s2i-python-container/blob/master/3.6/README.md)
```
oc new-app --name=api python:3.6~https://github.com/ev3-showcase/api-server#master \
-e MQTT_BROKER=message-broker \
-e MQTT_PORT=1883 \
-e APP_FILE=app.py
```

### cleanup
Just in Case, you want to remove the service again.
```
oc delete all -l app=api
```

## Front End Setup (https://github.com/ev3-showcase/frontend)
- type: Python:Django

### create app
This application is built the same way as the Flask api. But in this application we do not tell the builderimage how to start our application. 
Where flask does not have a specific way of starting a webserver, Django does. With this framework there is a standard way of running the server: `python manage.py runserver`
Thats one of a variety of framework specific patterns the Buildpack does look for. If it does find the manage.py file in the root project folder, it assumes it a Django application and will run it the way it is supposed to.
But we have to consider another Framework specific taks. Normally Django does crate it's database models on startup with `python manage.py migrate`. Here we are just running a stateless frontend application without any persistent data.
To prevent Django from creating a database, we need to tell the Builderimage to disable the database migration with: `-e DISABLE_MIGRATE=True`

```
oc new-app --name=frontend python:3.6~https://github.com/ev3-showcase/frontend#master \
-e API_SERVER=api \
-e API_PORT=8080 \
-e DISABLE_MIGRATE=True
```

### cleanup
Just in Case, you want to remove the service again.
```
oc delete all -l app=frontend
```


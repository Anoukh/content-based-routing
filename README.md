[![Build Status](https://travis-ci.org/ballerina-guides/content-based-routing.svg?branch=master)](https://travis-ci.org/ballerina-guides/content-based-routing)

# Content Based Routing

The Content-Based Router (CBR) reads the content of a message and routes it to a specific recipient based on its content. This approach is useful when an implementation of a specific logical function is distributed across multiple physical systems.

> This guide walks you through the process of implementing content-based routing using the Ballerina language.

This is a simple Ballerina code for content-based routing.

The following are the sections available in this guide.

- [What you'll build](#what-youll-build)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
- [Testing](#testing)
- [Deployment](#deployment)
- [Observability](#observability)

## What you’ll build

To understand how you can build a content-based routing system using Ballerina, let's consider a real-world use case of a company recruitment agency service that provides recruitment details of companies. When a company recruitment agency service sends a request that includes the company name (e.g., ABC Company), that particular request will be routed to its respective endpoint. After receiving the request from the content-based router (`company_recruitment_agency_service`), the relevant company's endpoint sends the response back to the caller. The following diagram illustrates this use case clearly.

![alt text](/images/content-based-routing.svg)


## Prerequisites
 
- [Ballerina Distribution](https://ballerina.io/learn/getting-started/)
- A Text Editor or an IDE 

### Optional Requirements
- Ballerina IDE plugins ([IntelliJ IDEA](https://plugins.jetbrains.com/plugin/9520-ballerina), [VSCode](https://marketplace.visualstudio.com/items?itemName=WSO2.Ballerina), [Atom](https://atom.io/packages/language-ballerina))
- [Docker](https://docs.docker.com/engine/installation/)
- [Kubernetes](https://kubernetes.io/docs/setup/)


## Implementation

> If you want to skip the basics, you can download the GitHub repo and directly move to the "Testing" section by skipping the "Implementation" section.   

### Create the project structure

Ballerina is a complete programming language that supports custom project structures. Use the following package structure for this guide.

```
content-based-routing
 └── guide
      └── company_data_service
           ├── company_data_service.bal 
      └── company_recruitment_agency_service  
	   ├──company_recruitment_agency_service.bal  
      └── tests
           ├──company_recruitment_agency_service_test.bal
```

Create the above directories in your local machine and also create empty `.bal` files. Open the terminal and navigate to `/content-based-routing/guide` and run Ballerina project initializing toolkit.

```bash
   $ ballerina init
```

### Developing the service
Let's look at the implementation of the `company_recruitment_agency_service`, which acts as the Content-Based Router.

Let's consider that a request comes to the company recruitment agency service with specific content. The `company_recruitment_agency_service` receives the request message, reads it, and routes the request to one of the recipients according to the message's content.

##### company_recruitment_agency_service.bal

```ballerina
import ballerina/http;
import ballerina/log;
import ballerina/mime;
import ballerina/io;

endpoint http:Listener comEP{
    port: 9091
};

//Client endpoint to communicate with company recruitment service
endpoint http:Client locationEP{
    url: "http://localhost:9090/companies"
};

//Service is invoked using basePath value "/checkVacancies"
@http:ServiceConfig{
    basePath: "/checkVacancies"
}

//"comapnyRecruitmentsAgency" routes requests to relevant endpoints and gets their responses.
service<http:Service> comapnyRecruitmentsAgency  bind comEP{

    // POST requests is directed to a specific company using, /checkVacancies/company.
    @http:ResourceConfig{
        methods: ["POST"],
        path: "/company"
    }
    comapnyRecruitmentsAgency(endpoint CompanyEP, http:Request req){
        //Get the JSON payload from the request message.
        var jsonMsg = req.getJsonPayload();
        
       //Parsing the JSON payload from the request
        match jsonMsg{
            json msg =>{
                //Get the string value relevant to the key `name`.
                string nameString;

                nameString = check <string>msg["Name"];

                //The HTTP response can be either error|empty|clientResponse
                (http:Response|error|()) clientResponse;

                if (nameString == "John and Brothers (pvt) Ltd"){
                    //Routes the payload to the relevant service.
                    clientResponse =
                    locationEP->get("/John-and-Brothers-(pvt)-Ltd");

                }else if(nameString == "ABC Company"){
                    clientResponse =
                    locationEP->get("/ABC-Company");

                }else if(nameString == "Smart Automobile"){
                    clientResponse =
                    locationEP->get("/Smart-Automobile");

                }else {

                    clientResponse = log:printError("Company Not Found!");
                }

                //Use respond() to send the client response back to the caller.
                //When the request is successful, the response is returned.
                //Sends back the clientResponse to the caller if no error is found.
                match clientResponse {
                    http:Response respone =>{
                        CompanyEP->respond(respone) but { error e =>
                        log:printError("Error sending response", err = e) };
                    }
                    error conError =>{
                        error err = {};
                        http:Response res = new;
                        res.statusCode = 500;
                        res.setPayload(err.message);
                        CompanyEP->respond(res) but { error e =>
                        log:printError("Error sending response", err = e) };
                    }
                    () => {}
                }
            }
            error err =>{
            //500 error response is constructed and sent back to the client.
                http:Response res = new;
                res.statusCode = 500;
                res.setPayload(err.message);
                CompanyEP->respond(res) but { error e =>
                log:printError("Error sending response", err = e) };
            }
        }
    }
}
```

Let's now look at `company_data_service`, which is responsible for communicating with all the company's endpoints.

#### company_data_service.bal

```ballerina
import ballerina/http;

endpoint http:Listener listener {
    port: 9090
};

// Company data management is done using an in memory map.
map<json> companyDataMap;


// RESTful service.
@http:ServiceConfig { basePath: "/companies" }
service<http:Service> orderMgt bind listener {

    // Resource that handles the HTTP GET requests that are directed to specific
    // company data using path '/John-and-Brothers-(pvt)-Ltd'
    @http:ResourceConfig {
        methods: ["GET"],
        path: "/John-and-Brothers-(pvt)-Ltd"
    }
    findJohnAndBrothersPvtLtd(endpoint client, http:Request req) {
        json? payload = {
            Name: "John and Brothers (pvt) Ltd",
            Total_number_of_Vacancies: 12,
            Available_job_roles : "Senior Software Engineer = 3 ,Marketing Executives =5 Management Trainees=4",
            CV_Closing_Date: "17/06/2018" ,
            ContactNo: 01123456 ,
            Email_Address: "careersjohn@jbrothers.com"
        };

        http:Response response;
        if (payload == null) {
            payload = "Data : 'John-and-Brothers-(pvt)-Ltd' cannot be found.";
        }

        // Set the JSON payload in the outgoing response message.
        response.setJsonPayload(payload);

        // Send response to the client.
        _ = client->respond(response);
    }

    // Resource that handles the HTTP GET requests that are directed to specific
    // company data using path '/ABC-Company'
    @http:ResourceConfig {
        methods: ["GET"],
        path: "/ABC-Company"
    }
    findAbcCompany(endpoint client, http:Request req) {
        json? payload = {
            Name:"ABC Company",
            Total_number_of_Vacancies: 10,
            Available_job_roles : "Senior Finance Manager = 2 ,Marketing Executives =6 HR Manager=2",
            CV_Closing_Date: "20/07/2018" ,
            ContactNo: 0112774 ,
            Email_Address: "careers@abc.com"
        };

        http:Response response;
        if (payload == null) {
            payload = "Data : 'ABC-Company' cannot be found.";
        }

        // Set the JSON payload in the outgoing response message.
        response.setJsonPayload(payload);

        // Send response to the client.
        _ = client->respond(response);
    }

    // Resource that handles the HTTP GET requests that are directed to specific
    // company data using path '/Smart-Automobile'
    @http:ResourceConfig {
        methods: ["GET"],
        path: "/Smart-Automobile"
    }
    findSmartAutomobile(endpoint client, http:Request req) {
        json? payload = {
            Name:"Smart Automobile",
            Total_number_of_Vacancies: 11,
            Available_job_roles : "Senior Finance Manager = 2 ,Marketing Executives =6 HR Manager=3",
            CV_Closing_Date: "20/07/2018" ,
            ContactNo: 0112774 ,
            Email_Address: "careers@smart.com"
        };

        http:Response response;
        if (payload == null) {
            payload = "Data : 'Smart-Automobile' cannot be found.";
        }

        // Set the JSON payload in the outgoing response message.
        response.setJsonPayload(payload);

        // Send response to the client.
        _ = client->respond(response);
    }
}
```

- According to the code implementation, `company_recruitment_agency_service` checks the request content and routes it to `company_data_service`.

- In the above implementation, `company_recruitment_agency_service` reads the request's JSON content ("Name") using `nameString` and sends the request to the relevant company. This is a resource that handles the HTTP POST requests that are directed to a specific company using `/checkVacancies/company`.

- After receiving the request from the content-based router (`company_recruitment_agency_service`), the `company_data_service` sends the relevant response back to the caller.

## Testing 

### Invoking the service

You can run the `company_recruitment_agency_service` that you developed above in your local environment. Open your command line and navigate to `guide/`, and execute the following command.

```
$ ballerina run company_data_service/
$ ballerina run company_recruitment_agency_service/
```

You can test the functionality of the `company_recruitment_agency_service` by sending an HTTP POST request. For example, we have used cURL commands to test each routing operation of `company_recruitment_agency_service` as follows.

**Route the request when "Name"="John and Brothers (pvt) Ltd"** 

```bash
 $ curl -v http://localhost:9091/checkVacancies/company -d'{"Name" :"John and Brothers (pvt) Ltd"}' -H "Content-Type:application/json"
  
 Output : 
  
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 9091 (#0)
> POST /checkVacancies/company HTTP/1.1
> Host: localhost:9090
> User-Agent: curl/7.47.0
> Accept: */*
> Content-Type:application/json
> Content-Length: 40
> 

* upload completely sent off: 40 out of 40 bytes
< HTTP/1.1 200 OK
< Date: Mon, 11 Jun 2018 13:30:00 GMT
< Content-Type: application/json
< Via: 1.1 vegur
< server: Cowboy
< content-length: 356

{
    "Name": "John and Brothers (pvt) Ltd",
    "Total_number_of_Vacancies": 12,
    "Available_job_roles" : "Senior Software Engineer = 3 ,Marketing Executives =5 Management Trainees=4",
    "CV_Closing_Date": "17/06/2018" ,
    "ContactNo": 1123456 ,
    "Email_Address": "careersjohn@jbrothers.com"
}
```

**Route the request when "Name"="ABC Company"**

```bash
$ curl -v http://localhost:9091/checkVacancies/company -d '{"Name" : "ABC Company"}' -H "Content-Type:application/json"

Output : 

*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 9091 (#0)
> POST /checkVacancies/company HTTP/1.1
> Host: localhost:9090
> User-Agent: curl/7.47.0
> Accept: */*
> Content-Type:application/json
> Content-Length: 22

* upload completely sent off: 40 out of 40 bytes
< HTTP/1.1 200 OK
< Date: Mon, 11 Jun 2018 13:30:00 GMT
< Content-Type: application/json
< Via: 1.1 vegur
< server: Cowboy
< content-length: 308

{
    "Name":"ABC Company",
    "Total_number_of_Vacancies": 10,
    "Available_job_roles" : "Senior Finance Manager = 2 ,Marketing Executives =6 HR Manager=2",
    "CV_Closing_Date": "20/07/2018" ,
    "ContactNo": 112774 ,
    "Email_Address":"careers@abc.com"
      
 }

```

**Route the request when "Name"="Smart Automobile"**

```bash
$ curl -v http://localhost:9091/checkVacancies/company -d '{"Name" : "Smart Automobile"}' -H "Content-Type:application/json"

Output :

*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 9091 (#0)
> POST /checkVacancies/company HTTP/1.1
> Host: localhost:9090
> User-Agent: curl/7.47.0
> Accept: */*
> Content-Type:application/json
> Content-Length: 29

* upload completely sent off: 29 out of 29 bytes
< HTTP/1.1 200 OK
< Date: Mon, 11 Jun 2018 12:27:45 GMT
< Content-Type: application/json
< Via: 1.1 vegur
< server: Cowboy
< content-length: 315

{
    "Name":"Smart Automobile",
    "Total_number_of_Vacancies": 11,
    "Available_job_roles" : "Senior Finance Manager = 2 ,Marketing Executives =6 HR Manager=3",
    "CV_Closing_Date": "20/07/2018" ,
    "ContactNo": 112774 ,
    "Email_Address": "careers@smart.com"
    
}
```

### Writing unit tests 

In Ballerina, the unit test cases should be in the same package inside a folder named as 'tests'.  When writing the test functions the below convention should be followed.

Test functions should be annotated with `@test:Config`. See the following example.

```ballerina
   @test:Config
   company_recruitment_agency_service) {
```
  
This guide contains unit test cases for each resource available in the `company_recruitment_agency_service` implemented above. 

To run the unit tests, open your command line and navigate to `/content-based-routing/guide`, and run the following command.

```bash
   $ ballerina test
```


## Deployment

Once you are done with the development, you can deploy the service using any of the methods that we listed below. 

### Deploying locally

As the first step, you can build a Ballerina executable archive (.balx) of the service that you developed above. Navigate to `/content-based-routing/guide` and run the following command. 

```bash
   $ ballerina build company_recruitment_agency_service
```

Once the `company_recruitment_agency_service.balx` file is created inside the target folder, you can run that with the following command. 

```bash
   $ ballerina run target/company_recruitment_agency_service.balx
```

The successful execution of the service will show us the following output. 

```
   ballerina: initiating service(s) in 'target/company_recruitment_agency_service.balx'
	ballerina: started HTTP/WS endpoint 0.0.0.0:9091
	ballerina: started HTTP/WS endpoint 0.0.0.0:9090
```

### Deploying on Docker

You can run the service that we developed above as a Docker container. As the Ballerina platform includes a [Ballerina_Docker_Extension](https://github.com/ballerinax/docker), which offers native support for running ballerina programs on containers, you just need to put the corresponding Docker annotations on your service code. 

In the `company_recruitment_agency_service`, you need to import  `ballerinax/docker` and use the annotation `@docker:Config` as shown below to enable Docker image generation during the build time. 

##### company_recruitment_agency_service.bal

```ballerina

import ballerina/http;
import ballerinax/docker;

@docker:Config {
    registry:"ballerina.guides.io",
    name:"company_recruitment_agency_service",
    tag:"v1.0"
}

@docker:Expose {}


endpoint http:Listener comEP{
    port: 9091
};

//Client endpoint to communicate with company recruitment service
endpoint http:Client locationEP{
    url: "http://localhost:9090/companies"
};

//Service is invoked using basePath value "/checkVacancies"
@http:ServiceConfig{
    basePath: "/checkVacancies"
}

service<http:Service> comapnyRecruitmentsAgency  bind comEP {

```

The `@docker:Config` annotation is used to provide the basic Docker image configurations for the sample. `@docker:Expose {}` is used to expose the port. 

Now you can build a Ballerina executable archive (.balx) of the service that we developed above, using the following command. This will also create the corresponding docker image using the docker annotations that you have configured above. Navigate to `/content-based-routing/guide` and run the following command.  

```
   $ ballerina build company_recruitment_agency_service
   
    @docker                  - complete 3/3 

   Run following command to start docker container: 
   docker run -d -p 9091:9091 ballerina.guides.io/company_recruitment_agency_service:v1.0
```

- Once you successfully build the Docker image, you can run it with the `docker run` command that is shown in the previous step.  

```bash   
   $ docker run -d -p 9091:9091 ballerina.guides.io/company_recruitment_agency_service:v1.0
```

Here you can run the Docker image with flag `-p <host_port>:<container_port>` so that you use the host port 9090 and the container port 9090. Therefore, you can access the service through the host port. 

Verify if the Docker container is running with the use of `$ docker ps`. The status of the Docker container should be shown as 'Up'. 

You can access the service using the same cURL commands that you used above. 

**Request when "Name"="John and Brothers (pvt) Ltd"**

```bash
    $ curl -v http://localhost:9091/checkVacancies/company -d '{"Name" :"John and Brothers (pvt) Ltd"}' -H "Content- Type:application/json"
    
```

**Request when "Name"="ABC Company"**

```bash
    $ curl -v http://localhost:9091/checkVacancies/company -d '{"Name" :"ABC Company"}' -H "Content- Type:application/json"
```

**Request when "Name"="Smart Automobile**

```bash
    $ curl -v http://localhost:9091/checkVacancies/company -d '{"Name" :"Smart Automobile"}' -H "Content- Type:application/json"
```

### Deploying on Kubernetes

You can run the service that you developed above on Kubernetes. The Ballerina language offers native support for running Ballerina programs on Kubernetes with the use of Kubernetes annotations you can include as part of your service code. Also, it will take care of the creation of the Docker images. So you don't need to explicitly create Docker images prior to deploying it on Kubernetes. 

Refer to [Ballerina_Kubernetes_Extension](https://github.com/ballerinax/kubernetes) for more details and samples on Kubernetes deployment with Ballerina. You can also find details on using Minikube to deploy Ballerina programs. 

Let's now see how we can deploy our `company_recruitment_agency_service` on Kubernetes.

First you need to import `ballerinax/kubernetes` and use `@kubernetes` annotations as shown below to enable Kubernetes deployment for the service you developed above. 

##### company_recruitment_agency_service.bal

```ballerina
import ballerina/http;
import ballerinax/kubernetes;

@kubernetes:Ingress {
    hostname:"ballerina.guides.io",
    name:"ballerina-guides-company_recruitment_agency_service",
    path:"/"
}

@kubernetes:Service {
    serviceType:"NodePort",
    name:"ballerina-guides-company_recruitment_agency_service"
}

@kubernetes:Deployment {
    image:"ballerina.guides.io/company_recruitment_agency_service:v1.0",
    name:"ballerina-guides-company_recruitment_agency_service"
}

endpoint http:Listener comEP{
    port: 9091
};

//Client endpoint to communicate with company recruitment service
endpoint http:Client locationEP{
    url: "http://localhost:9090/companies"
};

//Service is invoked using basePath value "/checkVacancies"
@http:ServiceConfig{
    basePath: "/checkVacancies"
}

//comapnyRecruitmentsAgency service to route each request to relevent endpoints and get their responses.
service<http:Service> comapnyRecruitmentsAgency  bind comEP {

``` 

Here you have used `@kubernetes:Deployment` to specify the Docker image name that will be created as part of building this service. 

You have also specified `@kubernetes:Service` so that it creates a Kubernetes service that exposes the Ballerina service that is running on a Pod.  

Additionally, you have used `@kubernetes:Ingress` which is the external interface to access your service (with path `/` and host name `ballerina.guides.io`).

Now you can build a Ballerina executable archive (.balx) of the service that you developed above using the following command. This will also create the corresponding Docker image and the Kubernetes artifacts using the Kubernetes annotations that you have configured above.
  
```
   $ ballerina build company_recruitment_agency_service
   
   @kubernetes:Service                      - complete 1/1
   @kubernetes:Ingress                      - complete 1/1
   @kubernetes:Docker                       - complete 3/3 
   @kubernetes:Deployment                   - complete 1/1
  
   Run following command to deploy kubernetes artifacts:  
   kubectl apply -f ./target/company_recruitment_agency_service/kubernetes
```

You can verify that the Docker image that you specified in `@kubernetes:Deployment` is created, by using `$ docker images`. 

Also the Kubernetes artifacts related your service will be generated in `./target/company_recruitment_agency_service/kubernetes`. 

Now you can create the Kubernetes deployment using:

```bash
   $ kubectl apply -f ./target/company_recruitment_agency_service/kubernetes 
 
   deployment.extensions "ballerina-guides-company_recruitment_agency_service" created
   ingress.extensions "ballerina-guides-company_recruitment_agency_service" created
   service "ballerina-guides-company_recruitment_agency_service" created
```

You can verify Kubernetes deployment, service and ingress are running properly, by using following Kubernetes commands.

```bash
   $ kubectl get service
   $ kubectl get deploy
   $ kubectl get pods
   $ kubectl get ingress
```

If everything is successfully deployed, you can invoke the service either via Node port or ingress. 

Node port:

**Request when "Name"="John and Brothers (pvt) Ltd"**

```bash
    $ curl -v http://localhost:<Node_Port>/checkVacancies/company -d '{"Name" :"John and Brothers (pvt) Ltd"}' -H "Content- Type:application/json"
```

**Request when "Name"="ABC Company"**

```bash
    $ curl -v http://localhost:<Node_Port>/checkVacancies/company -d '{"Name" :"ABC Company"}' -H "Content- Type:application/json"
```
**Request when "Name"="Smart Automobile**

```bash
    $ curl -v http://localhost:<Node_Port>/checkVacancies/company -d '{"Name" :"Smart Automobile"}' -H "Content- Type:application/json"
```

Ingress:

Add `/etc/hosts` entry to match hostname. 

``` 
   127.0.0.1 ballerina.guides.io
```

Access the service. 

**Request when "Name"="John and Brothers (pvt) Ltd"**

```bash
    $ curl -v http:/ballerina.guides.io/checkVacancies/company -d '{"Name" :"John and Brothers (pvt) Ltd"}' -H "Content- Type:application/json"
```

**Request when "Name"="ABC Company"**

```bash
    $ curl -v http:/ballerina.guides.io/checkVacancies/company -d '{"Name" :"ABC Company"}' -H "Content- Type:application/json"
```

**Request when "Name"="Smart Automobile**

```bash
    $ curl -v http://ballerina.guides.io/checkVacancies/company -d '{"Name" :"Smart Automobile"}' -H "Content- Type:application/json"
```

## Observability 

Ballerina is by default observable. Meaning you can easily observe your services, resources, etc. However, observability is disabled by default via configuration. Observability can be enabled by adding following configurations to `ballerina.conf` file in `/content-based-routing/guide`.

```ballerina
[b7a.observability]

[b7a.observability.metrics]
# Flag to enable Metrics
enabled=true

[b7a.observability.tracing]
# Flag to enable Tracing
enabled=true
```

> **NOTE**: The above configuration is the minimum configuration needed to enable tracing and metrics. With these configurations default values are loaded as the other configuration parameters of metrics and tracing.

### Tracing 

You can monitor Ballerina services using in built tracing capabilities of Ballerina. We'll use [Jaeger](https://github.com/jaegertracing/jaeger) as the distributed tracing system.

Follow the steps below to use tracing with Ballerina.

- You can add the following configurations for tracing. Note that these configurations are optional if you already have the basic configuration in `ballerina.conf` as described above.

```
   [b7a.observability]

   [b7a.observability.tracing]
   enabled=true
   name="jaeger"

   [b7a.observability.tracing.jaeger]
   reporter.hostname="localhost"
   reporter.port=5775
   sampler.param=1.0
   sampler.type="const"
   reporter.flush.interval.ms=2000
   reporter.log.spans=true
   reporter.max.buffer.spans=1000
```

- Run Jaeger docker image using the following command.

```bash
   $ docker run -d -p5775:5775/udp -p6831:6831/udp -p6832:6832/udp -p5778:5778 \
   -p16686:16686 -p14268:14268 jaegertracing/all-in-one:latest
```

- Navigate to `/content-based-routing/guide` and run the restful-service using following command.

```
   $ ballerina run company_recruitment_agency_service/
```

- Observe the tracing using Jaeger UI using following URL.

```
   http://localhost:16686
```

### Metrics
Metrics and alerts are built-in with Ballerina. We will use Prometheus as the monitoring tool. Follow the below steps to set up Prometheus and view metrics for Ballerina restful service.

- You can add the following configurations for metrics. Note that these configurations are optional if you already have the basic configuration in `ballerina.conf` as described under `Observability` section.

```ballerina
   [b7a.observability.metrics]
   enabled=true
   provider="micrometer"

   [b7a.observability.metrics.micrometer]
   registry.name="prometheus"

   [b7a.observability.metrics.prometheus]
   port=9700
   hostname="0.0.0.0"
   descriptions=false
   step="PT1M"
```

- Create a file `prometheus.yml` inside `/tmp/` location. Add the below configurations to the `prometheus.yml` file.

```
   global:
     scrape_interval:     15s
     evaluation_interval: 15s

   scrape_configs:
     - job_name: prometheus
       static_configs:
         - targets: ['172.17.0.1:9797']
```

> **NOTE**: Replace `172.17.0.1` if your local Docker IP differs from `172.17.0.1`.
   
Run the Prometheus Docker image using the following command.

```
   $ docker run -p 19090:9090 -v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml \
   prom/prometheus
```
   
You can access Prometheus at the following URL.

```
   http://localhost:19090/
```

> **NOTE**: By default Ballerina has the following metrics for the HTTP server connector. You can enter the following expression in Prometheus UI.
-  http_requests_total
-  http_response_time

### Logging

Ballerina has a log package for logging to the console. You can import the `ballerina/log` package and start logging. The following section describes how to search, analyze, and visualize logs in real time using Elastic Stack.

Start the Ballerina Service with the following command from the `/content-based-routing/guide` directory.

```
   $ nohup ballerina runcompany_recruitment_agency_service/ &>> ballerina.log&
```

> **NOTE**: This will write the console log to the `ballerina.log` file in the `/content-based-routing/guide` directory.

Start Elasticsearch using the following command

```
   $ docker run -p 9200:9200 -p 9300:9300 -it -h elasticsearch --name \
   elasticsearch docker.elastic.co/elasticsearch/elasticsearch:6.2.2 
```

> **NOTE**: Linux users might need to run `sudo sysctl -w vm.max_map_count=262144` to increase `vm.max_map_count`.
   
Start the Kibana plugin for data visualization with Elasticsearch.

```
   $ docker run -p 5601:5601 -h kibana --name kibana --link \
   elasticsearch:elasticsearch docker.elastic.co/kibana/kibana:6.2.2     
```

Configure logstash to format the Ballerina logs.

i) Create a file named `logstash.conf` with the following content

```
input {  
 beats{ 
     port => 5044 
 }  
}

filter {  
 grok{  
     match => { 
	 "message" => "%{TIMESTAMP_ISO8601:date}%{SPACE}%{WORD:logLevel}%{SPACE}
	 \[%{GREEDYDATA:package}\]%{SPACE}\-%{SPACE}%{GREEDYDATA:logMessage}"
     }  
 }  
}   

output {  
 elasticsearch{  
     hosts => "elasticsearch:9200"  
     index => "store"  
     document_type => "store_logs"  
 }  
}  
```

ii) Save the above `logstash.conf` inside a directory named as `{SAMPLE_ROOT}\pipeline`
     
iii) Start the logstash container, replace the {SAMPLE_ROOT} with your directory name
     
```
$ docker run -h logstash --name logstash --link elasticsearch:elasticsearch \
-it --rm -v ~/{SAMPLE_ROOT}/pipeline:/usr/share/logstash/pipeline/ \
-p 5044:5044 docker.elastic.co/logstash/logstash:6.2.2
```
  
 - Configure filebeat to ship the ballerina logs
    
i) Create a file named `filebeat.yml` with the following content.

```
filebeat.prospectors:
- type: log
  paths:
    - /usr/share/filebeat/ballerina.log
output.logstash:
  hosts: ["logstash:5044"]  
```

> **NOTE**: Modify the ownership of filebeat.yml file using `$chmod go-w filebeat.yml` 

ii) Save the above `filebeat.yml` inside a directory named as `{SAMPLE_ROOT}\filebeat`.
        
iii) Start the logstash container, replace the {SAMPLE_ROOT} with your directory name.
     
```
$ docker run -v {SAMPLE_ROOT}/filbeat/filebeat.yml:/usr/share/filebeat/filebeat.yml \
-v {SAMPLE_ROOT}/guide/company_recruitment_agency_service/ballerina.log:/usr/share\
/filebeat/ballerina.log --link logstash:logstash docker.elastic.co/beats/filebeat:6.2.2
```
 
 - Access Kibana to visualize the logs using the following URL:
 
```
   http://localhost:5601 
```
  
 

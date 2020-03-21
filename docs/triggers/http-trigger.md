# Kubeless, KNative & REST API Trigger

Argo Events offers HTTP trigger which can easily invoke serverless functions like Kubeless, Knative, Nuclio and make REST API calls.

<br/>
<br/>

<p align="center">
  <img src="https://github.com/argoproj/argo-events/blob/http-trigger/docs/assets/http-trigger.png?raw=true" alt="HTTP Trigger"/>
</p>

<br/>
<br/>

## REST API Calls

Consider a scenario where your REST API server needs to consume events from event-sources S3, GitHub, SQS etc. Usually, you'd end up writing
the integration yourself in the server code, although server logic has nothing to do any of the event-sources. This is where Argo Events HTTP trigger
can hel. The HTTP trigger takes the task of consuming events from event-sources away from API server and seamlessly integrates these events via REST API calls.


We will set up a basic go http server and connect it with the minio events.

1. The HTTP server simply prints the request body as follows,

        package main
        
        import (
        	"fmt"
        	"io/ioutil"
        	"net/http"
        )
        
        func hello(w http.ResponseWriter, req *http.Request) {
        	body, err := ioutil.ReadAll(req.Body)
        	if err != nil {
        		fmt.Printf("%+v\n", err)
        		return
        	}
        	fmt.Println(string(body))
        	fmt.Fprintf(w, "hello\n")
        }
        
        func main() {
        	http.HandleFunc("/hello", hello)
        	fmt.Println("server is listening on 8090")
        	http.ListenAndServe(":8090", nil)
        }

2. Deploy the HTTP server,

        kubectl -n argo-events apply -f https://raw.githubusercontent.com/argoproj/argo-events/master/examples/tutorials/09-http-trigger/http-server.yaml

3. Create a service to expose the http server

        kubectl -n argo-events apply -f https://raw.githubusercontent.com/argoproj/argo-events/master/examples/tutorials/09-http-trigger/http-server-svc.yaml

4. Either use Ingress, OpenShift Route or port-forwarding to expose the http server..

        kubectl -n argo-events port-forward <http-server-pod-name> 8090:8090

5. Our goals is to seamlessly integrate Minio S3 bucket notifications with REST API server created in previous step. So,
   lets set up the Minio Gateway and EventSource available [here](https://argoproj.github.io/argo-events/setup/minio/).
   Don't create the sensor as we will be deploying it in next step.

6. Create a sensor as follows,

        kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/master/examples/sensors/http-trigger.yaml

7. Now, drop a file onto `input` bucket in Minio server.

8. The sensor has triggered a http request to out basic go http server. Take a look at the logs

        server is listening on 8090
        {"type":"minio","bucket":"input"}

9. Great!!!

### Request Payload

In order to construct a request payload based on the event data, sensor offers 
`payload` field as a part of the HTP trigger.

Let's examine a HTTP trigger,

        http:
          serverURL: http://http-server.argo-events.svc:8090/hello
          payload:
            - src:
                dependencyName: test-dep
                dataKey: s3.bucket.name
              dest: bucket
            - src:
                dependencyName: test-dep
                contextKey: type
              dest: type
          method: POST  // GET, DELETE, POST, PUT, HEAD, etc.

The `payload` contains the list of `src` which refers to the source event and `dest` which refers to destination key within result request payload.

The `payload` declared above will generate a request payload like below,

        {
          "type": "type of event from event's context"
          "bucket": "bucket name from event data"
        }

The above payload will be passed in the HTTP request. You can add however many number of `src` and `dest` under `payload`. 

**Note**: Take a look at [Parameterization](https://argoproj.github.io/argo-events/tutorials/02-parameterization/) in order to understand how to extract particular key-value from
event data.
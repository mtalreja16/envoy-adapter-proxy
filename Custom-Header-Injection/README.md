<!-- Output copied to clipboard! -->


**Use-Case**

Apigee’s **Developer** and **App** entities have capability to attach Key-Value pairs i.e. custom attributes, this is often used to store additional metadata used by the Apigee runtime or/and by the backend. For example, you can add type, company, location, scopes, rate, etc. as key/value pairs.  This metadata can further be used for fine grained access control in the backend or routing the request within the microservices, etc.  However when using the Apigee Adapter for Envoy , it is a little tricky to pass the custom attribute(s) to your backend. Below is a sample response and you can find more info in the [docs](https://cloud.google.com/apigee/docs/api-platform/envoy-adapter/v2.0.x/example-apigee#test-the-installation).

```

{

 "headers": {

   "Accept": "*/*",

   "Host": "httpbin.org",

   "User-Agent": "curl/7.79.1",

   "X-Amzn-Trace-Id": "Root=1-63a2a6de-47d2610c438b2fb102e86cdd",

   "X-Api-Key": "key……oxoxoxoxo",

   "X-Apigee-Accesstoken": "",

   "X-Apigee-Api": "httpbin.org",

   "X-Apigee-Apiproducts": "httpbin-product",

   "X-Apigee-Application": "envoy-test",

   "X-Apigee-Authorized": "true",

   "X-Apigee-Clientid": "key……oxoxoxoxo",

   "X-Apigee-Developeremail": "",

   "X-Apigee-Environment": "eval",

   "X-Apigee-Organization": "org-name",

   "X-Apigee-Scope": "",

   "X-Envoy-Expected-Rq-Timeout-Ms": "15000"

 }

}

```

This article demonstrates a custom solution using** [Lua Scripts within Envoy filters](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/lua_filter#config-http-filters-lua)** to inject the custom attributes  in the generated JWT token from the remote-token proxy, 

To install Apigee Adapter for Envoy, you can follow the steps provided here: [https://cloud.google.com/apigee/docs/api-platform/envoy-adapter/latest/example-apigee](https://cloud.google.com/apigee/docs/api-platform/envoy-adapter/latest/example-apigee)

When you finish the installation, 2 proxies will be deployed to Apigee, called 



* remote-token
* remote-service

Now that the installation is complete, lets go over the steps needed for our use case :

**Step 1) Update remote-token API Proxy to embed custom Attributes in JWT Token**

Modify the **remote-token** proxy - > under the **ObtainAccessToken** flow 



* First add the **AccessEntity**  policy to retrieve the App OR Developer

    [https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/remote-token/policies/Access-Developer-Info.xml](https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/remote-token/policies/Access-Developer-Info.xml)


    ```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<AccessEntity async="false" continueOnError="false" enabled="true" name="Access-Developer-Info">
   <DisplayName>Access Developer Info</DisplayName>
   <FaultRules/>
   <Properties/>
   <EntityIdentifier ref="apikey" type="consumerkey"/>
   <EntityType value="developer"/>
</AccessEntity>
```


* Access-Developer-Info variable will return an output in XML format which needs to be converted to JSON using the XMLToJSON policy:

    [https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/remote-token/policies/Developers-to-JSON.xml](https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/remote-token/policies/Developers-to-JSON.xml)


    ```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<XMLToJSON async="false" continueOnError="true" enabled="true" name="Developers-to-JSON">
   <DisplayName>Developer Attributes to JSON</DisplayName>
   <FaultRules/>
   <Properties/>
   <OutputVariable>developerattributes</OutputVariable>
<Source>AccessEntity.ChildNodes.Access-Developer-Info.Developer.Attributes</Source>
   <Options>
       <TreatAsArray>
           <Path>Attributes/Attribute</Path>
       </TreatAsArray>
   </Options>
</XMLToJSON>
```


* The resulting JSON will include some extra attributes, that can be removed using JavaScript policy	

    [https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/remote-token/resources/jsc/set-jwt-variables.js](https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/remote-token/resources/jsc/set-jwt-variables.js)


    ```
var devAttributes=JSON.parse(context.getVariable("developerattributes"));
var devatt = {};
devAttributes.Attributes.Attribute.forEach(element => {
    devatt[element.Name] = element.Value;
});
context.setVariable("developerattributes", devatt);
```


* Now we have the “developerattributes” as a ** flow variable **which can be used within the “**Generate Access Token” **policy to set the claims:

    [https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/remote-token/policies/Generate-Access-Token.xml](https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/remote-token/policies/Generate-Access-Token.xml)


    ```
<AdditionalClaims>
   <Claim name="client_id" ref="apigee.client_id"/>
   <Claim name="access_token" ref="apigee.access_token"/>
   <Claim name="api_product_list" ref="apiProductList" type="string" array="true"/>
   <Claim name="application_name" ref="apigee.developer.app.name"/>
   <Claim name="developer_email" ref="apigee.developer.email"/>
   <Claim name="developer_attributes" ref="developerattributes" type="string"/>
   <Claim name="app_attributes" ref="appattributes" type="string"/>
   <Claim name="scope" ref="scope"/>
</AdditionalClaims>
```



Add your Developer and App AccessEntity policy before AccessTokenPolicy, like shown below, and also update the JavaScript (filename) to add attributes into variable shown earlier, this will how your remote-token proxy should look like after attaching the policies.


![alt_text](images/image1.png "image_tooltip")




* Deploy remote-token proxy after making all these changes..

**Step 2) Make a call to remote-token service to get a JWT Token, and Verify the Attributes in the response payload**

In step 1, we made modifications to remote-token proxy to add the attributes as claims, the next step is to generate a  JWT Token:



* Let's make a call to the remote-token service with the client_id and client_secret to generate the JWT Token, here is what the command will look like

    ```
curl https://domain.net/remote-token/token -d '{"client_id":"your_client_id","client_secret":"your_client_secret","grant_type":"client_credentials"}' -H "Content-type: application/json"
```



    This will return a JWT Token. If you decode this on [jwt.io](jwt.io), you will see information like this:


    ```
{
 "api_product_list": [
   "httpbin-product"
 ],
 "iss": "https://domain.net/remote-token/token",
 "developer_attributes": "{\"color\": \"red\", \"location\":\"Europe\"}",
 "client_id": "*************************************",
 "access_token": "It84nAUxF9ghXrNIXZyfqxqeFzmD",
 "aud": "remote-service-client",
 "application_name": "envoy-test",
 "nbf": 1671585877,
 "app_attributes": "{\"tier\":\"standard\"}",
 "developer_email": "******@example.com",
 "scope": "",
 "exp": 1671586777,
 "iat": 1671585877,
 "jti": "dada3e44-5115-4f35-b0f8-67de22b31031"
}
```



**Step 3) Update the envoy config and envoy adapter config to inject attributes as a header to an incoming  request.**

Now let's configure our Envoy config and envoy adapter config to include the Lua filter, so the attributes can be injected:



* Open the /samples/envoy-config.yaml in your working directory and
* Add `payload_in_metadata: jwt_payload`
* Add a new filter (see below) `envoy.filters.http.lua`  

        In the code below, we are adding a Lua filter using **inline_code** and it will be treated as a Global script. Envoy will execute this Global script for every http request. It will inspect the jwt payload, loop through all the attributes and extract “app_attributes” and , “developer_attributes” attributes and inject them as a custom header to the incoming http request which will be then passed to the backend. The sample of the file can be found in the below link.


        [https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/configs/samples/envoy-config.yaml](https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/configs/samples/envoy-config.yaml)


        ```
# evaluate JWT tokens, allow_missing allows API Key also
- name: envoy.filters.http.jwt_authn
 typed_config:
   "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
   providers:
     apigee:
       payload_in_metadata: jwt_payload
       #header_in_metadata: jwt_payload
   rules:
   - match:
       prefix: /
     requires:
       requires_any:
         requirements:
         - provider_name: apigee
         - allow_missing: {}
- name: envoy.filters.http.lua
 typed_config:
   "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
   inline_code: |
     function envoy_on_request(request_handle)
       local meta = request_handle:streamInfo():dynamicMetadata()
       for key, value in pairs(meta) do
         request_handle:logInfo(value.jwt_payload.developer_attributes)
         request_handle:headers():remove("developer_attributes")
         request_handle:headers():add("developer_attributes", value.jwt_payload.developer_attributes)
         request_handle:headers():remove("app_attributes")
         request_handle:headers():add("app_attributes", value.jwt_payload.app_attributes)
       end
     end
```


* Update apigee-adapter-service-cli/config.yaml and set auth section with  `jwt_provider_key: jwt_payload`	[https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/configs/apigee-remote-service-cli/config.yaml](https://github.com/mtalreja16/envoy-adapter-samples/blob/main/Custom-Header-Injection/configs/apigee-remote-service-cli/config.yaml)

    ```
config.yaml: |
   tenant:
     remote_service_api: https://domain-url/remote-service
     org_name: org-name
     env_name: eval
   analytics:
     collection_interval: 10s
   auth:
     jwt_provider_key: jwt_payload
     append_metadata_headers: true

```


* Restart both envoy and apigee-remote-service-envoy with updated configuration.

**Step 4) Make a call to the backend with a Bearer token.**

Finally, when you make a call to your backend with JWT Token you got from step 2


```
curl -H "Authorization: Bearer <token>" http://localhost:8080/headers -H "Host:  httpbin.org" -v
```


You will see the response contains developer and app info as custom headers, like below


```
{
 "headers": {
   "Accept": "*/*",
   "App-Attributes": "{\"tier\":\"standard\"}",
   "Developer-Attributes": "{\"color\":\"red\"},{\"location\":\"Europe\"}",
   "Host": "httpbin.org",
   "User-Agent": "curl/7.79.1",
   "X-Amzn-Trace-Id": "Root=1-63a2ab0b-72f329d85568c87606b48d98",
   "X-Apigee-Accesstoken": "token",
   "X-Apigee-Api": "httpbin.org",
   "X-Apigee-Apiproducts": "httpbin-product",
   "X-Apigee-Application": "envoy-test",
   "X-Apigee-Authorized": "true",
   "X-Apigee-Clientid": "client_id",
   "X-Apigee-Developeremail": "*******@example.com",
   "X-Apigee-Environment": "eval",
   "X-Apigee-Organization": "<org-name>",
   "X-Apigee-Scope": "",
   "X-Envoy-Expected-Rq-Timeout-Ms": "15000"
 }
}
```


You can find the full source code of proxy and envoy and envoy-adapter config [here](https://github.com/mtalreja16/envoy-adapter-proxy)

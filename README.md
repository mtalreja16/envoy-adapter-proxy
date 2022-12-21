<!-- Output copied to clipboard! -->

<!-- Yay, no errors, warnings, or alerts! -->

**Purpose of this article**

This article shows how to use Envoy filters within the Apigee Adapter for Envoy to inject custom headers based on the custom key/value pair(s) associated with Apigee’s **Developer** and **App** Entities.

**Use-Case**

Apigee’s **Developer** and **App** entities have capability to attach Key-Value pairs i.e. custom attributes, this is often used to store additional metadata  used by the  backend. For example, you can add location, scopes, rate, etc as key/value pairs.  This metadata can further be used for fine grained access control in the backend or routing the request within the microservices, etc.  However when using the Apigee Adapter for Envoy, it is a little tricky to pass it to your backend, as the Apigee Adapter for Envoy strips that data from the response. Below is a sample response and you can find more info in our [docs](https://cloud.google.com/apigee/docs/api-platform/envoy-adapter/v2.0.x/example-apigee#test-the-installation)


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

When you finish the installation, you will have deployed  2 proxies to Apigee, called 



* [remote-token ](https://github.com/mtalreja16/envoy-adapter-proxy/tree/main/remote-token)
* [remote-service](https://github.com/mtalreja16/envoy-adapter-proxy/tree/main/remote-service)

Now that the installation is complete, lets go over the steps needed for our use case :

**Step 1)**

Modify the **remote-token** proxy - > under the **ObtainAccessToken** flow 



* First add the **AccessEntity**  policy to retrieve the App OR Developer

    [https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/remote-token/policies/Access-Developer-Info.xml](https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/remote-token/policies/Access-Developer-Info.xml)


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


 



* `"Access-Developer-Info"` will return XML output, the  desirable state is turn this into JSON type, for that we will use XMLtoJSON Policy like below:

    [https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/remote-token/policies/Developers-to-JSON.xml](https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/remote-token/policies/Developers-to-JSON.xml)


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


* Since returned Json will not be in ideal format and will bring some extra attributes, we are going to clean this up using javascript process step, which will be like this:	[https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/remote-token/resources/jsc/set-jwt-variables.js](https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/remote-token/resources/jsc/set-jwt-variables.js)

    ```
    var developerattributes = context.getVariable("developerattributes");
    var devatt = [];
    if(developerattributes !== null){
     var obj = JSON.parse(developerattributes).Attributes.Attribute;
     if(obj !== null)
           obj.forEach(x => {
             var keyvalpair = {};
             keyvalpair[x.Name] = x.Value;
             devatt.push(keyvalpair);
           });
    context.setVariable("developerattributes", JSON.stringify(devatt));
    }
    ```


* Now we have a developerattributes as a **outputvariable**, we can use this variable to “**Generate Access Token” **policy set the claims:

    [https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/remote-token/policies/Generate-Access-Token.xml](https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/remote-token/policies/Generate-Access-Token.xml)


    ```
       <AdditionalClaims>
           <Claim name="client_id" ref="apigee.client_id"/>
           <Claim name="access_token" ref="apigee.access_token"/>
           <Claim name="api_product_list" ref="apiProductList" type="string" array="true"/>
           <Claim name="application_name" ref="apigee.developer.app.name"/>
           <Claim name="developer_email" ref="apigee.developer.email"/>
           <Claim name="developer_attributes" ref="developerattributes"/>
           <Claim name="app_attributes" ref="appattributes"/>
           <Claim name="scope" ref="scope"/>
       </AdditionalClaims>
    ```


* Deploy remote-token proxy after making all these changes..

**Step 2)**

In step 1, we made modifications to remote-token proxy, next step is to get those attributes in our JWT Token:



* Let's  make a call remote-token service with client_id and client_secret to generate the JWT Token, here is what command will look like

    ```
    curl https://domain.net/remote-token/token -d '{"client_id":"your_client_id","client_secret":"your_client_secret","grant_type":"client_credentials"}' -H "Content-type: application/json"
    ```



    This will return a JWT Token. If you decode this on jwt.io, you  will see information like this:


    ```
    {
     "api_product_list": [
       "httpbin-product"
     ],
     "iss": "https://domain.net/remote-token/token",
     "developer_attributes": "[{\"color\":\"red\"},{\"location\":\"Europe\"}]",
     "client_id": "*************************************",
     "access_token": "It84nAUxF9ghXrNIXZyfqxqeFzmD",
     "aud": "remote-service-client",
     "application_name": "envoy-test",
     "nbf": 1671585877,
     "app_attributes": "[{\"tier\":\"standard\"}]",
     "developer_email": "******@gmail.com",
     "scope": "",
     "exp": 1671586777,
     "iat": 1671585877,
     "jti": "dada3e44-5115-4f35-b0f8-67de22b31031"
    }

    ```


**Step 3)**

Now let's set up our Envoy config and envoy adapter config, so our attribute can be injected in when actual call made to backend:



* Open /samples/envoy-config.yaml and
* add `payload_in_metadata: jwt_payload`
* add new filter `envoy.filters.http.lua` : We are writing Lua filter as **inline_code** and it will be treated as a Global script. Envoy will execute this Global script for every http request, will inspect jwt payload, loop through all the properties and extract “app_attributes” and , “dev_attributes” properties and inject them as a custom header to http request which will be then passed to backend. 
[https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/configs/samples/envoy-config.yaml](https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/configs/samples/envoy-config.yaml)


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


* Update apigee-adapter-service-cli/config.yaml and set auth section with  `jwt_provider_key: jwt_payload`	[https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/configs/apigee-remote-service-cli/config.yaml](https://github.com/mtalreja16/envoy-adapter-proxy/blob/main/configs/apigee-remote-service-cli/config.yaml)

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

**Step 4)**

Finally, when you make a call to your backend with JWT Token you got from step 2


```
    curl -H "Authorization: Bearer <token>" http://localhost:8080/headers -H "Host:  httpbin.org" -v
```



    You will see the response contains developer and app info as custom headers, like below


```
    {
     "headers": {
       "Accept": "*/*",
       "App-Attributes": "[{\"tier\":\"standard\"}]",
       "Developer-Attributes": "[{\"color\":\"red\"},{\"location\":\"Europe\"}]",
       "Host": "httpbin.org",
       "User-Agent": "curl/7.79.1",
       "X-Amzn-Trace-Id": "Root=1-63a2ab0b-72f329d85568c87606b48d98",
       "X-Apigee-Accesstoken": "token",
       "X-Apigee-Api": "httpbin.org",
       "X-Apigee-Apiproducts": "httpbin-product",
       "X-Apigee-Application": "envoy-test",
       "X-Apigee-Authorized": "true",
       "X-Apigee-Clientid": "client_id",
       "X-Apigee-Developeremail": "*******@gmail.com",
       "X-Apigee-Environment": "eval",
       "X-Apigee-Organization": "<org-name>",
       "X-Apigee-Scope": "",
       "X-Envoy-Expected-Rq-Timeout-Ms": "15000"
     }
    }
```

You can find the full source code of proxy and envoy and envoy-adapter config [here](https://github.com/mtalreja16/envoy-adapter-proxy)

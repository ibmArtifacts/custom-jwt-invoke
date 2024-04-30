# User-Defined Policy (UDP / Custom Policy): custom-jwt-invoke
This UDP is to provide an APIC API to call the backend with a JWT, rather than just Username/Password on the Invoke policy.  
For versions of APIC v10.x  
More details about user-defined policies (aka custom policies) may be found in the [IBM API Connect documentation: User-defined policies](https://www.ibm.com/docs/en/api-connect/10.0.5.x_lts?topic=constructs-user-defined-policies).  

## Problem Description
Today, there's only a basic authentication call from the APIC Invoke Policy to target systems.  
  
![image](https://user-images.githubusercontent.com/66093865/162876312-c5b0e20d-9569-4e20-9fe6-2fd8243d4bbf.png)  
  
This custom policy will enhance the Invoke Policy to be able to use a JWT to call target systems, allowing for the request from APIC to backend target systems to authenticate a JWT rather than Basic Authentication.
  
![image](https://user-images.githubusercontent.com/66093865/162878276-40077b41-de22-4948-9d64-1d9de2c25c57.png)  
    
## Designing the Policy Palette (look, feel, and input of the policy)  
Please familiarize yourself with the JWT-Auth-Invoke-policy.yaml in the [JWT-Auth-Invoke-policy.zip](https://github.com/ibmArtifacts/custom-jwt-invoke/blob/main/JWT-Auth-Invoke-policy_v1.0.2.zip) in this repo.  
  
The properties are set accordingly per diagram below for the user to input values during design time.  
![image](https://github.com/ibmArtifacts/custom-jwt-invoke/assets/66093865/25fcbd47-7606-41da-825f-3f201c74dbc2)  
    
During runtime, the property values will be assigned to the local.parameter() keys, respectively in memory of the gateway, and could be called upon within the Assembly section as shown in the diagram below:  

![image](https://user-images.githubusercontent.com/66093865/162886594-2fb7a95d-e61a-42ae-9465-d3510bbe5a3f.png)  
  
## Assembly  
The assembly section contains the policies similar to the OOTB policies encapsulated into this custom policy.  

1.	set-variable: takes the values input from the custom policy and set it to the value for each key. For example, the udp-audclaim that is set in the Properties section will take what the developer inputs, and, during runtime, value will be set to the $(local.paramter.udp-audclaim) in the header aud-claim. These are headers set for the Invoke policy.  

```  
    - set-variable:
        version: 2.0.0
        title: setVar for JWT-call
        actions:
          - set: message.headers.x-ibm-client-id
            value: $(local.parameter.udp-clientid)
            type: any
          - set: message.headers.x-ibm-client-secret
            value: $(local.parameter.udp-clientsecret)
            type: any
          - set: message.headers.aud-claim
            value: $(local.parameter.udp-audclaim)
            type: any
          - set: message.headers.content-type
            value: 'application/json'
            type: any
          - set: message.headers.accept
            value: 'application/json'
            type: any

```  
2. gatewayscript: gws:Logger – is not necessary to the functionality of the solution but it logs details for debugging. As you will notice the script only has console logging, which will output the system logs. You will see these outputs in the trace section in the test tool.  
``` 
    - gatewayscript:
        version: 2.0.0
        title: 'gws:Logger'
        source: >-
          var apim = require('apim'); 
          console.log('****Completed setVar for JWT-call****'); 
          console.debug('****local.parameter-OUTPUT: ' +
          JSON.stringify(apim.getvariable('local'))); 

          console.debug('****Setting the following clientid: ' +
          apim.getvariable('local.parameter.udp-clientid')); 
          console.debug('****Setting the following clientsecret: ' +
          apim.getvariable('local.parameter.udp-clientsecret')); 
          console.debug('****Setting the following aud-claim: ' +
          apim.getvariable('local.parameter.udp-audclaim')); 
          console.log('****Sending data for JWT****');
```  
3.	Invoke: invoke-JWT-gen – similar to the Invoke policy in the Assemble design, this will take the user input value from `$(local.parameter.udp-jwtgenurl)` and `$(local.paramter.udp-ttl)` reflected in the diagram below and source code.  

```  
    - invoke:
        title: invoke-JWT-gen
        version: 2.0.0
        verb: GET
        target-url: $(local.parameter.udp-jwtgenurl)
        follow-redirects: false
        timeout: 60
        parameter-control:
          type: blocklist
          values: []
        header-control:
          type: blocklist
          values: []
        inject-proxy-headers: false
        backend-type: json
        cache-response: time-to-live
        cache-ttl: $(local.parameter.udp-ttl)
        output: jwt-response
        stop-on-error: []
```  

4. gatewayscript: The last gatewayscript will log the jwt response from the Invoke in debug level logging, the last 5 characters of the jwt in normal level logging, and set the jwt to the authorization header to be used for the proceeding policy.  

```
    - gatewayscript:
        version: 2.0.0
        title: gatewayscript
        source: >+
          console.log('****Invoke to JWT Gen completed****');

          var apim = require('apim');
          
          console.debug('****Full response from JWT Gen: ' + JSON.stringify(apim.getvariable('jwt-response')));

          var vJWTResponse = (apim.getvariable('jwt-response').body).toString();
          
          console.debug('****jwt-response.body from JWT Gen invoke: ' + vJWTResponse);

          apim.setvariable('message.headers.authorization', vJWTResponse);
          var vObfuscatedJWT = vJWTResponse.substr(vJWTResponse.length - 5);
          console.log('****JWT set to Authorization header (showing-last-5-characters-only): ' + vObfuscatedJWT);

          var vJWTGenUrl = apim.getvariable('local.parameter.udp-jwtgenurl');

          console.log('****JWT-gen URL: ' + vJWTGenUrl);

          console.debug('****local output: ' + JSON.stringify(apim.getvariable('local')));
```  

5. Once authorizing the policy is completed, the yaml may be zipped up and uploaded to the gateway.  
![image](https://github.com/ibmArtifacts/custom-jwt-invoke/assets/66093865/7c0da680-e383-4d20-b3d1-12c9545fbffd)  

NOTE: When you use this custom-JWT-udp policy, you may get the following error:  
![image](https://github.com/ibmArtifacts/custom-jwt-invoke/assets/66093865/f8d24423-1aa2-47dd-bfbf-28aed753d183)  
  
This is complaining about the `cache-ttl: $(local.parameter.udp-ttl)` within the invoke policy because it expects a string/numeric value, thus the special characters triggers a schema validation error.  
**Run the following steps to avoid this issue**: Turn off the schema validation in the catalog by going to the Catalog Setttings > Publish validations > Edit > uncheck the `Validate custom policy schema in assembly` and **Save**.  
  
![image](https://github.com/ibmArtifacts/custom-jwt-invoke/assets/66093865/fe66f94a-949d-4f8a-9f8c-97e8a3baf347)  

## Testing  
This test will only test the generation of the JWT with the audience and issuer claim set to the token. The client id and secret validation are turned off for this test.  
Here is a jwt-generate api service that will be the JWT provider, which will be generating the JWT that ultimately is invoked to the backend.  
You may published this on APIC and use this for the UDP to call the service in order to get the JWT.  
[jwt-generate_1.0.0.yaml](https://github.com/ibmArtifacts/custom-jwt-invoke/blob/main/jwt-generate_1.0.0.yaml)  
  
### Creating an API with the UDP  
1. Create a new API with the UDP inputting the audience-claim required for the JWT (as shown in the diagram below). Do not apply any security schemes for this test.
![image](https://github.com/ibmArtifacts/custom-jwt-invoke/assets/66093865/ea18d723-313e-415b-bda4-f34870589f0c)

NOTE: You may get the jwt-generate api service endpoint from that apis **Endpoint** section:  
![image](https://github.com/ibmArtifacts/custom-jwt-invoke/assets/66093865/5eebd54a-fdec-44b6-9236-7e16f8a9ac59)  
  
2. Publish the API.  
NOTE: You will notice the following error below. You may ignore that, but please republish the api.  
![image](https://github.com/ibmArtifacts/custom-jwt-invoke/assets/66093865/3f09a6a9-82d3-4c46-8502-5ab0a495865d)  
  
3. Once published, go to the **Test** tab of the api and click **Send** to invoke the jwt-generate service.
At this time, the jwt-udp api you've just tested has called the jwt provider api to retrieve a token, to which then the JWT is assigned to the Authorization header for it to be avilable downstream in a backend call.
You will see from the diagram below that the header contains "authorization", which will flow down stream to the next policy, which can be an Invoke to a backend service that requires the JWT.  
![image](https://github.com/ibmArtifacts/custom-jwt-invoke/assets/66093865/dd8ae147-c4b0-4498-8466-83ccf7bb84bf)  








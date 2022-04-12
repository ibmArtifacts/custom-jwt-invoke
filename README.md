# User-Defined Policy (UDP / Custom Policy): custom-jwt-invoke
This UDP is to provide an APIC API to call the backend with a JWT, rather than just Username/Password on the Invoke policy.  
For versions of APIC v10.x  

## Problem Description
Today, there's only a basical authentication call from the APIC Invoke Policy to target systems.  
  
![image](https://user-images.githubusercontent.com/66093865/162876312-c5b0e20d-9569-4e20-9fe6-2fd8243d4bbf.png)  
  
This custom policy will enhance the Invoke Policy to be able to use a JWT to call target systems, allowing for the request from APIC to backend target systems to authenticate a JWT rather than Basic Authentication.
  
![image](https://user-images.githubusercontent.com/66093865/162878276-40077b41-de22-4948-9d64-1d9de2c25c57.png)  
    
## Designing the Policy Palette (look, feel, and input of the policy)  
    
The properties are set accordingly per diagram below for the user to input value during design time.  
![image](https://user-images.githubusercontent.com/66093865/162878580-0b92f9de-8955-4745-b5c0-5e0fe399276f.png)
  
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
3.	Invoke: invoke-JWT-gen – similar to the Invoke policy in the Assemble design, this will take the user input value from $(local.parameter.udp-jwtgenurl) and $(local.paramter.udp-ttl) reflected in the diagram below and source code.  

![image](https://user-images.githubusercontent.com/66093865/162898639-2335f5db-98a6-45cc-aebc-d6fc9cd2ff26.png)




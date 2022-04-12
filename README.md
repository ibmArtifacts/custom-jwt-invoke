# User-Defined Policy (UDP / Custom Policy): custom-jwt-invoke
This UDP is to provide an APIC API to call the backend with a JWT, rather than just Username/Password on the Invoke policy.  
For versions of APIC v10.x  

## Problem Description
Today, there's only a basical authentication call from the APIC Invoke Policy to target systems.  
  
![image](https://user-images.githubusercontent.com/66093865/162876312-c5b0e20d-9569-4e20-9fe6-2fd8243d4bbf.png)  
  
This custom policy will enhance the Invoke Policy to be able to use a JWT to call target systems, allowing for the request from APIC to backend target systems to authenticate a JWT rather than Basic Authentication.
  
![image](https://user-images.githubusercontent.com/66093865/162878276-40077b41-de22-4948-9d64-1d9de2c25c57.png)  
    
## Designing the Policy Palette (look, feel, and input of the policy)  
    
![image](https://user-images.githubusercontent.com/66093865/162878580-0b92f9de-8955-4745-b5c0-5e0fe399276f.png)
  

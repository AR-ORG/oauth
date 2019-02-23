# oauth

##  Pre-requisite 
    
    1.  KOPS cluster up and running 
    
    2.  Account present on auth0.com (free tier should be fine) 
    
    3.  Existing domain name - we will use kubernetesfederatedcluster.com for our demo
    
# Demo on securing Kubernetes Dashboard with Auth0 and RBAC using name context 

##  Setup Application, User and Database on Auth0

    1.  Login to auth0.com with your credentials 
    
    2.  Click on New application -> Select "Regular Web App"
    
    3.  Give the application a name - ex : kubernetes 
    
    4.  Click Create 
    
    5.  On the Application Dashboard -> Click Settings 
    
    7.  Add the below settings - 
        
          a.  Allowed Callback URLs: http://authserver.kubernetesfederatedcluster.com/callback
          
          b.  Allowed Logout URLs: http://authserver.kubernetesfederatedcluster.com/
          
    8.  Save Changes 
    
    9.  Click on Connections -> Databases -> Create DB COnnections 
    
    10. Provide a name to the DB- we will use the default name : Username-Password-Authentication
    
    11. Click on Application -> Select our application (Kubernetes) 
    
    12. Click on Connections and enable the database Username-Password-Authentication
    
    13. Click on User -> Create User and fill in details for the user you want to create. Make sure to enable user from the link sent on email 
    
    14. Go to application -> Click Settings 
    
    15. Note the details -
    
          a.  Domain
          
          b.  Client ID
          
          c.  Client Secret
          

##  Enable Oauth on Kube-apiserver

    1.  kops edit cluster CLUSTER_NAME 
    
    2.  Under spec: add the below 
    
      kubeAPIServer:
      oidcClientID: MVr58j9piGPSFLhl0KN0DexcpgYeMnd4
      oidcIssuerURL: https://dev-em3g7oer.auth0.com/
      oidcUsernameClaim: name
    
    3.  kops update cluster {CLUSTER_NAME} --yes
    
    4.  Perform rolling update of the cluster 
    
    5.  In case you are not using kops, then edit either the kube-apiserver manifest or kube-apiserver systemd unit file to add the 
        below flags 
        
        --oidc-client-id
        
        --oidc-issuer-url
        
        --oidc-required-claim
        
##  Deploy Kubernetes Dashboard 

    a.  clone this repo
    
    b.  kubectl apply -f v1.8.3.yaml     --- This deploys dashboard version v1.8.3.
    
    c.  This file has been slightly provide clusterrole to the service account of dashboard and not just to kube-system.
    
    d.  This RBAC helps in getting information about the entire cluster and not just kube-system 
    
##  Deploy OAUTH Proxy server 

    a.  There are many proxy servers available that enables proxying to your application using the OpenID string (name in our case) 
    
    b.  On  Auth0 -> Click Application -> Select our Application (kubernetes) -> Click QUickStart 
    
    c.  This allows you to download the template of proxy server of your choice of programming language. 
    
    d.  You can also use external proxy server like Apache with mod_auth plugin
    
    e.  From our repo edit the below files - 
    
        a.  auth0-secrets.yml
        
            Copy your secret key from AUTH0 application
            
            echo -n YOUR_SECRET_KEY | base64 
            
            Add this base64 encoded key as a value to  AUTH0_CLIENT_SECRET: inside the file 
            
        b.  auth0-deployment.yml
        
            Add Client ID to value: AUTH0_CLIENT_ID
            
            Add Auth0 Domain to value: AUTH0_DOMAIN_NAME
            
            Add Callback URL to AUTH0_CALLBACK_URL
            
            Add Auth0 Domain to value: https://AUTH0_DOMAIN_NAME/userinfo
            
            Add Database name to AUTH0_CONNECTION
            
            Add KUBERNETES_UI_HOST as the DNS for kube-apiserver. you can get it by executing - grep -i server ~/.kube/config 
            
            Add APP_HOST as LOGOUT URL 
            
    f.  kubectl create -f auth0-secrets.yml -f auth0-service.yml -f auth0-deployment.yml
    
    g.  Verify pods and services are running 
    
##  Add records to route53

    a.  kubectl get svc kubernetes-auth-server
    
    b.  AWS will spin up a Loadbalancer automatically if you are using KOPS. Get the LB name
    
    c.  we are using kubernetesfederatedcluster.com as our Hosted Domain on Route53
    
    d.  Our callback URL is set as authserver.kubernetesfederatedcluster.com. 
    
    e.  On Route 53 - create a new A record as authserver 
    
    f.  Type will be A-IPv4 address
    
    g.  Alias as yes 
    
    h.  Set Alias target as the Loadbalancer name that is registered against authserver proxy service
    
    i. The A record propogation sometimes takes around 20 mins 
    
##  Create RBAC for users created as a part of AUTH0

    a.  To access dashboard Kubernetes must provide cluster-admin access to each user
    
    b.  You can restrict access using RBAC by defining appropriate privileges to different apigroups 
    
    c.  In our case - the users are defined at AUTH0 and only the contexts provided by AUTH0 is provided to apiserver 
    
    d.  We have kept the Claim Name as "name". The name is set as email address in AUth0 
    
    e.  Kubernetes will, by default, add a prefix to the name that it receives from OIDC 
    
    h.  The prefix is the DOMAIN_NAME 
    
    i. So the User eventually have the name as - "https://AUTH0_DOMAIN_NAME/#USER_EMAIL"
    
    j. When you create RBAC clusterrolebinding - add the Subject as User and Name as "https://AUTH0_DOMAIN_NAME/#USER_EMAIL"
    
    i. From our repo - edit rbac.yaml file with correct username 
    
    j.  kubectl create -f rbac.yaml
    
##  Launch Dashboard 

    Once A-Record is populated - you can visit - http://authserver.kubernetesfederation.com (Your A-RECORD.DOMAIN) and Login 
            
            


  
        
        
    

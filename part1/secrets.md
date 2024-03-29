**note to grader, please see "Assignment 3 - full write up" file in home directory with screenshots. This file will have same info, just w/o screenshots**

I started Part 1 by doing reconnaissance for Kubernetes YAML files which have the word ‘SECRET’ in them. Accordingly, I ran  `grep -r 'secret .` in the main directory to recursively find all files with the keyword ‘secret’ in them. 

After closer inspection of the results, I found that the following files contained secrets that were hard-coded or otherwise exposed. 

./GiftcardSite/k8/django-deploy.yaml
./GiftcardSite/LegacySite/views.py
./GiftcardSite/GiftcardSite/settings.py
./db/k8/db-deployment.yaml
/db/k8s/db-deployment.yaml


In the views.py file, I commented out a function that would expose the password in the debug logs, as suggested by Kevin's earlier comment. 

In the settings.py file, I commented out the SECRET_KEY and used an environmental variable instead to store the secret locally. 

The updated code for this is as follows: 

`SECRET_KEY = os.environ.get('SECRET_KEY') #storing secret as local var in secrets.yaml file`

This brought me to django-deploy.yaml and db-deployment.yaml files which both had hardcoded secrets. In order to resolve this, I followed the documentation from this article (https://betterprogramming.pub/how-to-use-kubernetes-secrets-for-storing-sensitive-config-data-f3c5e7d11c15?gi=e1fcd44dcef3) which suggested that I create a separate yaml file to store these secrets and then simply link them back via Reference in each of the yaml files above. Insofar as template for the new yaml file is concerned, I followed the existing ‘django-admin-pass-secret.yaml’ format and simply renamed it to ‘securedsecrets.yam;’ and added the missing SECRET_KEY which was commented out from the settings.py file.
For each of the secrets, I ensure they were base64 encoded by running 
`echo <key> | base64`

Next, I applied the updated secured secrets file by running `kubectl apply -f securedsecrets.yaml`

The updated YAML file looks as follows:

```
apiVersion: v1
kind: Secret
metadata:
    name: securedsecrets
type: Opaque
data:
    username: YWRtaW4=
    password: dGhpc2lzYXRlc3R0aGluZy4=
    secret_key: a21neXNhI2Z6KzkoejEqPWMweWRyaml6ayo3c3RobTJnYTF6ND1eNjEkY3hjcThiJGw=
```


Additionally, I ran`kubectl get secrets` which confirmed that securedsecrets were indeed applied.

Finally, I updated each of the above yaml files to link back to the secrets file, thus removing the hard-coded secret exposure 
/db/k8s/db-deployment.yaml, ./db/k8/db-deployment.yaml, and ./GiftcardSite/k8/django-deploy.yaml files. 

Finally,  in order to make sure the update propagated accordingly as per documentation from kubernetes (https://www.containiq.com/post/using-kubectl-to-restart-a-kubernetes-pod), I deleted each of the pods so the settings can refresh for each of the containers running
I did this by running `kubectl delete pod <insert pod name>`

Lastly, I checked the container's env file to ensure the keys were propagated accordingly.
I did this by running `kubectl exec -it <container name> /bin/sh`

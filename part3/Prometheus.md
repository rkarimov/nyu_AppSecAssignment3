**note to grader, please see "Assignment 3 - full write up" file in home directory with screenshots. This file will have same info, just w/o screenshots**
For **Part 3.1**, I started by analyzing with exploring prometheus security best practices with reading articles (https://jfrog.com/dont-let-prometheus-steal-your-fire/ and https://danuka-praneeth.medium.com/setting-up-a-comprehensive-monitoring-system-with-prometheus-6b2b73cf54d2). Following that, I carefully analyzed the views.py file for exposure of any sensitive metrics. The most obvious exposure was related `graphs[pword].inc()` which logged secret use during a POST request. I initially noticed this in part 1 as well and commented a portion of it out, but full remediated it now by commenting out everything from `if pword.. ` to `graphs[pword].inc()".`

Additionally, I commented out the tracker for the number of login requests as this may reveal sensitive behavior from customers

```
graphs = {}
graphs['r_counter'] = Counter('python_request_r_posts', 'The total number'\
  + ' of register posts.')
#graphs['l_counter'] = Counter('python_request_l_posts', 'The total number'\
#  + ' of login posts.')         #PART 3 FIX: graphs['l_counter'].inc() we don't want to expose number of logins 
graphs['b_counter'] = Counter('python_request_b_posts', 'The total number'\
  + ' of card buy posts.')
```
Additionally, the following lines were also commented out from views.py file 

```
#if pword not in graphs.keys():
#    graphs[pword] = Counter(f'counter_{pword}', 'The total number of '\
#      + f'times {pword} was used')
#graphs[pword].inc()
```

**Part 3.2**
For Part 3.2, I decided to expand monitoring by also logging the 404 errors returned. Accordingly, I added the `‘error_return_counter’` graph which would track the 404 errors on return statements. 
An except from such change looks as follows:
```
graphs['error_return_counter'].inc() ##Part 3 - track error returns
return HttpResponse("ERROR: 404 Not Found.")
```

Next, I searched for ‘404 Not found’ across the views.py file to find instances where an error is returned and found 4 of such instances. Accordingly, I added the `graphs['error_return_counter'].inc()` statement to each instance of error return, which would respectively increment the counter with each error in db.

**Part 3.3**
I started off by downloading helm using the instructions on its respective website (https://helm.sh/docs/intro/install/) 
Next, using the recently installed helm repo, I installed prometheus from its respective repo referenced here (https://github.com/prometheus-community/helm-charts) 

In accordance with instructions in tutorial (https://artifacthub.io/packages/helm/prometheus-community/prometheus), I ran `kubectl get services` and `kubectl get pods` to ensure prometheus was running accordingly 
Next, I ran `​minikube service --all` to locate all services running and isolate the prometheus server 

From there, I ran `kubectl expose service prometheus-1669583381-server --type=NodePort --target-port=9090 --name=prometheus-server-np` to expose the appropriate prometheus server to its respective port 

Additionally, I was able to confirm that it was running by visiting its respective webpage for prometheus server which is running (http://192.168.49.2:32460/graph?g0.expr=&g0.tab=1&g0.stacked=0&g0.show_exemplars=0&g0.range_input=1h)

Next, I had to adjust the configmap so the prometheus server would be to pull metrics from our website. I did this by first running `kubectl get configmap`to get the list of configmaps avaliable. 
Next, I edited the configmap by running `kubectl edit configmap prometheus-1669583381-server` for prometheus server to point to another job for `GiftCardSite_Monitoring` as follows:

```
## Part 3.3 adding monitoring for giftcardsite
- job_name: GiftCardSite_monitoring
static_configs:
- targets:
- proxy-service:8080
```
Once edited, it was confirmed that the config was saved with the folling response: 
`configmap/prometheus-1669583381-server edited`

Lastly, I piped a copy of the updated YAML file by running 
`kubectl get configmap prometheus-1669583381-server -o yaml > promethus-server_updated_copy.yaml`

Following this change, I simply restarted the PODS (by deleting each one via `kubectl delete pod <pod_name>`) and confirmed they were running accordingly after (`kubectl get pods`)
Additionally, I went into the `/etc` directory in server separately by running `kubectl exec --it prometheus-1669583381-server-88d7b9746-5znw8 /bin/sh` to ensure the YAML file updated accordingly 

Finally, I ran`“minkube service list prometheus` to confirm that the server was pointing to a PORT.
As a confirmation check for myself,  I visited the website and confirmed that monitoring for giftcard site was present 
Additionally, I checked to make sure that the counter `pword` filter didn’t exist and confirmed it was not present which means views.py is reflecting everything accordingly!
---
layout: post
title:  "Deploying from Travis-CI to Google Container Engine"
description: "A guide through survival CI scripting"
date:   2016-10-12 09:00:00
keywords: "gce, google, kubectl, kubernetes, docker, container, travis, ci"
category: ci
comments: true
---

<h2>Deploying from Travis-CI to Google Container Engine</h2>

[Travis-ci][travis] is a great [CI][ci] tool.
In comparison to other solutions, it has the advantage of being free for Open Source projects and is well documented.

This is why I've been using it for a while and more recently with [regexrace][regexrace], a project hosted in [Google Container Engine cluster][gce].
Since [Kubernetes][kubernetes] abstracts much of the deployment and management processes, the only thing we have to handle is how to trigger the deployment.

Which is incredibly simple once you've setup the gcloud CLI :

{% highlight bash %}
kubectl apply -f my_replica_controller_config.yml
kubectl rolling-update my_replica_controller --image=gcr.io/my_project/my_project:the_project_version --image-pull-policy Always
{% endhighlight %}

But if you try to create a bash script and trigger it from Travis you may encounter these errors :

{% highlight bash %}
Unable to connect to the server: dial tcp <your_cluster_ip>: i/o timeout
# OR
error: You must be logged in to the server (the server has asked for the client to provide credentials)
{% endhighlight %}

To get rid of these errors, we'll have to ensure kubectl is using the right credentials every time we use it.

<h3>If you didn't set up travis yet</h3>
To set up gcloud in Travis and push the Docker build to [Google Container Registry][gcr], you should either read [the good article][article_scott] of Scott Smerchek or (the very-fast way),
to copy/paste the following lines and set up the related environment variables in your Travis project settings :

{% highlight yaml %}
cache:
  directories:
    - "$HOME/google-cloud-sdk/"
after_success:
  - if [ ! -d "$HOME/google-cloud-sdk/bin" ]; then rm -rf $HOME/google-cloud-sdk; curl https://sdk.cloud.google.com | bash; fi
  # Add gcloud to $PATH
  - source /home/travis/google-cloud-sdk/path.bash.inc
  - gcloud version
  - gcloud --quiet components update kubectl
  # Auth flow
  - echo $GCLOUD_KEY | base64 --decode > gcloud.p12
  - gcloud auth activate-service-account $GCLOUD_EMAIL --key-file gcloud.p12
  - ssh-keygen -f ~/.ssh/google_compute_engine -N ""
  # Push to Google container registry
  - docker build -t gcr.io/$CLOUDSDK_CORE_PROJECT/$CLOUDSDK_CORE_PROJECT:v1 .
  - gcloud docker push gcr.io/$CLOUDSDK_CORE_PROJECT/$CLOUDSDK_CORE_PROJECT:v1 > /dev/null
  - gcloud container clusters get-credentials $CLOUDSDK_CORE_PROJECT
{% endhighlight %}

Related environment variables :

| Env variables                 | Content                                                |
| ----------------------------- |:------------------------------------------------------ |
| GCLOUD_KEY                    | `base64-encoded version of project_name-xxxxx.json`    |
| GCLOUD_EMAIL                  | `{string_of_characters}@developer.gserviceaccount.com` |
| CLOUDSDK_CORE_DISABLE_PROMPTS | `1`                                                    |
| CLOUDSDK_CORE_PROJECT         | `your_project_name_on_gcloud`                          |
| CLOUDSDK_COMPUTE_ZONE         | `us-east1-b` *(for example)*                           |
| GKE_USERNAME                  | `the_cluster_username`                                 |
| GKE_PASSWORD                  | `the_cluster_password`                                 |
| GKE_SERVER                    | `the_cluster_ip`                                       |


<h3>Let's fix the kubectl command</h3>

After getting all of these errors, I tried to get some debug logs and looked into the gcloud manual pages and Github issues. Finally, I found an explanation on [a Github issue][reason_why] and one of the last comment of zeg-io.

> Your ~/.kube/config is missing some value for authentication. To resolve this, run the command below on your gcloud shell (Google Console):
> $ export GOOGLE_APPLICATION_CREDENTIALS="/path/to/keyfile.json"
>
> Next, you'll need to run the get-credentials command below again to fetch credentials for a running cluster.
> $ gcloud container clusters get-credentials project-cluster-1 --zone us-central1-c
>
> Then, you may now run 'kubectl proxy' command again and let me know if you're still having trouble.

So,
{% highlight bash %}
gcloud container clusters get-credentials $CLOUDSDK_CORE_PROJECT
{% endhighlight %}
requires you to store a JSON file containing the credentials.

As we have it stored in the `$GCLOUD_KEY` environment variable and we decoded it into a `gcloud.p12` file, we can set the content of `$GOOGLE_APPLICATION_CREDENTIALS` to the path of that file.

For example :
{% highlight bash %}
$GOOGLE_APPLICATION_CREDENTIALS="/home/travis/gopath/src/github.com/thylong/regexrace/gcloud.p12"
{% endhighlight %}

The final result can be seen [here][regexrace_repo].

Enjoy !
Don't hesitate to comment / open pull requests or issues on the regexrace project.


[article_scott]:  http://scottsmerchek.com/2015/07/24/pushing-to-google-container-registry-from-circleci/
[ci]:             https://en.wikipedia.org/wiki/Continuous_integration
[travis]:         https://travis-ci.com
[gce]:            https://cloud.google.com/container-engine/
[gcr]:            https://cloud.google.com/container-registry/
[kubernetes]:     http://kubernetes.io/
[regexrace]:      http://regexrace.org
[regexrace_repo]: http://github.com/thylong/regexrace
[reason_why]:     https://github.com/kubernetes/kubernetes/issues/28612

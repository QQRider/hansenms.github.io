---
layout: post
title:  "Tweet from Docker Hub"
date:   2016-03-12 15:00:00 -0500
categories: DevOps Tips
---

I have written a few notes about [Docker](https://www.docker.com/) related stuff. I like it a lot. For some of our projects we use it as an integral part of our continuous integration testing and deployment strategy. When the CI system has built and tested a Docker image, it can be pushed to [Docker Hub](https://hub.docker.com) and be immediately available for deployment. That is great, but what if you would like to let your users know that there is a new release but you are too lazy to actually go to [Twitter](http://twitter.com) and post it? No need to despair. You can make Docker Hub tweet it for you. 

Docker Hub offers a [webhook](https://docs.docker.com/docker-hub/webhooks/) feature that will allow you to notify an web server that a repo has been updated. The webhook settings are available for all repositories that you have permissions to modify. To get started, go to a repository that you would like to send updates from and enter a URL for a server that you have access to. For instance, something like:

       http://myweb.myserver.com:9999/dockerhub?dockerkey=WhX59Cm3UA6qaxcacOyk4DS1Ly3hQbE7cDuWbavCpl180pICLJOvTHoZR2ak4go

The `dockerkey` is a long key of random characters that you can use in your web application to make sure you trust the client that is connecting. You can make it up or get it generated at [https://www.grc.com/passwords.htm](https://www.grc.com/passwords.htm). We will get back to this key.

Next thing you need to do is to write a web application that can receive the webhook from Docker Hub and turn it into a tweet. There are of course many ways to do that. Here is an example of a simple Python application written using [Flask](http://flask.pocoo.org/):

{% highlight python %}
from flask import Flask
from flask import request
from twitter import *
import json
import sys
import requests

app = Flask(__name__)
config = dict()

@app.route('/dockerhub', methods=['POST'])
def dockerhub():
    global dockerkey

    parsed_json = json.loads(request.data)
    tag = parsed_json[u'push_data'][u'tag'] 
    repo_name = parsed_json[u'repository'][u'repo_name']
    url = parsed_json[u'repository']['repo_url']

    msg = "New #MyFancyApplication release " + tag + ", #Docker image `docker pull " + repo_name + ":" + tag + "', " + url 
    if tag != 'latest' and request.args.get('dockerkey') == dockerkey:
        t = Twitter(auth=OAuth(config['twitter']['token'], config['twitter']['token_key'], config['twitter']['con_secret'], config['twitter']['con_secret_key']))
        t.statuses.update(status=msg)

    return "dockerhub"

if __name__ == "__main__":

    with open(sys.argv[1]) as config_file:    
        config = json.load(config_file)

    dockerkey = config['dockerkey']
        
    app.run(host='0.0.0.0', port=config['port'])
{% endhighlight %}

The application uses a json configuration file with the Twitter login information and the `dockerkey` that we used in the URL. As you can we check that we are receiving the correct key from Docker Hub before sending a tweet. The configuration file would look something like this:

{% highlight json %}
{
"port": 9999,
"twitter": {
           "token": "<TWITTER TOKEN>",
	   "token_key": "<TWITTER_TOKEN_KEY>",
	   "con_secret": "<TWITTER_CON_SECRET>",
	   "con_secret_key": "<TWITTER CON SECRET>"
},
           
"dockerkey": "WhX59Cm3UA6qaxcacOyk4DS1Ly3hQbE7cDuWbavCpl180pICLJOvTHoZR2ak4go"
}
{% endhighlight %}

In the URL above we are using HTTP instead of HTTPS. This is clearly problematic in terms of security. They key could easily be sniffed and the Twitter account could be taken over by somebody who knew the URL and the key. It is unlikely that somebody would stumble into it by accident, but if somebody really wanted make the Twitter account tweet, they could. In a real production scenario you would use a proper web server and HTTPS. Setting that up with certificates and everything is beyond the scope of this little tutorial. You can look at this [blog post](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uwsgi-and-nginx-on-ubuntu-14-04) for some inspiration. 

The last thing left is how you get the necessary information for the Twitter account. Fortunately that is easy too. Just go to [https://apps.twitter.com/](https://apps.twitter.com/) and create a new application and keys.

Now all you have to do is to start the Python application, e.g.:

    python docker_tweet.py config.json

And try to push to the repository that you have hooked up. Your Docker Hub repository should now be tweeting. 

Good luck. Let me know if you have comments/suggestions. 


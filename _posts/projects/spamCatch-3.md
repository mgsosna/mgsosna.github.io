---
layout: post
title: Building a full-stack spam catching app - 3. Frontend & Deployment
title-clean: Building a full-stack spam catching app <div class="a2">3. Frontend & Deployment</div>
author: matt_sosna
image: "images/projects/spamcatch-demo.png"
---

![]({{  site.baseurl  }}/images/projects/spamcatch-demo.png)

In short, here are the steps:
1. Train a [TF-IDF](https://monkeylearn.com/blog/what-is-tf-idf/) vectorizer on a corpus of [ham and spam text messages](https://www.kaggle.com/uciml/sms-spam-collection-dataset), removing stop words and performing [lemmatization](https://nlp.stanford.edu/IR-book/html/htmledition/stemming-and-lemmatization-1.html) to make it easier for a model to understand the content of each document
2. Train a [random forest](https://stackabuse.com/random-forest-algorithm-with-python-and-scikit-learn/) classifier to distinguish between "spam" and "ham" TF-IDF vectors
3. Build a simple [Flask](https://flask.palletsprojects.com/en/1.1.x/) app with endpoints for webpages and the random forest classifier
4. Write the HTML and CSS for the user-facing pages
5. Write the JavaScript to communicate between the user-facing pages and the spam classifier
6. Deploy to [Heroku](https://www.heroku.com/about) so others can use your app


---
<span style="font-size:18px">**1. Background** ([first post]({{  site.baseurl  }}/spamCatch-1))</span>
  - [Spam]({{  site.baseurl  }}/spamCatch-1/#spam)
  - [Strings to vectors]({{  site.baseurl  }}/spamCatch-1/#strings-to-vectors)
  - [Why random forest?]({{  site.baseurl  }}/spamCatch-1/#why-random-forest)
  - [What is Flask?]({{  site.baseurl  }}/spamCatch-1/#what-is-flask)<br><br>

<span style="font-size:18px">**2. The Backend** ([last post]({{  site.baseurl  }}/spamCatch-2))</span>
  - [**Python**]({{  site.baseurl  }}/spamCatch-2/#python)
    - [The TF-IDF vectorizer]({{  site.baseurl  }}/spamCatch-2/#the-tf-idf-vectorizer)
    - [The classifier]({{  site.baseurl  }}/spamCatch-2/#the-spam-classifier)
    - [Flask]({{  site.baseurl  }}/spamCatch-2/#flask)

<span style="font-size:18px">**3. The Frontend and Deployment** (this post)</span>
  - [**The front-end**](#the-front-end)
    - [HTML](#html)
    - [JavaScript](#javascript)
  - [**Deployment**](#deployment)

---

## The front-end
### HTML
Our HTML is stored in the `templates` directory, as Flask expects. We start with a header to load in D3.js, Bootstrap CSS, and our custom CSS files. We also include a `<style>` tag to specify a footer class.

{% include header-html.html %}
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>SpamCatch: Let's catch some spam</title>
    <script src="https://d3js.org/d3.v5.min.js"></script>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css">
    <link rel="stylesheet" href="../static/css/reset.css">
    <link rel="stylesheet" href="../static/css/styles.css">
</head>
```

This simple styling gives us the black background with white text.

{% include header-html.html %}
```html
<body style="background-color: black; color: white">
```

This `<div>` is our background image. I found it easier to have a CSS class with the image as the background rather than inserting the image with `<img>` tags.

{% include header-html.html %}
```html
<div class="hero text-center"></div>
```

The CSS for this class, in [styles.css](static/css/styles.css):

{% include header-css.html %}
```css
.hero {
    position: relative;
    height: 240px;
    padding: 1px;
    margin-top: -2px;
    margin-left: -10px;
    background: black;
        background-attachment: scroll;
        background-image: url("../images/eggs.png");
        background-size: auto;
        background-size: cover;
}
```

After a header to get us pumped up, we have an input field to type in the spam message, with a placeholder to guide the user. Hitting enter (or even clicking outside the box) will trigger the JavaScript event handler (which we'll cover in a moment), but we also add a button for clarity.

{% include header-html.html %}
```html
<div class="text-center">
    <h1 style="color:white">Let's catch some spam.</h1>
<input id="text" size="100" placeholder="Type a text message here"
 style="background-color: black; color: white; border-color: black; font-size:16px">
 <button id="button" type="button" class="btn-danger" style="font-size:17px">Submit</button>
</div>
```

We then have two elements that are empty at loading but will be updated with the model output once a user submits a message.

{% include header-html.html %}
```html
<h1 id="spamProb" class="text-center spamProb"></h1>
<div id="decision" class="text-center" style="font-size: 20px"></div>
```

Finally, we have our footer with a link to the `/about` endpoint, as well as our JavaScript script that will make our page dynamic.

{% include header-html.html %}
```html
<div id="footer">
<a href="/about" style="color:orange">More info</a>
</div>

</body>

<script src="../static/js/script.js"></script>
</html>
```

### JavaScript
We start by defining strings that we'll reference or modify later. `SPAM_API` is our classifier endpoint, while the following templates are the HTML we'll use for any probability returned by the endpoint. We use our custom `String.format` function to fill the `{0}` and `{1}` placeholders with inputted arguments; it's apparently taboo to modify built-in classes in JavaScript, but I was craving something analogous to Python's convenient string templating.

{% include header-javascript.html %}
```javascript
const SPAM_API = "/classify"
var spamProbTemplate = `Spam probability: <span style="color: {0}">{1}</span><span style="color:white">%</span>`;
var decisionTemplate = `<span style="color: {0}">{1}</span>`;
```

We then have our event handlers. The first function, `onSubmit`, gets the value of the input text field, logs it to the console, and passes it to `updateSpamProb`.

{% include header-javascript.html %}
```javascript
function onSubmit() {
    var text = d3.select("#text").property("value");
    console.log(text);
    updateSpamProb(text);
}
```

`updateSpamProb` then selects the (formerly empty) div whose ID is `spamProb`. It fills the URL with the text from the input field, then uses D3 to perform a `GET` request to our `classify` Flask endpoint. Our endpoint returns a probability of spam, which we convert to a HEX color with `prob2color`. We then format our `spamProbTemplate` with the color and spam probability (converted to a rounded percentage), and assign this to the div's HTML. We then use `updateDecision` to update the `decisionTemplate` with the corresponding message and color.

{% include header-javascript.html %}
```javascript
function updateSpamProb(value) {
    var div = d3.select("#spamProb");
    var url = `${SPAM_API}/${value}`;

    d3.json(url).then(prob => {
         var color = prob2color((1-prob));
         div.html(String.format(spamProbTemplate, color, Math.round(100*prob)));
         updateDecision(prob, color);
     });
}
```

Finally, we have the event listeners, which are what actually connect these JavaScript functions to our HTML page. When the user changes the text input field or clicks the button, we'll trigger `onSubmit`, which kicks off the whole process.

{% include header-javascript.html %}
```javascript
d3.select("#text").on("change", onSubmit);
d3.select("#button").on("click", onSubmit);
```

## Deployment
We can serve our app locally with `python app.py` and then navigating to `localhost:5000` (or whichever port your app ends up using), but the next level is being able to let anyone interact with the app. For this, we turn to Heroku. I found [this StackAbuse article](https://stackabuse.com/deploying-a-flask-application-to-heroku/) incredibly helpful. To summarize briefly:

1. Create a `Procfile` that's just the line `web: gunicorn app:app --preload --workers 1`.
  - This file tells Heroku to use `gunicorn` to serve our app, which is called `app` inside `app.py`, and to preload a worker before serving the app. Preloading causes Heroku's error logs to be much more informative.
2. Create a `requirements.txt` file with the necessary packages for your app to run. Make sure `Flask` and `gunicorn` are included.
3. Create a Heroku account and start a new application.
4. Link the GitHub repo for your app to your Heroku account, then manually deploy your app.
5. Repeat steps 1-4 a dozen times, puzzling over the error logs and making small changes. The final bug for me was needing to have `scikit-learn==0.21.3` in my requirements file, not `sklearn`.
6. Once you make it through, celebrate! You've deployed a web app!

## Conclusions
Ways to make it better:
* Looking for patterns in the URLs themselves. `www.google.com` is fine, but `u7x0apmw.com` isn't.
* Add features like number of letters that are capitalized, number of exclamation marks

## Footnotes
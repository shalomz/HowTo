# Build a Sentiment Analysis Tool for Twitter with this Simple Python Script
### PS: This is copied from [ Aylien Blog ](http://blog.aylien.com/2017/10/18/)

Twitter users around the world post around 350,000 new Tweets every minute,
creating 6,000 140-character long pieces of information every second. Twitter
is now a hugely valuable resource from which you can extract insights by using
text mining tools like sentiment analysis.

Within the social chatter being generated every second, there are vast amounts
of hugely valuable insights waiting to be extracted. With sentiment analysis,
we can generate insights about consumers' [reactions to
announcements](http://blog.aylien.com/ryanair-handles-pr-crisis-text-analysis-
case-study/), opinions on products or brands, and even track opinion about
[events as they unfold](http://blog.aylien.com/using-nlp-to-understand-how-
twitter-and-the-media-reacted-to-the-super-bowl-51-ads-battle/). For this
reason, you'll often hear sentiment analysis referred to as "opinion mining".

With this in mind, we decided to put together a useful tool built on a single
Python script to help you get started mining public opinion on Twitter.

## **What the script does**

Using this one script you can gather Tweets with the [Twitter
API](https://developer.twitter.com/), analyze their sentiment with the [AYLIEN
Text Analysis API](https://aylien.com/text-api/), and visualize the results
with matplotlib - **all for free**. The script also provides a visualization
and saves the results for you neatly in a CSV file to make the reporting and
analysis a little bit smoother.

Here are some of the cool things you do with this script:

  * Understand the public's reaction to news or events on Twitter
  * Measure the voice of your customers and their opinions on you or your competitors
  * Generate sales leads by identifying negative mentions of your competitors

You can see the script running a sample analysis of 50 Tweets mentioning Tesla
in our example GIF below - storing the results in a CSV file and showing a
visualization. The beauty of the script is you can search for whatever you
like and it will run your tweets through the same analysis pipeline.

![Tesla Sentiment](http://blog.aylien.com/wp-content/uploads/2017/10/Tesla-
Sentiment.gif)

## **Installing the dependencies &amp; getting API keys**

Since doing a sentiment analysis of Tweets with our API is so easy, installing
the libraries and getting your API keys is by far the most time-consuming part
of this blog.

We've collected them here as a four-step to do list:

  1. Make sure you have the following libraries installed (which you can do with pip):
  1. Get API keys for Twitter:
  * Getting the API keys from Twitter Developer (which you can do [here](https://developer.twitter.com/en/community)) is the most time consuming part of this process, but [this video](https://www.youtube.com/watch?v=5PUC9yGS4RI) can help you if you get lost.
  * **What it costs &amp; what you get**: the free Twitter plan lets you download 100 Tweets per search, and you can search Tweets from the previous seven days. If you want to upgrade from either of these limits, you'll need to pay for the Enterprise plan ($$)
  * To do the sentiment analysis, you'll need to sign up for our Text API's free plan and grab your API keys, which you can do [here](https://developer.aylien.com/signup).
  * **What it costs &amp; what you get**: the free Text API plan lets you analyze 30,000 pieces of text per month (1,000 per day). If you want to make more than 1,000 calls per day, our Micro plan lets you analyze 80,000 pieces of text for ($49/month)
  1. Copy, paste, and run the script below!

## **The Python script**

When you run this script it will ask you to specify what term you want to
search Tweets for, and then to specify how many Tweets you want to gather and
analyze.

    
    
     import sys import csv import tweepy import matplotlib.pyplot as plt from collections import Counter from aylienapiclient import textapi if sys.version_info[0] < 3: input = raw_input ## Twitter credentials consumer_key = "Your consumer key here" consumer_secret = "your secret consumer key here" access_token = "your access token here" access_token_secret = "your secret access token here" ## AYLIEN credentials application_id = "Your app ID here" application_key = "Your app key here" ## set up an instance of Tweepy auth = tweepy.OAuthHandler(consumer_key, consumer_secret) auth.set_access_token(access_token, access_token_secret) api = tweepy.API(auth) ## set up an instance of the AYLIEN Text API client = textapi.Client(application_id, application_key) ## search Twitter for something that interests you query = input("What subject do you want to analyze for this example? \n") number = input("How many Tweets do you want to analyze? \n") results = api.search( lang="en", q=query + " -rt", count=number, result_type="recent" ) print("--- Gathered Tweets \n") ## open a csv file to store the Tweets and their sentiment file_name = 'Sentiment_Analysis_of_{}_Tweets_About_{}.csv'.format(number, query) with open(file_name, 'w', newline='') as csvfile: csv_writer = csv.DictWriter( f=csvfile, fieldnames=["Tweet", "Sentiment"] ) csv_writer.writeheader() print("--- Opened a CSV file to store the results of your sentiment analysis... \n") ## tidy up the Tweets and send each to the AYLIEN Text API for c, result in enumerate(results, start=1): tweet = result.text tidy_tweet = tweet.strip().encode('ascii', 'ignore') if len(tweet) == 0: print('Empty Tweet') continue response = client.Sentiment({'text': tidy_tweet}) csv_writer.writerow({ 'Tweet': response['text'], 'Sentiment': response['polarity'] }) print("Analyzed Tweet {}".format(c)) ## count the data in the Sentiment column of the CSV file with open(file_name, 'r') as data: counter = Counter() for row in csv.DictReader(data): counter[row['Sentiment']] += 1 positive = counter['positive'] negative = counter['negative'] neutral = counter['neutral'] ## declare the variables for the pie chart, using the Counter variables for "sizes" colors = ['green', 'red', 'grey'] sizes = [positive, negative, neutral] labels = 'Positive', 'Negative', 'Neutral' ## use matplotlib to plot the chart plt.pie( x=sizes, shadow=True, colors=colors, labels=labels, startangle=90 ) plt.title("Sentiment of {} Tweets about {}".format(number, query)) plt.show() 

If you're new to Python, text mining, or sentiment analysis, the next sections
will walk through the main sections of the script.

## **The script in detail**

### **Python 2 &amp; 3**

With the migration from Python 2 to Python 3, you can run into a ton of
problems working with text data (if you're interested, check out [a great
summary of why](http://python-
notes.curiousefficiency.org/en/latest/python3/text_file_processing.html) by
Nick Coghlan). One of the changes is that Python 3 runs input() as a string,
whereas Python 2 runs input() as a Python expression, so these lines change
this to raw_input() if you're running Python 2.

    
    
     if sys.version_info[0] < 3: input = raw_input 

### **Input your search **

The goal of this post is to make it as quick and easy as possible to analyze
the sentiment of Tweets that interest you. This script does that by letting
you easily change the search term and sample size every time you run the
script from the shell using the input() method.

    
    
     query = input("What subject do you want to analyze for this example? \n") number = input("How many Tweets do you want to analyze? \n") 

### **Run your Twitter query**

We're grabbing the most recent Tweets relevant to your query, but you can
change this to 'popular' if you want to mine only the most popular Tweets
published, or 'mixed' for a bit of both. You can see we've also decided to
exclude retweets, but you might decide that you want to include them. You can
check the full list of parameters
[here](https://developer.twitter.com/en/docs/tweets/search/api-reference/get-
search-tweets.html). (From our experience there can be a lot of noise in
retaining Tweets that have been Retweeted.)

An important point to note here is that the Twitter API limits your results to
100 Tweets, and it doesn't return an error message if you try to search for
more than 100 Tweets. So if you input 500 Tweets, you'll only have 100 Tweets
to analyze, and title of your visualization will still read '500 Tweets.'

    
    
     results = api.search( lang="en", q=query + " -rt", count=number, result_type="recent" ) 

### **Open a CSV file for the Tweets &amp; Sentiment Analysis**

Writing the Tweets and their sentiment to a CSV file allows you to review the
API's analysis of each Tweet. First, we open a new CSV file and write the
headers.

    
    
     with open(file_name, 'w', newline='') as csvfile: csv_writer = csv.DictWriter( f=csvfile, fieldnames=["Tweet", "Sentiment"] ) csv_writer.writeheader() 

### **Tidy the Tweets**

Dealing with text on Twitter can be messy, so we've included this snippet to
tidy up the Tweets before you do the sentiment analysis. This means that your
results are more accurate, and you also don't waste your free AYLIEN credits
on empty Tweets.

    
    
     for c, result in enumerate(results, start=1): tweet = result.text tidy_tweet = tweet.strip().encode('ascii', 'ignore') if len(tweet) == 0: print('Empty Tweet') continue 

### **Write the Tweets &amp; their Sentiment to the CSV File**

You can see that actually getting the sentiment of a piece of text only takes
a couple of lines of code, and here we're writing the Tweet itself and the
result of the sentiment (positive, negative, or neutral) to the CSV file under
the headers we already wrote. You'll notice that we're actually writing the
Tweet as returned by the AYLIEN Text API instead of the Tweet we got from the
Twitter API. Even though they're both the same, writing the Tweet that the
AYLIEN API returns just reduces the potential for errors and mistakes.

We're also going to print something every time the script analyzes a Tweet.

    
    
     response = client.Sentiment({'text': tidy_tweet}) csv_writer.writerow({ 'Tweet': response['text'], 'Sentiment': response['polarity'] }) print("Analyzed Tweet {}".format(c)) 

![Screenshot \(546\)](http://blog.aylien.com/wp-
content/uploads/2017/10/Screenshot-546.png%20921w,%20http://blog.aylien.com
/wp-content/uploads/2017/10/Screenshot-546-300x107.png%20300w,%20http://blog.a
ylien.com/wp-content/uploads/2017/10/Screenshot-546-768x275.png%20768w)

If you want to include results on how confident the API is in the sentiment it
detects in each Tweet: just add  "response['polarity_confidence']" above and
add a corresponding header when you're opening your CSV file.

### **Count the results of the Sentiment Analysis**

Now that we've got a CSV file with the Tweets we've gathered and their
predicted sentiment, it's time to visualize these results so we can get an
idea of the sentiment immediately. To do this, we're just going to use
Python's standard counter library to count the number of times each sentiment
polarity appears in the 'Sentiment' column.

    
    
     with open(file_name, 'r') as data: counter = Counter() for row in csv.DictReader(data): counter[row['Sentiment']] += 1 positive = counter['positive'] negative = counter['negative'] neutral = counter['neutral'] 

### **Visualize the Sentiment of the Tweets**

Finally, we're going to plot the results of the count above on a simple pie
chart with matplotlib. This is just a case of declaring the variables and then
using matplotlib to base the sizes, labels, and colors of the chart on these
variables.

    
    
     colors = ['green', 'red', 'grey'] sizes = [positive, negative, neutral] labels = 'Positive', 'Negative', 'Neutral' ## use matplotlib to plot the chart plt.pie( x=sizes, shadow=True, colors=colors, labels=labels, startangle=90 ) plt.title("Sentiment of {} Tweets about {}".format(number, query)) plt.show() 

![Screenshot \(542\)](http://blog.aylien.com/wp-
content/uploads/2017/10/Screenshot-542.png%20923w,%20http://blog.aylien.com
/wp-content/uploads/2017/10/Screenshot-542-300x254.png%20300w,%20http://blog.a
ylien.com/wp-content/uploads/2017/10/Screenshot-542-768x649.png%20768w,%20http
://blog.aylien.com/wp-content/uploads/2017/10/Screenshot-542-651x550.png%20651
w,%20http://blog.aylien.com/wp-content/uploads/2017/10/Screenshot-542-60x50.pn
g%2060w,%20http://blog.aylien.com/wp-
content/uploads/2017/10/Screenshot-542-60x50@2x.png%20120w)

## **Go further with Sentiment Analysis**

If you want to go further with sentiment analysis you can try two things with
your AYLIEN API keys:

  * If you're looking into reviews of restaurants, hotels, cars, or airlines, you can try our [aspect-based sentiment analysis](https://aylien.com/text-api/aspect-based-sentiment-analysis/) feature. This will tell you what sentiment is attached to each aspect of a Tweet - for example positive sentiment shown towards food but negative sentiment shown towards staff.
  * If you want sentiment analysis customized for the problem you're trying to solve, take a look at [TAP](https://aylien.com/text-analysis-platform/), which lets you train your own language model from your browser.

## **Building a Sentiment Analysis Workflow for your Organization**

This script is built to give you a snapshot of sentiment at the time you run
it, so to keep abreast of any change in sentiment towards an organization
you're interested in, you should try running this script every day.

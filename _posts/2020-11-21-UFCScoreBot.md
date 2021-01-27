---
title: "UFC Score Bot: Web Crawler"
date: 2020-11-21T15:34:30-04:00
categories:
  - blog
tags:
  - Jekyll
  - update
---

UFCScoreBot is an automated fighter record retriever built to work with [Reddit](https://www.reddit.com/). It was created for the purpose of getting UFC fighter records without having to leave the site and is tied to a Reddit account with the name `/u/UFCScoreBot`. It can be activated by mentioning its username in a comment along with the name of a UFC fighter. It will return the fighter's record in the form of a reply to the invoking user.
<div style= "text-align: center"><img src="/assets/images/reddit_logo.png" width = "200" height = "200"/></div>

# In-Depth

## The Bot
The Bot is a constantly running service written in Python and connected to Reddit using PRAW which stands for Public Reddit API Wrapper. It uses this API to listen for "mentions" i.e comments that contain its username `/u/UFCScoreBot`. After it receives a mention notification it then parses the comment where it was mentioned and retrieves the UFC fighter's name in a first and last form. The code for the Bot part is fairly short and I have included it below.
```python
import praw
import sys
import logging
import os
import json

from time import sleep

TOP_DIR = os.path.dirname(os.path.dirname(os.path.realpath(__file__)))
SCORES_DIR = os.path.join(TOP_DIR, "scores")
SPIDER_FILE = os.path.join(TOP_DIR, "scores", "scores", "spiders", "charlotte.py")
OUTPUT_DIR = os.path.join(SCORES_DIR, "scrapy_output")

class UFCBot:
    def __init__(self):
      logging.info("Creating reddit instance")
      self.reddit = praw.Reddit(user_agent='Bobby')
    
    def start(self):
        logging.info("Starting bot")
        logging.info("Press \'Ctrl + C\' to exit")
        while True:
          try:
            for mention in self.reddit.inbox.mentions():
              args = self.parse(mention.body)
              if args != None:
                logging.info("Running scraper for: {}".format(mention.body))
                self.runScraper(args)
                logging.info("Preparing for reply")
                reply_body = self.getStats(args)
                logging.info("Replying to commment...")
                mention.reply(reply_body)
            logging.info("Checking for mentions again in 10 seconds...")
            sleep(10)
          except KeyboardInterrupt:
            logging.info("Exiting Bot")
            exit()

    def parse(self, comment):
        args_list = comment.split(" ")
        if len(args_list) != 3:
          logging.warning("Haven't implemented parsing for more/less than 3 words -- bot will not run spider")
          return None
        args_dict = {"first": args_list[1].lower(), "last": args_list[2].lower()}
        return args_dict
    
    def runScraper(self, args):
        runCommand = "scrapy crawl -a last={last_name} -a first={first_name} charlotte"
        if not os.getcwd() == SCORES_DIR:
          os.chdir("scores")
        os.system(runCommand.format(last_name=args["last"], first_name=args["first"]))

    def getStats(self, args):
        logging.info("Retreiving scraped data")
        target_file = os.path.join(OUTPUT_DIR, "{last}_{first}.json".format(last=args['last'], first=args['first']))
        with open(target_file, 'r+') as target:
          data = json.load(target)
        return data[0]['record']

if __name__ == "__main__":
    logging.basicConfig(
      stream=sys.stdout,
      level=logging.DEBUG,
      format='%(asctime)s [%(name)s] %(levelname)s: %(message)s',
      datefmt='%Y-%m-%d %H:%M:%S'
      )
    mybot = UFCBot()
    mybot.start()
```

## The Crawler
After a name has been retrieved, the Bot makes a call to [Scrapy](https://scrapy.org/), a web scraping framework, and passes it the name as well as a spider. A spider is what Scrapy uses to allow for custom crawling and scraping. For this project I named my spider charlotte(after charlotte's web) and she can be invoked using the following line: 

* `scrapy crawl -a last={last_name} -a first={first_name} charlotte`. 

The spider then searches [www.ufcstats.com](http://ufcstats.com) for the fighters page and then once on the page it looks for that fighters record and saves it to a file in JSON format. The Bot then uses the contents of this file to form a reply to the invoking user. Since the code for the spider is fairly short as well I have included it below.

```python
import scrapy
import sys
import os

from scores.items import ScoresItem
from scrapy.spidermiddlewares.httperror import HttpError
from twisted.internet.error import DNSLookupError
from twisted.internet.error import TimeoutError, TCPTimedOutError

class ScoreSpider(scrapy.Spider):
    name = "charlotte"

    def start_requests(self):
        self.logger.debug("Initiating request")
        queryUrl = "http://www.ufcstats.com/statistics/fighters/search?query=" + self.last.strip().lower()
        urls = [queryUrl]
        for url in urls:
          yield scrapy.Request(url=url, callback=self.parse_fighters, errback=self.errback_general)


    def errback_general(self, failure):
        self.logger.error(repr(failure))
        if failure.check(HttpError):
          # these exceptions come from HttpError spider middleware
          # you can get the non-200 response
          response = failure.value.response
          self.logger.error('HttpError on {response_url}'.format(response_url=response.url))

        elif failure.check(DNSLookupError):
          # this is the original request
          request = failure.request
          self.logger.error('DNSLookupError on {request_url}'.format(request_url=request.url))

        elif failure.check(TimeoutError, TCPTimedOutError):
          request = failure.request
          self.logger.error('TimeoutError on {request_url}'.format(request_url=request.url))

    def parse_fighters(self, response):
        '''
            Find fighter
        '''
        self.logger.debug("Looking at fighter list")
        cleaned_first = self.first.strip()
        xpathStr = "//tbody/tr/td/a[contains(translate(text(), '{upper_case}', '{lower_case}'), '{first_name}')]/@href"
        href_list = response.xpath(xpathStr.format(
          upper_case=cleaned_first.upper(),
          lower_case=cleaned_first.lower(),
          first_name=cleaned_first.lower()
        )).extract()
        if len(href_list) == 0:
          self.logger.debug("Found no match")
          return
        elif len(href_list) == 1:
          self.logger.debug("Found 1 match")
          yield response.follow(href_list[0], callback=self.parse_stats, errback=self.errback_general)
        elif len(href_list) > 1:
          self.logger.debug("Found multiple matches")
          self.logger.warning("Haven't implemented multiple matches")
          return

    def parse_stats(self, response):
        '''
            Extract record
        '''
        self.logger.debug("Creating item")
        item = ScoresItem()
        xpathStr = '//*[@class="b-content__title-record"]/text()'
        record_list = response.xpath(xpathStr).extract()
        if(len(record_list) == 1):
          self.logger.debug("Storing record in item")
          item['record'] = record_list[0].strip()
        else:
          self.logger.warning("Haven't implemented multiple list record")
          return
        return item        

```

Here is a [link](https://github.com/kcharellano/UFCScoreBot) to the github repository. If you made it this far, thanks for reading!
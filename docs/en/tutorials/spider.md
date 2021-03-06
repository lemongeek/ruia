# Spider Control

`ruia.Spider` is used to control the whole spider,
it provides the following functions:

* Normalize your code
* Maintain a event loop
* Manage requests and responses
* Concurrency control
* Manage middlewares and plugins

Although it works well,
to use only `ruia.Item` to create a spider,
`ruia` recommend to use `ruia.Spider` to implement a stronger spider.

## Normalize your code

`ruia.Spider` requires a class property `start_urls` as the entry point of a spider.
Inner, `ruia` will iterate `start_urls`,
and send a request to server for each request.
After receiving server response,
`ruia` will call `spider.parse(response)`,
and this is the main part of your spider.

Here's a simple parse example, to simply save response fields to a text file.
We only have to define `start_urls`,
and implement a `parst` method.

```python
import aiofiles

from ruia import Spider, Item, TextField, AttrField


class HackerNewsItem(Item):
    target_item = TextField(css_select='tr.athing')
    title = TextField(css_select='a.storylink')
    url = AttrField(css_select='a.storylink', attr='href')

class HackerNewsSpider(Spider):
    start_urls = [f'https://news.ycombinator.com/news?p={index}' for index in range(3)]

    async def parse(self, response):
        async for item in HackerNewsItem.get_items(html=response.html):
            yield item

    async def process_item(self, item: HackerNewsItem):
        """Ruia build-in method"""
        async with aiofiles.open('./hacker_news.txt', 'a') as f:
            await f.write(str(item.title) + '\n')

```

`aiofiles` is a third-party library to operate files in asynchronous way.
It provides APIs the same as python standard `open` function.

Now, we have written a spider,
and time to start crawling.

```python
import aiofiles

from ruia import Spider, Item, TextField, AttrField


class HackerNewsItem(Item):
    target_item = TextField(css_select='tr.athing')
    title = TextField(css_select='a.storylink')
    url = AttrField(css_select='a.storylink', attr='href')

class HackerNewsSpider(Spider):
    start_urls = [f'https://news.ycombinator.com/news?p={index}' for index in range(3)]

    async def parse(self, response):
        async for item in HackerNewsItem.get_items(html=response.html):
            yield item

    async def process_item(self, item: HackerNewsItem):
        """Ruia build-in method"""
        async with aiofiles.open('./hacker_news.txt', 'a') as f:
            await f.write(str(item.title) + '\n')

if __name__ == '__main__':
    HackerNewsSpider.start()

``` 

Done.
Now your code is more readable and maintainable.

## Send Further Requests

I don't think that just crawling the catalogue of news satisfied you.
Next we will crawl news itself.
Hacker news gathers news from many websites,
it's not easy to parse each article of it.
For this example, we'll crawl [Github Developer Documentation](https://developer.github.com/v3/).

```python
# Target: https://developer.github.com/v3/
from ruia import *


class CatalogueItem(Item):
    target_item = TextField(css_select='.sidebar-menu a')
    title = TextField(css_select='a')
    link = AttrField(css_select='a', attr='href')

    def clean_link(self, value):
        return f'https://developer.github.com{value}'


class PageItem(Item):
    content = HtmlField(css_select='.content')


class GithubDeveloperSpider(Spider):
    start_urls = ['https://developer.github.com/v3/']

    async def parse(self, response: Response):
        catalogue = []
        async for cat in CatalogueItem.get_items(html=response.html):
            catalogue.append(cat)
        for page in catalogue[:6]:
            if '#' in page.link:
                continue
            yield Request(url=page.link, metadata={'title': page.title}, callback=self.parse_page)

    async def parse_page(self, response: Response):
        item = await PageItem.get_item(html=response.html)
        title = response.metadata['title']
        print(title, len(item.content))


if __name__ == '__main__':
    GithubDeveloperSpider.start()

```

See the `GithubDeveloperSpider.parse` method.
After extracting titles and urls,
it `yield` a request.

About `yield`,
you should learn from python documentation.
Here we can regard it as sending a task to background process.

Okay, now that we have already send the request to background process,
we have loss the control of the request.
Nothing serious,
after the request finished,
the response will send to its `callback` parameter.
`callback` parameter should be a function, or something callable.
In `parse_page` method,
we accept the response.
Then it comes with another problem:
we have already get the page title from catalogue in method `parse`,
but they are not in the context of `parse_page`.
That's why we need a `metadata` argument.
we put data into `metadata` in the previous method,
and get data from it in the following method.

Now, run the spider.

```text
output:
Media Types 8652
Overview 38490
OAuth Authorizations API 66565
Other Authentication Methods 6651
Troubleshooting 2551
```

## Concurrency Control

Let's repeat the Github Developer spider.

```python
# Target: https://developer.github.com/v3/
from ruia import *


class CatalogueItem(Item):
    target_item = TextField(css_select='.sidebar-menu a')
    title = TextField(css_select='a')
    link = AttrField(css_select='a', attr='href')

    def clean_link(self, value):
        return f'https://developer.github.com{value}'


class PageItem(Item):
    content = HtmlField(css_select='.content')


class GithubDeveloperSpider(Spider):
    start_urls = ['https://developer.github.com/v3/']
    concurrency = 5

    async def parse(self, response: Response):
        catalogue = []
        async for cat in CatalogueItem.get_items(html=response.html):
            catalogue.append(cat)
        for page in catalogue[:20]:
            if '#' in page.link:
                continue
            yield Request(url=page.link, metadata={'title': page.title}, callback=self.parse_page)

    async def parse_page(self, response: Response):
        item = await PageItem.get_item(html=response.html)
        title = response.metadata['title']
        print(title, len(item.content))


if __name__ == '__main__':
    GithubDeveloperSpider.start()

```

This time, there's a line added:

```python
    concurrency = 5
```

Here's a brief introduction about concurrency.
Some websites are friendly to crawlers,
while some are not.
If you visit a website too frequently,
you will be banned from the server.
Besides, to be a good crawler,
we should protect the server,
rather than making it crash.
Not every server can burden a huge spider.

To protect both, we have to control our concurrency.
Concurrency means the connection numbers in a time.
In this case, we set it to 5.

Let's have a short look on the log.

```text
Output:
[2019:01:23 00:01:59]-ruia-INFO  spider : Spider started!
[2019:01:23 00:01:59]-ruia-WARNINGspider : ruia tried to use loop.add_signal_handler but it is not implemented on this platform.
[2019:01:23 00:01:59]-ruia-WARNINGspider : ruia tried to use loop.add_signal_handler but it is not implemented on this platform.
[2019:01:23 00:01:59]-Request-INFO  request: <GET: https://developer.github.com/v3/>
[2019:01:23 00:02:00]-Request-INFO  request: <GET: https://developer.github.com/v3/>
[2019:01:23 00:02:00]-Request-INFO  request: <GET: https://developer.github.com/v3/media/>
[2019:01:23 00:02:00]-Request-INFO  request: <GET: https://developer.github.com/v3/oauth_authorizations/>
[2019:01:23 00:02:00]-Request-INFO  request: <GET: https://developer.github.com/v3/auth/>
[2019:01:23 00:02:00]-Request-INFO  request: <GET: https://developer.github.com/v3/troubleshooting/>
[2019:01:23 00:02:01]-Request-INFO  request: <GET: https://developer.github.com/v3/previews/>
Overview 38490
[2019:01:23 00:02:02]-Request-INFO  request: <GET: https://developer.github.com/v3/versions/>
OAuth Authorizations API 66565
[2019:01:23 00:02:02]-Request-INFO  request: <GET: https://developer.github.com/v3/activity/>
Media Types 8652
[2019:01:23 00:02:02]-Request-INFO  request: <GET: https://developer.github.com/v3/activity/events/>
Troubleshooting 2551
[2019:01:23 00:02:02]-Request-INFO  request: <GET: https://developer.github.com/v3/activity/events/types/>
API Previews 19537
[2019:01:23 00:02:02]-Request-INFO  request: <GET: https://developer.github.com/v3/activity/feeds/>
Other Authentication Methods 6651
[2019:01:23 00:02:03]-Request-INFO  request: <GET: https://developer.github.com/v3/activity/notifications/>
Versions 1344
Feeds 14090
[2019:01:23 00:02:03]-Request-INFO  request: <GET: https://developer.github.com/v3/activity/starring/>
Activity 2178
[2019:01:23 00:02:04]-Request-INFO  request: <GET: https://developer.github.com/v3/activity/watching/>
[2019:01:23 00:02:05]-Request-INFO  request: <GET: https://developer.github.com/v3/checks/>
Events 11844
Starring 55228
[2019:01:23 00:02:05]-Request-INFO  request: <GET: https://developer.github.com/v3/checks/runs/>
[2019:01:23 00:02:05]-Request-INFO  request: <GET: https://developer.github.com/v3/checks/suites/>
Event Types & Payloads 1225037
Notifications 65679
Watching 35775
Checks 7379
Check Runs 116607
[2019:01:23 00:02:06]-ruia-INFO  spider : Stopping spider: ruia
Check Suites 115330
[2019:01:23 00:02:06]-ruia-INFO  spider : Total requests: 18
[2019:01:23 00:02:06]-ruia-INFO  spider : Time usage: 0:00:07.342048
[2019:01:23 00:02:06]-ruia-INFO  spider : Spider finished!

```

Focus on the first several lines.

```text
[2019:01:23 00:01:54]-Request-INFO  request: <GET: https://developer.github.com/v3/>
[2019:01:23 00:02:00]-Request-INFO  request: <GET: https://developer.github.com/v3/>
[2019:01:23 00:02:00]-Request-INFO  request: <GET: https://developer.github.com/v3/media/>
[2019:01:23 00:02:00]-Request-INFO  request: <GET: https://developer.github.com/v3/oauth_authorizations/>
[2019:01:23 00:02:00]-Request-INFO  request: <GET: https://developer.github.com/v3/auth/>
[2019:01:23 00:02:00]-Request-INFO  request: <GET: https://developer.github.com/v3/troubleshooting/>
[2019:01:23 00:02:05]-Request-INFO  request: <GET: https://developer.github.com/v3/previews/>
Overview 38490
[2019:01:23 00:02:07]-Request-INFO  request: <GET: https://developer.github.com/v3/versions/>
OAuth Authorizations API 66565
```

The first request is at our requesting the catalogue page.
Then, our spider send 5 requests at almost same time, at `[00:02:00]`.
5 seconds later, at `[00:02:05]`, our spider receives a response, and then sent another request.
The response was parsed immediately.
2 seconds later, at `[00:02:07]`, our spider receives another response,
and sent another request.
Then, it parsed the response immediately.

That is to say,
at any time,
there are 5 connections between spider and server.
That is concurrency control.

Hey, notice that our spider sent 5 requests at same time!
Thanks to python's `asyncio` library,
we can write asynchronous crawler easier and faster.
Coroutines runs faster than multi-threadings.

## Use Middleware

`Ruia` provides mainly two ways to enhance itself.

Firstly let's talk about middlewares.
Middlewares are used to process a request before it's sending
and to process a response after it's receiving
In a word, it is something between your spider and server.

[Here](https://github.com/ruia-plugins/ruia-ua) is a simple middleware named `ruia-ua`,
it is used to automatically add random User-Agent to your requests.

Firstly, install `ruia-ua`.

```shell
pip install ruia-ua
```

Then, add it to your spider.

```python
from ruia import AttrField, TextField, Item, Spider
from ruia_ua import middleware


class HackerNewsItem(Item):
    target_item = TextField(css_select='tr.athing')
    title = TextField(css_select='a.storylink')
    url = AttrField(css_select='a.storylink', attr='href')

    async def clean_title(self, value):
        return value


class HackerNewsSpider(Spider):
    start_urls = ['https://news.ycombinator.com/news?p=1', 'https://news.ycombinator.com/news?p=2']
    concurrency = 10

    async def parse(self, res):
        async for item in HackerNewsItem.get_items(html=res.html):
            print(item.title)


if __name__ == '__main__':
    HackerNewsSpider.start(middleware=middleware)

```

`ruia.Spider` has an argument `middleware`.
It receives a list or a single middleware.

## Use Plugin

If you want better control of your spider,
try to use some plugins.

[ruia-pyppeteer](https://github.com/ruia-plugins/ruia-pyppeteer) is a `ruia` plugin used for loading JavaScript.

Firstly, install `ruia-pyppeteer`.

```shell
pip install ruia_pyppeteer
# New features
pip install git+https://github.com/ruia-plugins/ruia-pyppeteer
```

!!!Note
    When you use load_js, it will download a recent version of Chromium (~100MB). This only happens once.


Here is a simple example to show how to load JavaScript.

```python
import asyncio

from ruia_pyppeteer import PyppeteerRequest as Request

request = Request("https://www.jianshu.com/", load_js=True)
response = asyncio.get_event_loop().run_until_complete(request.fetch())
print(response.html)
```

Here is an example to use it in your spider:

```python
from ruia import AttrField, TextField, Item

from ruia_pyppeteer import PyppeteerSpider as Spider
from ruia_pyppeteer import PyppeteerRequest as Request


class JianshuItem(Item):
    target_item = TextField(css_select='ul.list>li')
    author_name = TextField(css_select='a.name')
    author_url = AttrField(attr='href', css_select='a.name')

    async def clean_author_url(self, author_url):
        return f"https://www.jianshu.com{author_url}"


class JianshuSpider(Spider):
    start_urls = ['https://www.jianshu.com/']
    concurrency = 10
    # Load js on the first request
    load_js = True

    async def parse(self, response):
        async for item in JianshuItem.get_items(html=response.html):
            # Loading js by using PyppeteerRequest
            yield Request(url=item.author_url, load_js=self.load_js, callback=self.parse_item)

    async def parse_item(self, response):
        print(response)

if __name__ == '__main__':
    JianshuSpider.start()
```


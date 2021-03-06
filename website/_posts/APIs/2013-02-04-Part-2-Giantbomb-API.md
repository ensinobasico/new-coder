---
layout: post.html
title: "Part 2: Giantbomb API"
tags: [api, generator]
url: "/api/part-2/"
---

Parsing of a publicly available API.

### Giantbomb API class

Similar to our `CPIData` class, we want to write a class for our Giantbomb API.

Before we continue, make sure you’ve signed up for a developer’s API key with [Giantbomb](http://www.giantbomb.com/api/). Do not share your key information with anyone (but if you do accidently, be sure to regenerate your key).  An API key (or token, etc) is used by folks that have public APIs (like [Twitter](http://dev.twitter.com), [Yelp](http://www.yelp.com/developers/documentation/v2/overview), etc) to identify your unique application (or user/developer).

So in our `GiantbombAPI` class, we’ll create a constructor to initialize our API key. Everytime we instantiate this class, we’ll need to pass in our API key. This makes our code portable, so I can pass this piece of code onto someone else where they can pass in their own API key while still using our code to grab information from the API.

**Note:** It’s essential to make our code portable, or to have the code abstracted away from specifics to us as a user, developer, our machine, etc. It would do no one (but ourselves) any good if we hard-coded in our API key within this class; it’s better to pass it in as a parameter when instantiating the class.

In our class, we’ll also define a base URL for which all calls to the API will use.  Since we’re making calls to a URL, we will use the `requests` library again later on in this class (no new import statement need since we’re in the same .py file).

```python
class GiantbombAPI(object):
    """
    Very simple implementation of the Giantbomb API that only offers the
    GET /platforms/ call as a generator.

    Note that this implementation only exposes of the API what we really need.
    """

    base_url = 'http://www.giantbomb.com/api'

    def __init__(self, api_key):
        self.api_key = api_key
```

Next, we’ll define one method (our class’s only method besides our constructor) that will make a request to the `base_url` with the parameters that we feed the method: `sort`, `filter`, `field_list`. 

This method is a generator function (a clue is the `yield` statement instead of a `return` statement).

#### For the curious
The `yield` keyword is similar to `return`. The `parse` function, specifically the `for deal in selector` bit, we’ve essentially built a generator (it will generate data on the fly). StackOverflow has a good [explanation](http://stackoverflow.com/questions/231767/the-python-yield-keyword-explained) of what’s happening in our function: The first time the function will run, it will run from the beginning until it hits yield, then it’ll return the first value of the loop. Then, each other call will run the loop you have written in the function one more time, and return the next value, until there is no value to return.  The generator is considered empty once the function runs but does not hit yield anymore. It can be because the loop had come to ends, or because you do not satisfy a “if/else” anymore. 

#### Back to the tutorial
Follow the inline comments to undestand the process flow of `get_platforms`:

```python
	def get_platforms(self, sort=None, filter=None, field_list=None):
	    """Generator yielding platforms matching the given criteria. If no 
	    limit is specified, this will return *all* platforms.

	    """

	    # The API itself allows us to filter the data returned either
	    # by requesting only a subset of data elements or a subset with each
	    # data element (like only the name, the price and the release date).
	    #
	    # The following lines also do value-format conversions from what's
	    # common in Python (lists, dictionaries) into what the API requires.
	    # This is especially apparent with the filter-parameter where we
	    # need to convert a dictionary of criteria into a comma-seperated
	    # list of key:value pairs.
	    params = {}
	    if sort is not None:
	        params['sort'] = sort
	    if field_list is not None:
	        params['field_list'] = ','.join(field_list)
	    if filter is not None:
	    	params['filter'] = filter
	        parsed_filters = []
	        for key, value in filter.iteritems():
	            parsed_filters.append('{0}:{1}'.format(key, value))
	        params['filter'] = ','.join(parsed_filters)

	    # Last but not least we append our API key to the list of parameters
	    # and tell the API that we would like to have our data being returned
	    # as JSON.
	    params['api_key'] = self.api_key
	    params['format'] = 'json'

	    incomplete_result = True
	    num_total_results = None
	    num_fetched_results = 0
	    counter = 0

	    while incomplete_result:
	        # Giantbomb's limit for items in a result set for this API is 100
	        # items. But given that there are more than 100 platforms in their
	        # database we will have to fetch them in more than one call.
	        #
	        # Most APIs that have such limits (and most do) offer a way to
	        # page through result sets using either a "page" or (as is here
	        # the case) an "offset" parameter which allows you to "skip" a
	        # certain number of items.
	        params['offset'] = num_fetched_results
	        result = requests.get(self.base_url + '/platforms/',
	                              params=params)
	        result = result.json()
	        if num_total_results is None:
	            num_total_results = int(result['number_of_total_results'])
	        num_fetched_results += int(result['number_of_page_results'])
	        if num_fetched_results >= num_total_results:
	            incomplete_result = False
	        for item in result['results']:
	            logging.debug("Yielding platform {0} of {1}".format(
	                counter + 1,
	                num_total_results))

	            # Since this is supposed to be an abstraction, we also convert
	            # values here into a more useful format where appropriate.
	            if 'original_price' in item and item['original_price']:
	                item['original_price'] = float(item['original_price'])

	            # The "yield" keyword is what makes this a generator.
	            # Implementing this method as generator has the advantage
	            # that we can stop fetching of further data from the server
	            # dynamically from the outside by simply stop iterating over
	            # the generator.
	            yield item
	            counter += 1
```

Did you notice the `logging.debug` item? We’re using Python’s `logging` module from its standard library, so we’ll need to add another import statement at the beginning:

```python
import logging
```

It’s important to log what we’re doing throughout our script writing.  In case we catch an error somewhere, our logs can give us more helpful clues to where the error took place, or why it happened. You can configure the `logging` module to either save to a specific file, or have it as output to the console as the script processes. We’ll configure our logs in the final part of the API tutorial.

Next, we should also define a helper function, **outside** of our `GiantbombAPI` class, so that we can make sure that each platform that is yielded from `get_platform` is valid (some data we get back may not have a date that we can grab for our CPI data, or a price).  We do this outside of the class so it’s independent of our API object. This is akin to our helper functions that we made for `map.py` in our DataViz tutorial.

```python
def is_valid_dataset(platform):
    """Filters out datasets that we can't use since they are either lacking
    a release date or an original price. For rendering the output we also
    require the name and abbreviation of the platform.

    """
    if 'release_date' not in platform or not platform['release_date']:
        logging.warn(u"{0} has no release date".format(platform['name']))
        return False
    if 'original_price' not in platform or not platform['original_price']:
        logging.warn(u"{0} has no original price".format(platform['name']))
        return False
    if 'name' not in platform or not platform['name']:
        logging.warn(u"No platform name found for given dataset")
        return False
    if 'abbreviation' not in platform or not platform['abbreviation']:
        logging.warn(u"{0} has no abbreviation".format(platform['name']))
        return False
    return True
```

You should not be scared by the format we requested: `params['format'] = 'json'` – we had some fun with JSON-like parsing in our DataViz tutorial. This is the format we will get back when we call this method.  

[The next part will take that JSON data to save as a CSV or generate a plot for us &rarr;]( {{ get_url("/api/part-3/")}})
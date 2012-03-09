# IMS LTI

This ruby library is to help create Tool Providers and Tool Consumers for the [IMS LTI standard](http://www.imsglobal.org/lti/index.html).

## Installation
This is packaged as the `ims-lti` rubygem, so you can just add the dependency to your Gemfile or install the gem on your system:

    gem install ims-lti

To require the library in your project:

    require 'ims/lti'

To validate the OAuth signatures you need to require the appropriate request proxy for your application. For example, in a Sinatra app you'd require:

    require 'oauth/request_proxy/rack_request'

For further information see the [oauth-ruby](https://github.com/oauth/oauth-ruby) project.

## Usage
This readme won't cover the LTI standard, just how to use the library. It will be very helpful to read the [LTI documentation](http://www.imsglobal.org/lti/index.html)

In LTI there are Tool Providers (TP) and Tool Consumers (TC), this library is useful for implementing both. Here is an overview of the communication process: [LTI 1.1 Introduction](http://www.imsglobal.org/lti/v1p1pd/ltiIMGv1p1pd.html#_Toc309649680)

### Tool Provider
As a TP your app will receive a POST request with a bunch of [LTI launch data](http://www.imsglobal.org/lti/v1p1pd/ltiIMGv1p1pd.html#_Toc309649684) and it will be signed with OAuth using a key/secret that both the TP and TC share. This is covered in the [LTI security model](http://www.imsglobal.org/lti/v1p1pd/ltiIMGv1p1pd.html#_Toc309649685)

Here is an example of a simple TP Sinatra app using this gem: [LTI Tool Provider](https://github.com/instructure/lti_tool_provider)

This library doesn't help the TP manage the consumer keys and secrets. The POST headers will contain the `oauth_consumer_key` and your app can use that to look up the appropriate `oauth_consumer_secret`. Once you have the necessary credentials you can initialize a `ToolProvider` object with them and the post parameters:

```ruby
# Initialize TP object with OAuth creds and post parameters
provider = IMS::LTI::ToolProvider.new(consumer_key, consumer_secret, params)

# Verify OAuth signature by passing the request object
if provider.valid_request?(request)
  # success
else
  # handle invalid OAuth
end
```

Once your TP object is initialized and verified you can load your tool. All of the [launch data](http://www.imsglobal.org/lti/v1p1pd/ltiIMGv1p1pd.html#_Toc309649684) is available in the TP object along with some convenience methods like `provider.username` which will try to find the name from the 3 potential name launch data attributes.

#### Returning Results of a Quiz/Assignment
If your TP provides some kind of assessment service you can write grades back to the TC. This is documented in the LTI docs [here](http://www.imsglobal.org/lti/v1p1pd/ltiIMGv1p1pd.html#_Toc309649690).

You can check whether your the TC is expecting a grade write-back:

```ruby
if provider.outcome_service?
  # ready for grade write-back
else
  # normal tool launch without grade write-back
end
```

To write the grade back to the TC your tool will do a POST directly back to the URL the TC passed in the launch data. You can use the TP object to do that for you:

```ruby
# post the score to the TC, score should be a float >= 0.0 and <= 1.0
# this returns an OutcomeResponse object
response = provider.post_replace_result!(score)
if response.success?
  # grade write worked
elsif response.processing?
elsif response.unsupported?
else
  # failed
end
```

You can see the error code documentation [here](http://www.imsglobal.org/gws/gwsv1p0/imsgws_baseProfv1p0.html#1639667).
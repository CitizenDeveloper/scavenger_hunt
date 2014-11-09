# Code

```rb
module Stripe
  class Charge < APIResource

    def refund(params={})
      response, api_key = Stripe.request(:post, refund_url, @api_key, params)
      refresh_from(response, api_key)
    end

    def capture(params={})
      response, api_key = Stripe.request(:post, capture_url, @api_key, params)
      refresh_from(response, api_key)
    end

    def update_dispute(params)
      response, api_key = Stripe.request(:post, dispute_url, @api_key, params)
      refresh_from({ :dispute => response }, api_key, true)
      dispute
    end

    def close_dispute
      response, api_key = Stripe.request(:post, close_dispute_url, @api_key)
      refresh_from(response, api_key)
    end

    private

    def refund_url
      url + '/refund'
    end

    def capture_url
      url + '/capture'
    end

    def dispute_url
      url + '/dispute'
    end

    def close_dispute_url
      url + '/dispute/close'
    end
  end
end
```

# Questions

The following code interacts with:
  - a database
  - an API
  - neither

The code above relates to the following:
  - credit card transactions
  - couples therapy
  - restaurant point of sale

Does the above code support creating a subscription?
  - Yes
  - No

What lines of code deal with refunds?

Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data

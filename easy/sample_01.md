# Code

```rb
desc "Updates a user's mailchimp list"

task assign_mailchimp_bucket: :environment do
  bucketable_users = User.where.not(
    mailchimp_list: [:unsubscribed, :diy_for, :designer, :darby_girl, :prior_user, :vips]
  ).where(created_at: Date.parse('2014-03-06')..Time.now)

  bucketable_users.each do |user|s
    if user.spree_orders.complete.any? || user.subscribers.active.any?
      user.made_a_purchase
    elsif user.t1? && user.has_incomplete_diy_for?
      user.interested_in_diy_for
    elsif user.t1? && user.spree_orders.any?{ |order| order.line_items.present? }
      user.added_item_to_cart
    else
      user.took_no_action
    end
  end
end
```

# Questions

1. What does the above code do?
  - Updates a users mailchimp list
  - Assigns the user to a subscription service
  - Refunds a credit card transaction

2. If a user has a complete order, what event is triggered?
  - `user.made_a_purchase`
  - `user.interested_in_diy_for`
  - `user.added_item_to_cart`
  - `user.took_no_action`

3. Bucketable users are users in what time frame?
  - Today and yesterday
  - March 6th, 2014 to today
  - last week

4. Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data

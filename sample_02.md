# Context

The code here moves your users through an email marketing pipeline.  It runs nightly,
and it checks to see what actions a user has performed before reassigning them
to a new email campaign list provided by the popular email marketing site Mailchimp.

A user can be in one 'state' at a time, i.e. one email marketing list at a time, and
certain 'events' change their state.

Recently, a developer on your team fixed a few bugs that had crept into this system.
You'd like to double check the logic before pushing the new changes because
this system emails millions of your customers.

# Code

```rb
module Mailchimpable

  #                      ### Marketing Pipeline ###
  #                      last updated: Mar. 04 2014

  # LEGEND
  #   *email lists* are prefixed with a colon (i.e. :unbucketed)
  #   *user actions* are not prefeixed with a colon (i.e. purchase)

  #        :unbucketed                                                 Day 0
  #         /         \
  #    purchase      no-action                                         Day 1
  #       /                  \
  # :welcome_to_ds           :your_surprise
  #    /      \             /   |    |     \
  # :subscription :vips  *purchase cart diyemail no-action             Day 3
  #                           /      |              \
  #           :lets_get_crafty     :reveal       :pinwin
  #           /    \              /      \         /    \
  #  *purchase     no-action no-action    *purchase   no-action        Day 5
  #                 |            |                        |
  #               :goody      :grab_it                :diy_that
  #               /     \     /       \                 /   \
  #        no-action   *purchase    no-action   *purchase   no-action  Day 7
  #            |                        |                    |
  #        :exclusive               :exclusive           :exclusive

  #        *purchase
  #          /      \
  #  :subscription   :vips    <-- depends on purchase of subscription box

  included do
    state_machine :mailchimp_list, initial: :unbucketed do
      state :unbucketed,
            :unsubscribed,
            :designer,
            :teen_subscription,
            :welcome_to_ds,
            :your_surprise,
            :subscription,
            :vips,
            :lets_get_crafty,
            :reveal,
            :pinwin,
            :goody,
            :grab_it,
            :diy_that,
            :exclusive,
            :welcome,
            :prior_user,
            :incomplete_signup

      before_transition any => any, do: :mailchimp_list_unsubscribe
      after_transition  any => any, do: :mailchimp_list_subscribe

      event :unbucket do
        transition incomplete_signup: :unbucketed
      end

      event :took_no_action do
        transition [:diy_that, :grab_it, :goody] => :exclusive
        transition reveal:          :grab_it
        transition lets_get_crafty: :goody
        transition pinwin:          :diy_that
        transition your_surprise:   :pinwin
        transition unbucketed:      :your_surprise
      end

      event :made_a_purchase do
        transition all - [:unbucketed] => :subscription,     if: :purchased_subscription?
        transition all - [:unbucketed] => :vips,         unless: :purchased_subscription?
        transition unbucketed: :welcome_to_ds,               if: :t0?
      end

      event :added_item_to_cart do
        transition your_surprise: :lets_get_crafty
      end

      event :interested_in_subscription do
        transition your_surprise: :reveal
      end
    end

    after_create :subscribe_to_teen_subscription_list, if: :teen_subscription_role?
    after_create :subscribe_to_designer_list,          if: :designer_applicant?
    after_create :mailchimp_list_subscribe
  end

  def teen_subscription_role?
    roles.include?(:teen_subscription)
  end

  def mailchimp_list_subscribe
    Resque.enqueue(SubscribeUserToMailchimpList, id, mailchimp_list)
  end

  def mailchimp_list_unsubscribe
    Resque.enqueue(UnsubscribeUserFromMailchimpList, id, mailchimp_list)
  end

  def purchased_subscription?
    subscribers.active.any?{|subscriber| subscriber.offer.name == 'Subscription'}
  end

  def has_incomplete_subscription?
    subscribers.any?{|subscriber| subscriber.offer.name == 'Subscription'}
  end

  def subscribe_to_teen_subscription_list
    update(mailchimp_list: :teen_subscription)
  end

  def subscribe_to_designer_list
    update(mailchimp_list: :designer)
  end

  def day_old?
    created_at < 1.days.ago # user is older than 1 day
  end

  def t0?
    created_at > 2.days.ago # user is younger than 2 days old
  end

  def t1?
    created_at > 4.days.ago # user is younger than 4 days old
  end
end
```

# Questions

How many email marketing campaign lists is the code managing?

___number___

If a user adds an item to a cart and they are currently subscribed to the `your_surprise`
email list, what list are they added to?

__text_field___

Before the new changes, customers weren't being unsubscribed from their previous
email list before being subscribed to a new list.  Identify the line of code that
will unsubscribe the customer after a change to their email marketing list occurs.

__line_number___











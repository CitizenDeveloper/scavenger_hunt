# Context

This code represents a credit card in a billing system, its validation, storage
and retrieval.  Think of the online shopping that you've done in the past and
what its like to enter a credit card in Amazon.com or similar shopping site.

Your goal here is to learn more about this code so that you can confidently discuss
the features of the system while speaking with potential customers. In your last
meeting with them they were wondering the following:

- how many different types of credit cards do you support?
- how are you ensuring valid credit card information enters the system?
- what data are you collecting from our customers when they register a credit card?
- what rules do you support around refunds and credits to customers?

# Code

```rb
module Spree
  class CreditCard < ActiveRecord::Base
    has_many :payments, as: :source

    before_save :set_last_digits

    attr_accessor :number, :verification_value

    validates :month, :year, numericality: { only_integer: true }
    validates :number, presence: true, unless: :has_payment_profile?, on: :create
    validates :verification_value, presence: true, unless: :has_payment_profile?, on: :create
    validate :expiry_not_in_the_past

    scope :with_payment_profile, -> { where('gateway_customer_profile_id IS NOT NULL') }

    alias_attribute :brand, :cc_type

    CARD_TYPES = {
      visa: /^4[0-9]{12}(?:[0-9]{3})?$/,
      master: /(^5[1-5][0-9]{14}$)|(^6759[0-9]{2}([0-9]{10})$)|(^6759[0-9]{2}([0-9]{12})$)|(^6759[0-9]{2}([0-9]{13})$)/,
      diners_club: /^3(?:0[0-5]|[68][0-9])[0-9]{11}$/,
      american_express: /^3[47][0-9]{13}$/,
      discover: /^6(?:011|5[0-9]{2})[0-9]{12}$/,
      jcb: /^(?:2131|1800|35\d{3})\d{11}$/
    }

    def expiry=(expiry)
      if expiry.present?
        self[:month], self[:year] = expiry.delete(' ').split('/')
        self[:year] = "20" + self[:year] if self[:year].length == 2
      end
    end

    def number=(num)
      @number = num.gsub(/[^0-9]/, '') rescue nil
    end

    # cc_type is set by jquery.payment, which helpfully provides different
    # types from Active Merchant. Converting them is necessary.
    def cc_type=(type)
      self[:cc_type] = case type
      when 'mastercard', 'maestro' then 'master'
      when 'amex' then 'american_express'
      when 'dinersclub' then 'diners_club'
      when '' then try_type_from_number
      else type
      end
    end

    def set_last_digits
      number.to_s.gsub!(/\s/,'')
      verification_value.to_s.gsub!(/\s/,'')
      self.last_digits ||= number.to_s.length <= 4 ? number : number.to_s.slice(-4..-1)
    end

    def try_type_from_number
      numbers = number.delete(' ') if number
      CARD_TYPES.find{|type, pattern| return type.to_s if numbers =~ pattern}.to_s
    end

    def name?
      first_name? && last_name?
    end

    def name
      "#{first_name} #{last_name}"
    end

    def verification_value?
      verification_value.present?
    end

    # Show the card number, with all but last 4 numbers replace with "X". (XXXX-XXXX-XXXX-4338)
    def display_number
      "XXXX-XXXX-XXXX-#{last_digits}"
    end

    def actions
      %w{capture void credit}
    end

    # Indicates whether its possible to capture the payment
    def can_capture?(payment)
      payment.pending? || payment.checkout?
    end

    # Indicates whether its possible to void the payment.
    def can_void?(payment)
      !payment.void?
    end

    # Indicates whether its possible to credit the payment.  Note that most gateways require that the
    # payment be settled first which generally happens within 12-24 hours of the transaction.
    def can_credit?(payment)
      return false unless payment.completed?
      return false unless payment.order.payment_state == 'credit_owed'
      payment.credit_allowed > 0
    end

    def has_payment_profile?
      gateway_customer_profile_id.present?
    end

    def to_active_merchant
      ActiveMerchant::Billing::CreditCard.new(
        :number => number,
        :month => month,
        :year => year,
        :verification_value => verification_value,
        :first_name => first_name,
        :last_name => last_name
      )
    end

    private

    def expiry_not_in_the_past
      if year.present? && month.present?
        time = "#{year}-#{month}-1".to_time
        if time < Time.zone.now.to_time.beginning_of_month
          errors.add(:base, :card_expired)
        end
      end
    end
  end
end
```

# Questions

What data can we infer is NOT collected by the system in order to register a new credit card?

```
the answer is among the following options

  - name
  - billing zipcode
  - card number
  - expiration month and year
```

Hints:
- this code includes lines that explicitly create a new credit card


---

How many credit card types are supported by this system?

`the answer is a number`

Hints:
- this code explicitly defines the types of cards supported

---

Which line is responsible for ensuring that the credit card expiration month
and year are both numbers (as opposed to some arbitrary text, for example)?

`the answer is a line number`

Hints:
- credit card month and year must both be numbers
- programmers are specific about numbers and distinguish between integers and floating point numbers
- credit cards with non integer months and years would be invalidated by this code


---

When can a payment not be credited?

```
the answer is among the following options

  - when the payment is incomplete or credit is owed
  - when the payment is complete and credit is not owed
  - when the payment is incomplete or credit is owed or zero credit is allowed
  - no payments can be credited
```

Hints:
- the ability to credit a payment is explicitly defined in this code
- the rules for being able to credit a customer are all bundled close together
- comments are used by developers to help clarify complex decisions made by the system

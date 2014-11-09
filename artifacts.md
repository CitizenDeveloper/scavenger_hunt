
-----------

```rb
module Spree
  class Variant < Spree::Base
    belongs_to :product
    belongs_to :tax_category

    has_many :inventory_units
    has_many :line_items
    has_many :orders
    has_many :prices
    has_many :stock_items
    has_many :stock_locations, through: :stock_items
    has_many :stock_movements, through: :stock_items

    has_and_belongs_to_many :option_values
    has_one :default_price

    validate :check_price

    validates :cost_price, numericality: { greater_than_or_equal_to: 0 }
    validates :price,      numericality: { greater_than_or_equal_to: 0 }
    validates_uniqueness_of :sku

    after_create :create_stock_items

    def tax_category
      if self[:tax_category_id].nil?
        product.tax_category
      else
        TaxCategory.find(self[:tax_category_id])
      end
    end

    def cost_price=(price)
      self[:cost_price] = parse_price(price) if price.present?
    end

    # returns number of units currently on backorder for this variant.
    def on_backorder
      inventory_units.with_state('backordered').size
    end

    # use deleted? rather than checking the attribute directly. this
    # allows extensions to override deleted? if they want to provide
    # their own definition.
    def deleted?
      !!deleted_at
    end

    def default_price
      Spree::Price.unscoped { super }
    end

    def has_default_price?
      !self.default_price.nil?
    end

    def price_in(currency)
      prices.select{ |price| price.currency == currency }.first || Spree::Price.new(variant_id: self.id, currency: currency)
    end

    def amount_in(currency)
      price_in(currency).try(:amount)
    end

    def name_and_sku
      "#{name} - #{sku}"
    end

    def sku_and_options_text
      "#{sku} #{options_text}".strip
    end

    def in_stock?
      Rails.cache.fetch(in_stock_cache_key) do
        total_on_hand > 0
      end
    end

    def can_supply?(quantity=1)
      Spree::Stock::Quantifier.new(self).can_supply?(quantity)
    end

    def total_on_hand
      Spree::Stock::Quantifier.new(self).total_on_hand
    end

    private

    def set_master_out_of_stock
      if product.master && product.master.in_stock?
        product.master.stock_items.update_all(:backorderable => false)
        product.master.stock_items.each { |item| item.reduce_count_on_hand_to_zero }
      end
    end

    # Ensures a new variant takes the product master price when price is not supplied
    def check_price
      if price.nil? && Spree::Config[:require_master_price]
        raise 'No master variant found to infer price' unless (product && product.master)
        raise 'Must supply price for variant or master.price for product.' if self == product.master
        self.price = product.master.price
      end
      if currency.nil?
        self.currency = Spree::Config[:currency]
      end
    end

    def default_price_changed?
      default_price && (default_price.changed? || default_price.new_record?)
    end

    def save_default_price
      default_price.save if default_price_changed?
    end

    def set_cost_currency
      self.cost_currency = Spree::Config[:currency] if cost_currency.nil? || cost_currency.empty?
    end

    def create_stock_items
      StockLocation.where(propagate_all_variants: true).each do |stock_location|
        stock_location.propagate_variant(self)
      end
    end

    def set_position
      self.update_column(:position, product.variants.maximum(:position).to_i + 1)
    end

    def in_stock_cache_key
      "variant-#{id}-in_stock"
    end
  end
end
```

What does the following code refer to?
  - Credit cards
  - Point of sale
  - Store products

What line of code returns the number of stock on backorder?

What line of code returns the number of stock in the store?

What lines of code convert prices into new currencies?

When setting the master to 'out of stock', are all stock items backorderable?

Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data







---------

```rb
module Spree
  class Address < Spree::Base
    belongs_to :country, class_name: "Spree::Country"
    belongs_to :state, class_name: "Spree::State"

    has_many :shipments, inverse_of: :address

    validates :firstname, :lastname, :address1, :city, :country, presence: true
    validates :zipcode, presence: true, if: :require_zipcode?
    validates :phone, presence: true, if: :require_phone?

    validate :state_validate, :postal_code_validate

    alias_attribute :first_name, :firstname
    alias_attribute :last_name, :lastname

    def full_name
      "#{firstname} #{lastname}".strip
    end

    def state_text
      state.try(:abbr) || state.try(:name) || state_name
    end

    def same_as?(other)
      return false if other.nil?
      attributes.except('id', 'updated_at', 'created_at') == other.attributes.except('id', 'updated_at', 'created_at')
    end

    alias same_as same_as?

    def to_s
      "#{full_name}: #{address1}"
    end

    def ==(other_address)
      self_attrs = self.attributes
      other_attrs = other_address.respond_to?(:attributes) ? other_address.attributes : {}

      [self_attrs, other_attrs].each { |attrs| attrs.except!('id', 'created_at', 'updated_at', 'order_id') }

      self_attrs == other_attrs
    end

    # Generates an ActiveMerchant compatible address hash
    def active_merchant_hash
      {
        name: full_name,
        address1: address1,
        address2: address2,
        city: city,
        state: state_text,
        zip: zipcode,
        country: country.try(:iso),
        phone: phone
      }
    end

    def require_phone?
      true
    end

    def require_zipcode?
      true
    end

    private

    def postal_code_validate
      return if country.blank? || country.iso.blank? || !require_zipcode?

      postal_code = TwitterCldr::Shared::PostalCodes.for_territory(country.iso)
      errors.add(:zipcode, :invalid) if !postal_code.valid?(zipcode.to_s)
    end
  end
end
```


Does an address have a phone number?

Does this address implementation support multiple countries?

What line of code checks for zipcodes being provided?

If the state doesn't have an abbreviation, what shows up when we ask for the
name of the state?

Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data














----------------


```rb
require 'thread_safe'
require 'active_support/core_ext/array/prepend_and_append'
require 'active_support/i18n'

module ActiveSupport
  module Inflector
    extend self

    # A singleton instance of this class is yielded by Inflector.inflections,
    # which can then be used to specify additional inflection rules. If passed
    # an optional locale, rules for other languages can be specified. The
    # default locale is <tt>:en</tt>. Only rules for English are provided.
    #
    #   ActiveSupport::Inflector.inflections(:en) do |inflect|
    #     inflect.plural /^(ox)$/i, '\1\2en'
    #     inflect.singular /^(ox)en/i, '\1'
    #
    #     inflect.irregular 'octopus', 'octopi'
    #
    #     inflect.uncountable 'equipment'
    #   end
    #
    # New rules are added at the top. So in the example above, the irregular
    # rule for octopus will now be the first of the pluralization and
    # singularization rules that is runs. This guarantees that your rules run
    # before any of the rules that may already have been loaded.

    class Inflections
      attr_reader :plurals, :singulars, :uncountables, :humans, :acronyms, :acronym_regex

      def initialize
        @plurals, @singulars, @uncountables, @humans, @acronyms, @acronym_regex = [], [], [], [], {}, /(?=a)b/
      end

      # Specifies a new acronym. An acronym must be specified as it will appear
      # in a camelized string. An underscore string that contains the acronym
      # will retain the acronym when passed to +camelize+, +humanize+, or
      # +titleize+. A camelized string that contains the acronym will maintain
      # the acronym when titleized or humanized, and will convert the acronym
      # into a non-delimited single lowercase word when passed to +underscore+.
      #
      #   acronym 'HTML'
      #   titleize 'html'     # => 'HTML'
      #   camelize 'html'     # => 'HTML'
      #   underscore 'MyHTML' # => 'my_html'
      #
      # The acronym, however, must occur as a delimited unit and not be part of
      # another word for conversions to recognize it:
      #
      #   acronym 'HTTP'
      #   camelize 'my_http_delimited' # => 'MyHTTPDelimited'
      #   camelize 'https'             # => 'Https', not 'HTTPs'
      #   underscore 'HTTPS'           # => 'http_s', not 'https'
      #
      #   acronym 'HTTPS'
      #   camelize 'https'   # => 'HTTPS'
      #   underscore 'HTTPS' # => 'https'
      #
      # Note: Acronyms that are passed to +pluralize+ will no longer be
      # recognized, since the acronym will not occur as a delimited unit in the
      # pluralized result. To work around this, you must specify the pluralized
      # form as an acronym as well:
      #
      #    acronym 'API'
      #    camelize(pluralize('api')) # => 'Apis'
      #
      #    acronym 'APIs'
      #    camelize(pluralize('api')) # => 'APIs'
      #
      # +acronym+ may be used to specify any word that contains an acronym or
      # otherwise needs to maintain a non-standard capitalization. The only
      # restriction is that the word must begin with a capital letter.
      #
      #   acronym 'RESTful'
      #   underscore 'RESTful'           # => 'restful'
      #   underscore 'RESTfulController' # => 'restful_controller'
      #   titleize 'RESTfulController'   # => 'RESTful Controller'
      #   camelize 'restful'             # => 'RESTful'
      #   camelize 'restful_controller'  # => 'RESTfulController'
      #
      #   acronym 'McDonald'
      #   underscore 'McDonald' # => 'mcdonald'
      #   camelize 'mcdonald'   # => 'McDonald'
      def acronym(word)
        @acronyms[word.downcase] = word
        @acronym_regex = /#{@acronyms.values.join("|")}/
      end

      # Specifies a new pluralization rule and its replacement. The rule can
      # either be a string or a regular expression. The replacement should
      # always be a string that may include references to the matched data from
      # the rule.
      def plural(rule, replacement)
        @uncountables.delete(rule) if rule.is_a?(String)
        @uncountables.delete(replacement)
        @plurals.prepend([rule, replacement])
      end

      # Specifies a new singularization rule and its replacement. The rule can
      # either be a string or a regular expression. The replacement should
      # always be a string that may include references to the matched data from
      # the rule.
      def singular(rule, replacement)
        @uncountables.delete(rule) if rule.is_a?(String)
        @uncountables.delete(replacement)
        @singulars.prepend([rule, replacement])
      end

      # Specifies a new irregular that applies to both pluralization and
      # singularization at the same time. This can only be used for strings, not
      # regular expressions. You simply pass the irregular in singular and
      # plural form.
      #
      #   irregular 'octopus', 'octopi'
      #   irregular 'person', 'people'
      def irregular(singular, plural)
        @uncountables.delete(singular)
        @uncountables.delete(plural)

        s0 = singular[0]
        srest = singular[1..-1]

        p0 = plural[0]
        prest = plural[1..-1]

        if s0.upcase == p0.upcase
          plural(/(#{s0})#{srest}$/i, '\1' + prest)
          plural(/(#{p0})#{prest}$/i, '\1' + prest)

          singular(/(#{s0})#{srest}$/i, '\1' + srest)
          singular(/(#{p0})#{prest}$/i, '\1' + srest)
        else
          plural(/#{s0.upcase}(?i)#{srest}$/,   p0.upcase   + prest)
          plural(/#{s0.downcase}(?i)#{srest}$/, p0.downcase + prest)
          plural(/#{p0.upcase}(?i)#{prest}$/,   p0.upcase   + prest)
          plural(/#{p0.downcase}(?i)#{prest}$/, p0.downcase + prest)

          singular(/#{s0.upcase}(?i)#{srest}$/,   s0.upcase   + srest)
          singular(/#{s0.downcase}(?i)#{srest}$/, s0.downcase + srest)
          singular(/#{p0.upcase}(?i)#{prest}$/,   s0.upcase   + srest)
          singular(/#{p0.downcase}(?i)#{prest}$/, s0.downcase + srest)
        end
      end

      # Specifies words that are uncountable and should not be inflected.
      #
      #   uncountable 'money'
      #   uncountable 'money', 'information'
      #   uncountable %w( money information rice )
      def uncountable(*words)
        @uncountables += words.flatten.map(&:downcase)
      end

      # Specifies a humanized form of a string by a regular expression rule or
      # by a string mapping. When using a regular expression based replacement,
      # the normal humanize formatting is called after the replacement. When a
      # string is used, the human form should be specified as desired (example:
      # 'The name', not 'the_name').
      #
      #   human /_cnt$/i, '\1_count'
      #   human 'legacy_col_person_name', 'Name'
      def human(rule, replacement)
        @humans.prepend([rule, replacement])
      end
    end
  end
end
```


What does the above code do?
  - refunds credit card transactions
  - manages store inventory
  - converts english words into plurals

According to the above code, what is the plural of octopus?

What happens when acronyms are passed to pluralize?

What is the result of camelizing mcdonald?
  - Mcdonald
  - McDonald
  - mc_donald
  - mcdonald

Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data











--------
```rb

# encoding: utf-8

module ActiveSupport
  # The Inflector transforms words from singular to plural, class names to table
  # names, modularized class names to ones without, and class names to foreign
  # keys. The default inflections for pluralization, singularization, and
  # uncountable words are kept in inflections.rb.
  #
  # The Rails core team has stated patches for the inflections library will not
  # be accepted in order to avoid breaking legacy applications which may be
  # relying on errant inflections. If you discover an incorrect inflection and
  # require it for your application or wish to define rules for languages other
  # than English, please correct or add them yourself (explained below).
  module Inflector

    # Returns the plural form of the word in the string.
    #
    # If passed an optional +locale+ parameter, the word will be
    # pluralized using rules defined for that language. By default,
    # this parameter is set to <tt>:en</tt>.
    #
    #   'post'.pluralize             # => "posts"
    #   'octopus'.pluralize          # => "octopi"
    #   'sheep'.pluralize            # => "sheep"
    #   'words'.pluralize            # => "words"
    #   'CamelOctopus'.pluralize     # => "CamelOctopi"
    #   'ley'.pluralize(:es)         # => "leyes"
    def pluralize(word, locale = :en)
      apply_inflections(word, inflections(locale).plurals)
    end

    # The reverse of +pluralize+, returns the singular form of a word in a
    # string.
    #
    # If passed an optional +locale+ parameter, the word will be
    # singularized using rules defined for that language. By default,
    # this parameter is set to <tt>:en</tt>.
    #
    #   'posts'.singularize            # => "post"
    #   'octopi'.singularize           # => "octopus"
    #   'sheep'.singularize            # => "sheep"
    #   'word'.singularize             # => "word"
    #   'CamelOctopi'.singularize      # => "CamelOctopus"
    #   'leyes'.singularize(:es)       # => "ley"
    def singularize(word, locale = :en)
      apply_inflections(word, inflections(locale).singulars)
    end

    # By default, +camelize+ converts strings to UpperCamelCase. If the argument
    # to +camelize+ is set to <tt>:lower</tt> then +camelize+ produces
    # lowerCamelCase.
    #
    # +camelize+ will also convert '/' to '::' which is useful for converting
    # paths to namespaces.
    #
    #   'active_model'.camelize                # => "ActiveModel"
    #   'active_model'.camelize(:lower)        # => "activeModel"
    #   'active_model/errors'.camelize         # => "ActiveModel::Errors"
    #   'active_model/errors'.camelize(:lower) # => "activeModel::Errors"
    #
    # As a rule of thumb you can think of +camelize+ as the inverse of
    # +underscore+, though there are cases where that does not hold:
    #
    #   'SSLError'.underscore.camelize # => "SslError"
    def camelize(term, uppercase_first_letter = true)
      string = term.to_s
      if uppercase_first_letter
        string = string.sub(/^[a-z\d]*/) { inflections.acronyms[$&] || $&.capitalize }
      else
        string = string.sub(/^(?:#{inflections.acronym_regex}(?=\b|[A-Z_])|\w)/) { $&.downcase }
      end
      string.gsub!(/(?:_|(\/))([a-z\d]*)/i) { "#{$1}#{inflections.acronyms[$2] || $2.capitalize}" }
      string.gsub!(/\//, '::')
      string
    end

    # Makes an underscored, lowercase form from the expression in the string.
    #
    # Changes '::' to '/' to convert namespaces to paths.
    #
    #   'ActiveModel'.underscore         # => "active_model"
    #   'ActiveModel::Errors'.underscore # => "active_model/errors"
    #
    # As a rule of thumb you can think of +underscore+ as the inverse of
    # +camelize+, though there are cases where that does not hold:
    #
    #   'SSLError'.underscore.camelize # => "SslError"
    def underscore(camel_cased_word)
      return camel_cased_word unless camel_cased_word =~ /[A-Z-]|::/
      word = camel_cased_word.to_s.gsub(/::/, '/')
      word.gsub!(/(?:(?<=([A-Za-z\d]))|\b)(#{inflections.acronym_regex})(?=\b|[^a-z])/) { "#{$1 && '_'}#{$2.downcase}" }
      word.gsub!(/([A-Z\d]+)([A-Z][a-z])/,'\1_\2')
      word.gsub!(/([a-z\d])([A-Z])/,'\1_\2')
      word.tr!("-", "_")
      word.downcase!
      word
    end

    # Tweaks an attribute name for display to end users.
    #
    # Specifically, +humanize+ performs these transformations:
    #
    #   * Applies human inflection rules to the argument.
    #   * Deletes leading underscores, if any.
    #   * Removes a "_id" suffix if present.
    #   * Replaces underscores with spaces, if any.
    #   * Downcases all words except acronyms.
    #   * Capitalizes the first word.
    #
    # The capitalization of the first word can be turned off by setting the
    # +:capitalize+ option to false (default is true).
    #
    #   humanize('employee_salary')              # => "Employee salary"
    #   humanize('author_id')                    # => "Author"
    #   humanize('author_id', capitalize: false) # => "author"
    #   humanize('_id')                          # => "Id"
    #
    # If "SSL" was defined to be an acronym:
    #
    #   humanize('ssl_error') # => "SSL error"
    #
    def humanize(lower_case_and_underscored_word, options = {})
      result = lower_case_and_underscored_word.to_s.dup

      inflections.humans.each { |(rule, replacement)| break if result.sub!(rule, replacement) }

      result.sub!(/\A_+/, '')
      result.sub!(/_id\z/, '')
      result.tr!('_', ' ')

      result.gsub!(/([a-z\d]*)/i) do |match|
        "#{inflections.acronyms[match] || match.downcase}"
      end

      if options.fetch(:capitalize, true)
        result.sub!(/\A\w/) { |match| match.upcase }
      end

      result
    end

    # Capitalizes all the words and replaces some characters in the string to
    # create a nicer looking title. +titleize+ is meant for creating pretty
    # output. It is not used in the Rails internals.
    #
    # +titleize+ is also aliased as +titlecase+.
    #
    #   'man from the boondocks'.titleize   # => "Man From The Boondocks"
    #   'x-men: the last stand'.titleize    # => "X Men: The Last Stand"
    #   'TheManWithoutAPast'.titleize       # => "The Man Without A Past"
    #   'raiders_of_the_lost_ark'.titleize  # => "Raiders Of The Lost Ark"
    def titleize(word)
      humanize(underscore(word)).gsub(/\b(?<!['’`])[a-z]/) { $&.capitalize }
    end

    # Create the name of a table like Rails does for models to table names. This
    # method uses the +pluralize+ method on the last word in the string.
    #
    #   'RawScaledScorer'.tableize # => "raw_scaled_scorers"
    #   'egg_and_ham'.tableize     # => "egg_and_hams"
    #   'fancyCategory'.tableize   # => "fancy_categories"
    def tableize(class_name)
      pluralize(underscore(class_name))
    end

    # Create a class name from a plural table name like Rails does for table
    # names to models. Note that this returns a string and not a Class (To
    # convert to an actual class follow +classify+ with +constantize+).
    #
    #   'egg_and_hams'.classify # => "EggAndHam"
    #   'posts'.classify        # => "Post"
    #
    # Singular names are not handled correctly:
    #
    #   'calculus'.classify     # => "Calculu"
    def classify(table_name)
      # strip out any leading schema name
      camelize(singularize(table_name.to_s.sub(/.*\./, '')))
    end

    # Replaces underscores with dashes in the string.
    #
    #   'puni_puni'.dasherize # => "puni-puni"
    def dasherize(underscored_word)
      underscored_word.tr('_', '-')
    end

    # Removes the module part from the expression in the string.
    #
    #   'ActiveRecord::CoreExtensions::String::Inflections'.demodulize # => "Inflections"
    #   'Inflections'.demodulize                                       # => "Inflections"
    #   '::Inflections'.demodulize                                     # => "Inflections"
    #   ''.demodulize                                                  # => ""
    #
    # See also +deconstantize+.
    def demodulize(path)
      path = path.to_s
      if i = path.rindex('::')
        path[(i+2)..-1]
      else
        path
      end
    end

    # Removes the rightmost segment from the constant expression in the string.
    #
    #   'Net::HTTP'.deconstantize   # => "Net"
    #   '::Net::HTTP'.deconstantize # => "::Net"
    #   'String'.deconstantize      # => ""
    #   '::String'.deconstantize    # => ""
    #   ''.deconstantize            # => ""
    #
    # See also +demodulize+.
    def deconstantize(path)
      path.to_s[0, path.rindex('::') || 0] # implementation based on the one in facets' Module#spacename
    end

    # Tries to find a constant with the name specified in the argument string.
    #
    #   'Module'.constantize     # => Module
    #   'Test::Unit'.constantize # => Test::Unit
    #
    # The name is assumed to be the one of a top-level constant, no matter
    # whether it starts with "::" or not. No lexical context is taken into
    # account:
    #
    #   C = 'outside'
    #   module M
    #     C = 'inside'
    #     C               # => 'inside'
    #     'C'.constantize # => 'outside', same as ::C
    #   end
    #
    # NameError is raised when the name is not in CamelCase or the constant is
    # unknown.
    def constantize(camel_cased_word)
      names = camel_cased_word.split('::')

      # Trigger a built-in NameError exception including the ill-formed constant in the message.
      Object.const_get(camel_cased_word) if names.empty?

      # Remove the first blank element in case of '::ClassName' notation.
      names.shift if names.size > 1 && names.first.empty?

      names.inject(Object) do |constant, name|
        if constant == Object
          constant.const_get(name)
        else
          candidate = constant.const_get(name)
          next candidate if constant.const_defined?(name, false)
          next candidate unless Object.const_defined?(name)

          # Go down the ancestors to check if it is owned directly. The check
          # stops when we reach Object or the end of ancestors tree.
          constant = constant.ancestors.inject do |const, ancestor|
            break const    if ancestor == Object
            break ancestor if ancestor.const_defined?(name, false)
            const
          end

          # owner is in Object, so raise
          constant.const_get(name, false)
        end
      end
    end

    # Tries to find a constant with the name specified in the argument string.
    #
    #   'Module'.safe_constantize     # => Module
    #   'Test::Unit'.safe_constantize # => Test::Unit
    #
    # The name is assumed to be the one of a top-level constant, no matter
    # whether it starts with "::" or not. No lexical context is taken into
    # account:
    #
    #   C = 'outside'
    #   module M
    #     C = 'inside'
    #     C                    # => 'inside'
    #     'C'.safe_constantize # => 'outside', same as ::C
    #   end
    #
    # +nil+ is returned when the name is not in CamelCase or the constant (or
    # part of it) is unknown.
    #
    #   'blargle'.safe_constantize  # => nil
    #   'UnknownModule'.safe_constantize  # => nil
    #   'UnknownModule::Foo::Bar'.safe_constantize  # => nil
    def safe_constantize(camel_cased_word)
      constantize(camel_cased_word)
    rescue NameError => e
      raise if e.name && !(camel_cased_word.to_s.split("::").include?(e.name.to_s) ||
        e.name.to_s == camel_cased_word.to_s)
    rescue ArgumentError => e
      raise unless e.message =~ /not missing constant #{const_regexp(camel_cased_word)}\!$/
    end

    # Returns the suffix that should be added to a number to denote the position
    # in an ordered sequence such as 1st, 2nd, 3rd, 4th.
    #
    #   ordinal(1)     # => "st"
    #   ordinal(2)     # => "nd"
    #   ordinal(1002)  # => "nd"
    #   ordinal(1003)  # => "rd"
    #   ordinal(-11)   # => "th"
    #   ordinal(-1021) # => "st"
    def ordinal(number)
      abs_number = number.to_i.abs

      if (11..13).include?(abs_number % 100)
        "th"
      else
        case abs_number % 10
          when 1; "st"
          when 2; "nd"
          when 3; "rd"
          else    "th"
        end
      end
    end

    # Turns a number into an ordinal string used to denote the position in an
    # ordered sequence such as 1st, 2nd, 3rd, 4th.
    #
    #   ordinalize(1)     # => "1st"
    #   ordinalize(2)     # => "2nd"
    #   ordinalize(1002)  # => "1002nd"
    #   ordinalize(1003)  # => "1003rd"
    #   ordinalize(-11)   # => "-11th"
    #   ordinalize(-1021) # => "-1021st"
    def ordinalize(number)
      "#{number}#{ordinal(number)}"
    end

    private

    # Mounts a regular expression, returned as a string to ease interpolation,
    # that will match part by part the given constant.
    #
    #   const_regexp("Foo::Bar::Baz") # => "Foo(::Bar(::Baz)?)?"
    #   const_regexp("::")            # => "::"
    def const_regexp(camel_cased_word) #:nodoc:
      parts = camel_cased_word.split("::")

      return Regexp.escape(camel_cased_word) if parts.blank?

      last  = parts.pop

      parts.reverse.inject(last) do |acc, part|
        part.empty? ? acc : "#{part}(::#{acc})?"
      end
    end

    # Applies inflection rules for +singularize+ and +pluralize+.
    #
    #  apply_inflections('post', inflections.plurals)    # => "posts"
    #  apply_inflections('posts', inflections.singulars) # => "post"
    def apply_inflections(word, rules)
      result = word.to_s.dup

      if word.empty? || inflections.uncountables.include?(result.downcase[/\b\w+\Z/])
        result
      else
        rules.each { |(rule, replacement)| break if result.sub!(rule, replacement) }
        result
      end
    end
  end
end
```


Which is not supported in the above code?
  - camelize
  - underscore
  - pluralize
  - singularize
  - constantize
  - ordinal
  - uppercase
  - dasherize

What line appends "st" to 1 to make it "1st"?

What happens when Calculus is classified?

What happens when two concatenated words are pluralized, e.g. 'DogCamel'?
  - 'DogsCamels'
  - 'Dogs Camels'
  - 'DogCamels'

Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data


















---------

```rb
module ActiveSupport
  module NumberHelper
    class NumberToPhoneConverter < NumberConverter #:nodoc:
      def convert
        str  = country_code(opts[:country_code])
        str << convert_to_phone_number(number.to_s.strip)
        str << phone_ext(opts[:extension])
      end

      private

        def convert_to_phone_number(number)
          if opts[:area_code]
            convert_with_area_code(number)
          else
            convert_without_area_code(number)
          end
        end

        def convert_with_area_code(number)
          number.gsub!(/(\d{1,3})(\d{3})(\d{4}$)/,"(\\1) \\2#{delimiter}\\3")
          number
        end

        def convert_without_area_code(number)
          number.gsub!(/(\d{0,3})(\d{3})(\d{4})$/,"\\1#{delimiter}\\2#{delimiter}\\3")
          number.slice!(0, 1) if start_with_delimiter?(number)
          number
        end

        def start_with_delimiter?(number)
          delimiter.present? && number.start_with?(delimiter)
        end

        def delimiter
          opts[:delimiter] || "-"
        end

        def country_code(code)
          code.blank? ? "" : "+#{code}#{delimiter}"
        end

        def phone_ext(ext)
          ext.blank? ? "" : " x #{ext}"
        end
    end
  end
end
```

How is an extension represented by the above code?
  - ext 123
  - x 123
  - 123

The default phone number delimiter is:
  - a "."
  - a "-"
  - a ","
  - there is none

The above code can only support American phone numbers.
  - true
  - false

Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data


















-----------

```js

d3.rgb = d3_rgb;

function d3_rgb(r, g, b) {
  return this instanceof d3_rgb ? void (this.r = ~~r, this.g = ~~g, this.b = ~~b)
      : arguments.length < 2 ? (r instanceof d3_rgb ? new d3_rgb(r.r, r.g, r.b)
      : d3_rgb_parse("" + r, d3_rgb, d3_hsl_rgb))
      : new d3_rgb(r, g, b);
}

function d3_rgbNumber(value) {
  return new d3_rgb(value >> 16, value >> 8 & 0xff, value & 0xff);
}

function d3_rgbString(value) {
  return d3_rgbNumber(value) + "";
}

var d3_rgbPrototype = d3_rgb.prototype = new d3_color;

d3_rgbPrototype.brighter = function(k) {
  k = Math.pow(0.7, arguments.length ? k : 1);
  var r = this.r,
      g = this.g,
      b = this.b,
      i = 30;
  if (!r && !g && !b) return new d3_rgb(i, i, i);
  if (r && r < i) r = i;
  if (g && g < i) g = i;
  if (b && b < i) b = i;
  return new d3_rgb(Math.min(255, r / k), Math.min(255, g / k), Math.min(255, b / k));
};

d3_rgbPrototype.darker = function(k) {
  k = Math.pow(0.7, arguments.length ? k : 1);
  return new d3_rgb(k * this.r, k * this.g, k * this.b);
};

d3_rgbPrototype.hsl = function() {
  return d3_rgb_hsl(this.r, this.g, this.b);
};

d3_rgbPrototype.toString = function() {
  return "#" + d3_rgb_hex(this.r) + d3_rgb_hex(this.g) + d3_rgb_hex(this.b);
};

function d3_rgb_hex(v) {
  return v < 0x10
      ? "0" + Math.max(0, v).toString(16)
      : Math.min(255, v).toString(16);
}

function d3_rgb_parse(format, rgb, hsl) {
  var r = 0, // red channel; int in [0, 255]
      g = 0, // green channel; int in [0, 255]
      b = 0, // blue channel; int in [0, 255]
      m1, // CSS color specification match
      m2, // CSS color specification type (e.g., rgb)
      color;

  /* Handle hsl, rgb. */
  m1 = /([a-z]+)\((.*)\)/i.exec(format);
  if (m1) {
    m2 = m1[2].split(",");
    switch (m1[1]) {
      case "hsl": {
        return hsl(
          parseFloat(m2[0]), // degrees
          parseFloat(m2[1]) / 100, // percentage
          parseFloat(m2[2]) / 100 // percentage
        );
      }
      case "rgb": {
        return rgb(
          d3_rgb_parseNumber(m2[0]),
          d3_rgb_parseNumber(m2[1]),
          d3_rgb_parseNumber(m2[2])
        );
      }
    }
  }

  /* Named colors. */
  if (color = d3_rgb_names.get(format)) return rgb(color.r, color.g, color.b);

  /* Hexadecimal colors: #rgb and #rrggbb. */
  if (format != null && format.charAt(0) === "#" && !isNaN(color = parseInt(format.slice(1), 16))) {
    if (format.length === 4) {
      r = (color & 0xf00) >> 4; r = (r >> 4) | r;
      g = (color & 0xf0); g = (g >> 4) | g;
      b = (color & 0xf); b = (b << 4) | b;
    } else if (format.length === 7) {
      r = (color & 0xff0000) >> 16;
      g = (color & 0xff00) >> 8;
      b = (color & 0xff);
    }
  }

  return rgb(r, g, b);
}

function d3_rgb_hsl(r, g, b) {
  var min = Math.min(r /= 255, g /= 255, b /= 255),
      max = Math.max(r, g, b),
      d = max - min,
      h,
      s,
      l = (max + min) / 2;
  if (d) {
    s = l < .5 ? d / (max + min) : d / (2 - max - min);
    if (r == max) h = (g - b) / d + (g < b ? 6 : 0);
    else if (g == max) h = (b - r) / d + 2;
    else h = (r - g) / d + 4;
    h *= 60;
  } else {
    h = NaN;
    s = l > 0 && l < 1 ? 0 : h;
  }
  return new d3_hsl(h, s, l);
}

function d3_rgb_lab(r, g, b) {
  r = d3_rgb_xyz(r);
  g = d3_rgb_xyz(g);
  b = d3_rgb_xyz(b);
  var x = d3_xyz_lab((0.4124564 * r + 0.3575761 * g + 0.1804375 * b) / d3_lab_X),
      y = d3_xyz_lab((0.2126729 * r + 0.7151522 * g + 0.0721750 * b) / d3_lab_Y),
      z = d3_xyz_lab((0.0193339 * r + 0.1191920 * g + 0.9503041 * b) / d3_lab_Z);
  return d3_lab(116 * y - 16, 500 * (x - y), 200 * (y - z));
}

function d3_rgb_xyz(r) {
  return (r /= 255) <= 0.04045 ? r / 12.92 : Math.pow((r + 0.055) / 1.055, 2.4);
}

function d3_rgb_parseNumber(c) { // either integer or percentage
  var f = parseFloat(c);
  return c.charAt(c.length - 1) === "%" ? Math.round(f * 2.55) : f;
}

var d3_rgb_names = d3.map({
  aliceblue: 0xf0f8ff,
  antiquewhite: 0xfaebd7,
  aqua: 0x00ffff,
  aquamarine: 0x7fffd4,
  azure: 0xf0ffff,
  beige: 0xf5f5dc,
  bisque: 0xffe4c4,
  black: 0x000000,
  blanchedalmond: 0xffebcd,
  blue: 0x0000ff,
  blueviolet: 0x8a2be2,
  brown: 0xa52a2a,
  burlywood: 0xdeb887,
  cadetblue: 0x5f9ea0,
  chartreuse: 0x7fff00,
  chocolate: 0xd2691e,
  coral: 0xff7f50,
  cornflowerblue: 0x6495ed,
  cornsilk: 0xfff8dc,
  crimson: 0xdc143c,
  cyan: 0x00ffff,
  darkblue: 0x00008b,
  darkcyan: 0x008b8b,
  darkgoldenrod: 0xb8860b,
  darkgray: 0xa9a9a9,
  darkgreen: 0x006400,
  darkgrey: 0xa9a9a9,
  darkkhaki: 0xbdb76b,
  darkmagenta: 0x8b008b,
  darkolivegreen: 0x556b2f,
  darkorange: 0xff8c00,
  darkorchid: 0x9932cc,
  darkred: 0x8b0000,
  darksalmon: 0xe9967a,
  darkseagreen: 0x8fbc8f,
  darkslateblue: 0x483d8b,
  darkslategray: 0x2f4f4f,
  darkslategrey: 0x2f4f4f,
  darkturquoise: 0x00ced1,
  darkviolet: 0x9400d3,
  deeppink: 0xff1493,
  deepskyblue: 0x00bfff,
  dimgray: 0x696969,
  dimgrey: 0x696969,
  dodgerblue: 0x1e90ff,
  firebrick: 0xb22222,
  floralwhite: 0xfffaf0,
  forestgreen: 0x228b22,
  fuchsia: 0xff00ff,
  gainsboro: 0xdcdcdc,
  ghostwhite: 0xf8f8ff,
  gold: 0xffd700,
  goldenrod: 0xdaa520,
  gray: 0x808080,
  green: 0x008000,
  greenyellow: 0xadff2f,
  grey: 0x808080,
  honeydew: 0xf0fff0,
  hotpink: 0xff69b4,
  indianred: 0xcd5c5c,
  indigo: 0x4b0082,
  ivory: 0xfffff0,
  khaki: 0xf0e68c,
  lavender: 0xe6e6fa,
  lavenderblush: 0xfff0f5,
  lawngreen: 0x7cfc00,
  lemonchiffon: 0xfffacd,
  lightblue: 0xadd8e6,
  lightcoral: 0xf08080,
  lightcyan: 0xe0ffff,
  lightgoldenrodyellow: 0xfafad2,
  lightgray: 0xd3d3d3,
  lightgreen: 0x90ee90,
  lightgrey: 0xd3d3d3,
  lightpink: 0xffb6c1,
  lightsalmon: 0xffa07a,
  lightseagreen: 0x20b2aa,
  lightskyblue: 0x87cefa,
  lightslategray: 0x778899,
  lightslategrey: 0x778899,
  lightsteelblue: 0xb0c4de,
  lightyellow: 0xffffe0,
  lime: 0x00ff00,
  limegreen: 0x32cd32,
  linen: 0xfaf0e6,
  magenta: 0xff00ff,
  maroon: 0x800000,
  mediumaquamarine: 0x66cdaa,
  mediumblue: 0x0000cd,
  mediumorchid: 0xba55d3,
  mediumpurple: 0x9370db,
  mediumseagreen: 0x3cb371,
  mediumslateblue: 0x7b68ee,
  mediumspringgreen: 0x00fa9a,
  mediumturquoise: 0x48d1cc,
  mediumvioletred: 0xc71585,
  midnightblue: 0x191970,
  mintcream: 0xf5fffa,
  mistyrose: 0xffe4e1,
  moccasin: 0xffe4b5,
  navajowhite: 0xffdead,
  navy: 0x000080,
  oldlace: 0xfdf5e6,
  olive: 0x808000,
  olivedrab: 0x6b8e23,
  orange: 0xffa500,
  orangered: 0xff4500,
  orchid: 0xda70d6,
  palegoldenrod: 0xeee8aa,
  palegreen: 0x98fb98,
  paleturquoise: 0xafeeee,
  palevioletred: 0xdb7093,
  papayawhip: 0xffefd5,
  peachpuff: 0xffdab9,
  peru: 0xcd853f,
  pink: 0xffc0cb,
  plum: 0xdda0dd,
  powderblue: 0xb0e0e6,
  purple: 0x800080,
  red: 0xff0000,
  rosybrown: 0xbc8f8f,
  royalblue: 0x4169e1,
  saddlebrown: 0x8b4513,
  salmon: 0xfa8072,
  sandybrown: 0xf4a460,
  seagreen: 0x2e8b57,
  seashell: 0xfff5ee,
  sienna: 0xa0522d,
  silver: 0xc0c0c0,
  skyblue: 0x87ceeb,
  slateblue: 0x6a5acd,
  slategray: 0x708090,
  slategrey: 0x708090,
  snow: 0xfffafa,
  springgreen: 0x00ff7f,
  steelblue: 0x4682b4,
  tan: 0xd2b48c,
  teal: 0x008080,
  thistle: 0xd8bfd8,
  tomato: 0xff6347,
  turquoise: 0x40e0d0,
  violet: 0xee82ee,
  wheat: 0xf5deb3,
  white: 0xffffff,
  whitesmoke: 0xf5f5f5,
  yellow: 0xffff00,
  yellowgreen: 0x9acd32
});

d3_rgb_names.forEach(function(key, value) {
  d3_rgb_names.set(key, d3_rgbNumber(value));
});
```

The above code deals with:
  - refunding credit card transactions
  - pluralizing english text
  - converting colors to red, green, blue (rgb) values

What lines of code store the names of colors?

If we wanted to darken a color, what line of code would we look to?

How does that line of code work?
  - it reduces each rgb value
  - it increases each rgb value
  - it increases the a value in an rgba number

Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data













-----

```js
/*
 * Chartkick.js
 * Create beautiful Javascript charts with minimal code
 * https://github.com/ankane/chartkick.js
 * v1.2.2
 * MIT License
 */

/*jslint browser: true, indent: 2, plusplus: true, vars: true */

(function (window) {
  'use strict';

  var config = window.Chartkick || {};
  var Chartkick, ISO8601_PATTERN, DECIMAL_SEPARATOR, adapters = [];

  // helpers

  function isArray(variable) {
    return Object.prototype.toString.call(variable) === "[object Array]";
  }

  function isFunction(variable) {
    return variable instanceof Function;
  }

  function isPlainObject(variable) {
    return !isFunction(variable) && variable instanceof Object;
  }

  // https://github.com/madrobby/zepto/blob/master/src/zepto.js
  function extend(target, source) {
    var key;
    for (key in source) {
      if (isPlainObject(source[key]) || isArray(source[key])) {
        if (isPlainObject(source[key]) && !isPlainObject(target[key])) {
          target[key] = {};
        }
        if (isArray(source[key]) && !isArray(target[key])) {
          target[key] = [];
        }
        extend(target[key], source[key]);
      } else if (source[key] !== undefined) {
        target[key] = source[key];
      }
    }
  }

  function merge(obj1, obj2) {
    var target = {};
    extend(target, obj1);
    extend(target, obj2);
    return target;
  }

  // https://github.com/Do/iso8601.js
  ISO8601_PATTERN = /(\d\d\d\d)(\-)?(\d\d)(\-)?(\d\d)(T)?(\d\d)(:)?(\d\d)?(:)?(\d\d)?([\.,]\d+)?($|Z|([\+\-])(\d\d)(:)?(\d\d)?)/i;
  DECIMAL_SEPARATOR = String(1.5).charAt(1);

  function parseISO8601(input) {
    var day, hour, matches, milliseconds, minutes, month, offset, result, seconds, type, year;
    type = Object.prototype.toString.call(input);
    if (type === '[object Date]') {
      return input;
    }
    if (type !== '[object String]') {
      return;
    }
    if (matches = input.match(ISO8601_PATTERN)) {
      year = parseInt(matches[1], 10);
      month = parseInt(matches[3], 10) - 1;
      day = parseInt(matches[5], 10);
      hour = parseInt(matches[7], 10);
      minutes = matches[9] ? parseInt(matches[9], 10) : 0;
      seconds = matches[11] ? parseInt(matches[11], 10) : 0;
      milliseconds = matches[12] ? parseFloat(DECIMAL_SEPARATOR + matches[12].slice(1)) * 1000 : 0;
      result = Date.UTC(year, month, day, hour, minutes, seconds, milliseconds);
      if (matches[13] && matches[14]) {
        offset = matches[15] * 60;
        if (matches[17]) {
          offset += parseInt(matches[17], 10);
        }
        offset *= matches[14] === '-' ? -1 : 1;
        result -= offset * 60 * 1000;
      }
      return new Date(result);
    }
  }
  // end iso8601.js

  function negativeValues(series) {
    var i, j, data;
    for (i = 0; i < series.length; i++) {
      data = series[i].data;
      for (j = 0; j < data.length; j++) {
        if (data[j][1] < 0) {
          return true;
        }
      }
    }
    return false;
  }

  function jsOptionsFunc(defaultOptions, hideLegend, setMin, setMax, setStacked) {
    return function (series, opts, chartOptions) {
      var options = merge({}, defaultOptions);
      options = merge(options, chartOptions || {});

      // hide legend
      // this is *not* an external option!
      if (opts.hideLegend) {
        hideLegend(options);
      }

      // min
      if ("min" in opts) {
        setMin(options, opts.min);
      } else if (!negativeValues(series)) {
        setMin(options, 0);
      }

      // max
      if ("max" in opts) {
        setMax(options, opts.max);
      }

      if (opts.stacked) {
        setStacked(options);
      }

      if (opts.colors) {
        options.colors = opts.colors;
      }

      // merge library last
      options = merge(options, opts.library || {});

      return options;
    };
  }

  function setText(element, text) {
    if (document.body.innerText) {
      element.innerText = text;
    } else {
      element.textContent = text;
    }
  }

  function chartError(element, message) {
    setText(element, "Error Loading Chart: " + message);
    element.style.color = "#ff0000";
  }

  function getJSON(element, url, success) {
    var $ = window.jQuery || window.Zepto || window.$;
    $.ajax({
      dataType: "json",
      url: url,
      success: success,
      error: function (jqXHR, textStatus, errorThrown) {
        var message = (typeof errorThrown === "string") ? errorThrown : errorThrown.message;
        chartError(element, message);
      }
    });
  }

  function errorCatcher(chart, callback) {
    try {
      callback(chart);
    } catch (err) {
      chartError(chart.element, err.message);
      throw err;
    }
  }

  function fetchDataSource(chart, callback) {
    if (typeof chart.dataSource === "string") {
      getJSON(chart.element, chart.dataSource, function (data, textStatus, jqXHR) {
        chart.data = data;
        errorCatcher(chart, callback);
      });
    } else {
      chart.data = chart.dataSource;
      errorCatcher(chart, callback);
    }
  }

  // type conversions

  function toStr(n) {
    return "" + n;
  }

  function toFloat(n) {
    return parseFloat(n);
  }

  function toDate(n) {
    if (typeof n !== "object") {
      if (typeof n === "number") {
        n = new Date(n * 1000); // ms
      } else { // str
        // try our best to get the str into iso8601
        // TODO be smarter about this
        var str = n.replace(/ /, "T").replace(" ", "").replace("UTC", "Z");
        n = parseISO8601(str) || new Date(n);
      }
    }
    return n;
  }

  function toArr(n) {
    if (!isArray(n)) {
      var arr = [], i;
      for (i in n) {
        if (n.hasOwnProperty(i)) {
          arr.push([i, n[i]]);
        }
      }
      n = arr;
    }
    return n;
  }

  function sortByTime(a, b) {
    return a[0].getTime() - b[0].getTime();
  }

  if ("Highcharts" in window) {
    var HighchartsAdapter = new function () {
      var Highcharts = window.Highcharts;

      var defaultOptions = {
        chart: {},
        xAxis: {
          labels: {
            style: {
              fontSize: "12px"
            }
          }
        },
        yAxis: {
          title: {
            text: null
          },
          labels: {
            style: {
              fontSize: "12px"
            }
          }
        },
        title: {
          text: null
        },
        credits: {
          enabled: false
        },
        legend: {
          borderWidth: 0
        },
        tooltip: {
          style: {
            fontSize: "12px"
          }
        },
        plotOptions: {
          areaspline: {},
          series: {
            marker: {}
          }
        }
      };

      var hideLegend = function (options) {
        options.legend.enabled = false;
      };

      var setMin = function (options, min) {
        options.yAxis.min = min;
      };

      var setMax = function (options, max) {
        options.yAxis.max = max;
      };

      var setStacked = function (options) {
        options.plotOptions.series.stacking = "normal";
      };

      var jsOptions = jsOptionsFunc(defaultOptions, hideLegend, setMin, setMax, setStacked);

      this.renderLineChart = function (chart, chartType) {
        chartType = chartType || "spline";
        var chartOptions = {};
        if (chartType === "areaspline") {
          chartOptions = {
            plotOptions: {
              areaspline: {
                stacking: "normal"
              },
              series: {
                marker: {
                  enabled: false
                }
              }
            }
          };
        }
        var options = jsOptions(chart.data, chart.options, chartOptions), data, i, j;
        options.xAxis.type = chart.options.discrete ? "category" : "datetime";
        options.chart.type = chartType;
        options.chart.renderTo = chart.element.id;

        var series = chart.data;
        for (i = 0; i < series.length; i++) {
          data = series[i].data;
          if (!chart.options.discrete) {
            for (j = 0; j < data.length; j++) {
              data[j][0] = data[j][0].getTime();
            }
          }
          series[i].marker = {symbol: "circle"};
        }
        options.series = series;
        new Highcharts.Chart(options);
      };

      this.renderPieChart = function (chart) {
        var chartOptions = {};
        if (chart.options.colors) {
          chartOptions.colors = chart.options.colors;
        }
        var options = merge(merge(defaultOptions, chartOptions), chart.options.library || {});
        options.chart.renderTo = chart.element.id;
        options.series = [{
          type: "pie",
          name: "Value",
          data: chart.data
        }];
        new Highcharts.Chart(options);
      };

      this.renderColumnChart = function (chart, chartType) {
        var chartType = chartType || "column";
        var series = chart.data;
        var options = jsOptions(series, chart.options), i, j, s, d, rows = [];
        options.chart.type = chartType;
        options.chart.renderTo = chart.element.id;

        for (i = 0; i < series.length; i++) {
          s = series[i];

          for (j = 0; j < s.data.length; j++) {
            d = s.data[j];
            if (!rows[d[0]]) {
              rows[d[0]] = new Array(series.length);
            }
            rows[d[0]][i] = d[1];
          }
        }

        var categories = [];
        for (i in rows) {
          if (rows.hasOwnProperty(i)) {
            categories.push(i);
          }
        }
        options.xAxis.categories = categories;

        var newSeries = [];
        for (i = 0; i < series.length; i++) {
          d = [];
          for (j = 0; j < categories.length; j++) {
            d.push(rows[categories[j]][i] || 0);
          }

          newSeries.push({
            name: series[i].name,
            data: d
          });
        }
        options.series = newSeries;

        new Highcharts.Chart(options);
      };

      var self = this;

      this.renderBarChart = function (chart) {
        self.renderColumnChart(chart, "bar");
      };

      this.renderAreaChart = function (chart) {
        self.renderLineChart(chart, "areaspline");
      };
    };
    adapters.push(HighchartsAdapter);
  }
  if (window.google && window.google.setOnLoadCallback) {
    var GoogleChartsAdapter = new function () {
      var google = window.google;

      var loaded = {};
      var callbacks = [];

      var runCallbacks = function () {
        var cb, call;
        for (var i = 0; i < callbacks.length; i++) {
          cb = callbacks[i];
          call = google.visualization && ((cb.pack == "corechart" && google.visualization.LineChart) || (cb.pack == "timeline" && google.visualization.Timeline))
          if (call) {
            cb.callback();
            callbacks.splice(i, 1);
            i--;
          }
        }
      };

      var waitForLoaded = function (pack, callback) {
        if (!callback) {
          callback = pack;
          pack = "corechart";
        }

        callbacks.push({pack: pack, callback: callback});

        if (loaded[pack]) {
          runCallbacks();
        } else {
          loaded[pack] = true;

          // https://groups.google.com/forum/#!topic/google-visualization-api/fMKJcyA2yyI
          var loadOptions = {
            packages: [pack],
            callback: runCallbacks
          };
          if (config.language) {
            loadOptions.language = config.language;
          }
          google.load("visualization", "1", loadOptions);
        }
      };

      // Set chart options
      var defaultOptions = {
        chartArea: {},
        fontName: "'Lucida Grande', 'Lucida Sans Unicode', Verdana, Arial, Helvetica, sans-serif",
        pointSize: 6,
        legend: {
          textStyle: {
            fontSize: 12,
            color: "#444"
          },
          alignment: "center",
          position: "right"
        },
        curveType: "function",
        hAxis: {
          textStyle: {
            color: "#666",
            fontSize: 12
          },
          gridlines: {
            color: "transparent"
          },
          baselineColor: "#ccc",
          viewWindow: {}
        },
        vAxis: {
          textStyle: {
            color: "#666",
            fontSize: 12
          },
          baselineColor: "#ccc",
          viewWindow: {}
        },
        tooltip: {
          textStyle: {
            color: "#666",
            fontSize: 12
          }
        }
      };

      var hideLegend = function (options) {
        options.legend.position = "none";
      };

      var setMin = function (options, min) {
        options.vAxis.viewWindow.min = min;
      };

      var setMax = function (options, max) {
        options.vAxis.viewWindow.max = max;
      };

      var setBarMin = function (options, min) {
        options.hAxis.viewWindow.min = min;
      };

      var setBarMax = function (options, max) {
        options.hAxis.viewWindow.max = max;
      };

      var setStacked = function (options) {
        options.isStacked = true;
      };

      var jsOptions = jsOptionsFunc(defaultOptions, hideLegend, setMin, setMax, setStacked);

      // cant use object as key
      var createDataTable = function (series, columnType) {
        var data = new google.visualization.DataTable();
        data.addColumn(columnType, "");

        var i, j, s, d, key, rows = [];
        for (i = 0; i < series.length; i++) {
          s = series[i];
          data.addColumn("number", s.name);

          for (j = 0; j < s.data.length; j++) {
            d = s.data[j];
            key = (columnType === "datetime") ? d[0].getTime() : d[0];
            if (!rows[key]) {
              rows[key] = new Array(series.length);
            }
            rows[key][i] = toFloat(d[1]);
          }
        }

        var rows2 = [];
        for (i in rows) {
          if (rows.hasOwnProperty(i)) {
            rows2.push([(columnType === "datetime") ? new Date(toFloat(i)) : i].concat(rows[i]));
          }
        }
        if (columnType === "datetime") {
          rows2.sort(sortByTime);
        }
        data.addRows(rows2);

        return data;
      };

      var resize = function (callback) {
        if (window.attachEvent) {
          window.attachEvent("onresize", callback);
        } else if (window.addEventListener) {
          window.addEventListener("resize", callback, true);
        }
        callback();
      };

      this.renderLineChart = function (chart) {
        waitForLoaded(function () {
          var options = jsOptions(chart.data, chart.options);
          var data = createDataTable(chart.data, chart.options.discrete ? "string" : "datetime");
          chart.chart = new google.visualization.LineChart(chart.element);
          resize(function () {
            chart.chart.draw(data, options);
          });
        });
      };

      this.renderPieChart = function (chart) {
        waitForLoaded(function () {
          var chartOptions = {
            chartArea: {
              top: "10%",
              height: "80%"
            }
          };
          if (chart.options.colors) {
            chartOptions.colors = chart.options.colors;
          }
          var options = merge(merge(defaultOptions, chartOptions), chart.options.library || {});

          var data = new google.visualization.DataTable();
          data.addColumn("string", "");
          data.addColumn("number", "Value");
          data.addRows(chart.data);

          chart.chart = new google.visualization.PieChart(chart.element);
          resize(function () {
            chart.chart.draw(data, options);
          });
        });
      };

      this.renderColumnChart = function (chart) {
        waitForLoaded(function () {
          var options = jsOptions(chart.data, chart.options);
          var data = createDataTable(chart.data, "string");
          chart.chart = new google.visualization.ColumnChart(chart.element);
          resize(function () {
            chart.chart.draw(data, options);
          });
        });
      };

      this.renderBarChart = function (chart) {
        waitForLoaded(function () {
          var chartOptions = {
            hAxis: {
              gridlines: {
                color: "#ccc"
              }
            }
          };
          var options = jsOptionsFunc(defaultOptions, hideLegend, setBarMin, setBarMax, setStacked)(chart.data, chart.options, chartOptions);
          var data = createDataTable(chart.data, "string");
          chart.chart = new google.visualization.BarChart(chart.element);
          resize(function () {
            chart.chart.draw(data, options);
          });
        });
      };

      this.renderAreaChart = function (chart) {
        waitForLoaded(function () {
          var chartOptions = {
            isStacked: true,
            pointSize: 0,
            areaOpacity: 0.5
          };
          var options = jsOptions(chart.data, chart.options, chartOptions);
          var data = createDataTable(chart.data, chart.options.discrete ? "string" : "datetime");
          chart.chart = new google.visualization.AreaChart(chart.element);
          resize(function () {
            chart.chart.draw(data, options);
          });
        });
      };

      this.renderGeoChart = function (chart) {
        waitForLoaded(function () {
          var chartOptions = {
            legend: "none",
            colorAxis: {
              colors: chart.options.colors || ["#f6c7b6", "#ce502d"]
            }
          };
          var options = merge(merge(defaultOptions, chartOptions), chart.options.library || {});

          var data = new google.visualization.DataTable();
          data.addColumn("string", "");
          data.addColumn("number", "Value");
          data.addRows(chart.data);

          chart.chart = new google.visualization.GeoChart(chart.element);
          resize(function () {
            chart.chart.draw(data, options);
          });
        });
      };

      this.renderTimeline = function (chart) {
        waitForLoaded("timeline", function () {
          var chartOptions = {
            legend: "none"
          };

          if (chart.options.colors) {
            chartOptions.colorAxis.colors = chart.options.colors;
          }
          var options = merge(merge(defaultOptions, chartOptions), chart.options.library || {});

          var data = new google.visualization.DataTable();
          data.addColumn({type: "string", id: "Name"});
          data.addColumn({type: "date", id: "Start"});
          data.addColumn({type: "date", id: "End"});
          data.addRows(chart.data);

          chart.chart = new google.visualization.Timeline(chart.element);

          resize(function () {
            chart.chart.draw(data, options);
          });
        });
      };
    };

    adapters.push(GoogleChartsAdapter);
  }

  // TODO add adapter option
  // TODO remove chartType if cross-browser way
  // to get the name of the chart class
  function renderChart(chartType, chart) {
    var i, adapter, fnName;
    fnName = "render" + chartType;

    for (i = 0; i < adapters.length; i++) {
      adapter = adapters[i];
      if (isFunction(adapter[fnName])) {
        return adapter[fnName](chart);
      }
    }
    throw new Error("No adapter found");
  }

  // process data

  function processSeries(series, opts, time) {
    var i, j, data, r, key;

    // see if one series or multiple
    if (!isArray(series) || typeof series[0] !== "object" || isArray(series[0])) {
      series = [{name: "Value", data: series}];
      opts.hideLegend = true;
    } else {
      opts.hideLegend = false;
    }
    if (opts.discrete) {
      time = false;
    }

    // right format
    for (i = 0; i < series.length; i++) {
      data = toArr(series[i].data);
      r = [];
      for (j = 0; j < data.length; j++) {
        key = data[j][0];
        key = time ? toDate(key) : toStr(key);
        r.push([key, toFloat(data[j][1])]);
      }
      if (time) {
        r.sort(sortByTime);
      }
      series[i].data = r;
    }

    return series;
  }

  function processSimple(data) {
    var perfectData = toArr(data), i;
    for (i = 0; i < perfectData.length; i++) {
      perfectData[i] = [toStr(perfectData[i][0]), toFloat(perfectData[i][1])];
    }
    return perfectData;
  }

  function processTime(data)
  {
    var i;
    for (i = 0; i < data.length; i++) {
      data[i][1] = toDate(data[i][1]);
      data[i][2] = toDate(data[i][2]);
    }
    return data;
  }

  function processLineData(chart) {
    chart.data = processSeries(chart.data, chart.options, true);
    renderChart("LineChart", chart);
  }

  function processColumnData(chart) {
    chart.data = processSeries(chart.data, chart.options, false);
    renderChart("ColumnChart", chart);
  }

  function processPieData(chart) {
    chart.data = processSimple(chart.data);
    renderChart("PieChart", chart);
  }

  function processBarData(chart) {
    chart.data = processSeries(chart.data, chart.options, false);
    renderChart("BarChart", chart);
  }

  function processAreaData(chart) {
    chart.data = processSeries(chart.data, chart.options, true);
    renderChart("AreaChart", chart);
  }

  function processGeoData(chart) {
    chart.data = processSimple(chart.data);
    renderChart("GeoChart", chart);
  }

  function processTimelineData(chart) {
    chart.data = processTime(chart.data);
    renderChart("Timeline", chart);
  }

  function setElement(chart, element, dataSource, opts, callback) {
    if (typeof element === "string") {
      element = document.getElementById(element);
    }
    chart.element = element;
    chart.options = opts || {};
    chart.dataSource = dataSource;
    Chartkick.charts[element.id] = chart;
    fetchDataSource(chart, callback);
  }

  // define classes

  Chartkick = {
    LineChart: function (element, dataSource, opts) {
      setElement(this, element, dataSource, opts, processLineData);
    },
    PieChart: function (element, dataSource, opts) {
      setElement(this, element, dataSource, opts, processPieData);
    },
    ColumnChart: function (element, dataSource, opts) {
      setElement(this, element, dataSource, opts, processColumnData);
    },
    BarChart: function (element, dataSource, opts) {
      setElement(this, element, dataSource, opts, processBarData);
    },
    AreaChart: function (element, dataSource, opts) {
      setElement(this, element, dataSource, opts, processAreaData);
    },
    GeoChart: function (element, dataSource, opts) {
      setElement(this, element, dataSource, opts, processGeoData);
    },
    Timeline: function (element, dataSource, opts) {
      setElement(this, element, dataSource, opts, processTimelineData);
    },
    charts: {}
  };

  window.Chartkick = Chartkick;
}(window));
```


What does the above code do?
  - graph data
  - pluralize english words
  - converts colors into red, green, and blue (rgb) values

Which chart is not supported?
  - line
  - pie
  - bar
  - geo
  - area
  - timeline
  - chord

When processing a series, how are values sorted?
  - by value
  - by time

When processing a series, if the series is not an array, is the legend shown or hidden?
  - shown
  - hidden

What columns are added when rendering a timeline?

What is not in an ISO8601 date?
  - day
  - hour
  - milliseconds
  - minutes
  - month
  - seconds
  - year
  - day of week

Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data
























------------

```rb
/*
     _ _      _       _
 ___| (_) ___| | __  (_)___
/ __| | |/ __| |/ /  | / __|
\__ \ | | (__|   < _ | \__ \
|___/_|_|\___|_|\_(_)/ |___/
                   |__/

 Version: 1.3.14
  Author: Ken Wheeler
 Website: http://kenwheeler.github.io
    Docs: http://kenwheeler.github.io/slick
    Repo: http://github.com/kenwheeler/slick
  Issues: http://github.com/kenwheeler/slick/issues

 */

/* global window, document, define, jQuery, setInterval, clearInterval */

(function(factory) {
    'use strict';
    if (typeof define === 'function' && define.amd) {
        define(['jquery'], factory);
    } else if (typeof exports !== 'undefined') {
        module.exports = factory(require('jquery'));
    } else {
        factory(jQuery);
    }

}(function($) {
    'use strict';
    var Slick = window.Slick || {};

    Slick = (function() {

        var instanceUid = 0;

        function Slick(element, settings) {

            var _ = this,
                responsiveSettings, breakpoint;

            _.defaults = {
                accessibility: true,
                adaptiveHeight: false,
                appendArrows: $(element),
                appendDots: $(element),
                arrows: true,
                asNavFor: null,
                prevArrow: '<button type="button" data-role="none" class="slick-prev">Previous</button>',
                nextArrow: '<button type="button" data-role="none" class="slick-next">Next</button>',
                autoplay: false,
                autoplaySpeed: 3000,
                centerMode: false,
                centerPadding: '50px',
                cssEase: 'ease',
                customPaging: function(slider, i) {
                    return '<button type="button" data-role="none">' + (i + 1) + '</button>';
                },
                dots: false,
                dotsClass: 'slick-dots',
                draggable: true,
                easing: 'linear',
                fade: false,
                focusOnSelect: false,
                infinite: true,
                initialSlide: 0,
                lazyLoad: 'ondemand',
                onBeforeChange: null,
                onAfterChange: null,
                onInit: null,
                onReInit: null,
                onSetPosition: null,
                pauseOnHover: true,
                pauseOnDotsHover: false,
                respondTo: 'window',
                responsive: null,
                rtl: false,
                slide: 'div',
                slidesToShow: 1,
                slidesToScroll: 1,
                speed: 500,
                swipe: true,
                swipeToSlide: false,
                touchMove: true,
                touchThreshold: 5,
                useCSS: true,
                variableWidth: false,
                vertical: false,
                waitForAnimate: true
            };

            _.initials = {
                animating: false,
                dragging: false,
                autoPlayTimer: null,
                currentDirection: 0,
                currentLeft: null,
                currentSlide: 0,
                direction: 1,
                $dots: null,
                listWidth: null,
                listHeight: null,
                loadIndex: 0,
                $nextArrow: null,
                $prevArrow: null,
                slideCount: null,
                slideWidth: null,
                $slideTrack: null,
                $slides: null,
                sliding: false,
                slideOffset: 0,
                swipeLeft: null,
                $list: null,
                touchObject: {},
                transformsEnabled: false
            };

            $.extend(_, _.initials);

            _.activeBreakpoint = null;
            _.animType = null;
            _.animProp = null;
            _.breakpoints = [];
            _.breakpointSettings = [];
            _.cssTransitions = false;
            _.paused = false;
            _.positionProp = null;
            _.respondTo = null;
            _.shouldClick = true;
            _.$slider = $(element);
            _.$slidesCache = null;
            _.transformType = null;
            _.transitionType = null;
            _.windowWidth = 0;
            _.windowTimer = null;

            _.options = $.extend({}, _.defaults, settings);

            _.currentSlide = _.options.initialSlide;

            _.originalSettings = _.options;
            responsiveSettings = _.options.responsive || null;

            if (responsiveSettings && responsiveSettings.length > -1) {
                _.respondTo = _.options.respondTo || "window";
                for (breakpoint in responsiveSettings) {
                    if (responsiveSettings.hasOwnProperty(breakpoint)) {
                        _.breakpoints.push(responsiveSettings[
                            breakpoint].breakpoint);
                        _.breakpointSettings[responsiveSettings[
                            breakpoint].breakpoint] =
                            responsiveSettings[breakpoint].settings;
                    }
                }
                _.breakpoints.sort(function(a, b) {
                    return b - a;
                });
            }

            _.autoPlay = $.proxy(_.autoPlay, _);
            _.autoPlayClear = $.proxy(_.autoPlayClear, _);
            _.changeSlide = $.proxy(_.changeSlide, _);
            _.clickHandler = $.proxy(_.clickHandler, _);
            _.selectHandler = $.proxy(_.selectHandler, _);
            _.setPosition = $.proxy(_.setPosition, _);
            _.swipeHandler = $.proxy(_.swipeHandler, _);
            _.dragHandler = $.proxy(_.dragHandler, _);
            _.keyHandler = $.proxy(_.keyHandler, _);
            _.autoPlayIterator = $.proxy(_.autoPlayIterator, _);

            _.instanceUid = instanceUid++;

            // A simple way to check for HTML strings
            // Strict HTML recognition (must start with <)
            // Extracted from jQuery v1.11 source
            _.htmlExpr = /^(?:\s*(<[\w\W]+>)[^>]*)$/;

            _.init();

            _.checkResponsive();

        }

        return Slick;

    }());

    Slick.prototype.addSlide = function(markup, index, addBefore) {

        var _ = this;

        if (typeof(index) === 'boolean') {
            addBefore = index;
            index = null;
        } else if (index < 0 || (index >= _.slideCount)) {
            return false;
        }

        _.unload();

        if (typeof(index) === 'number') {
            if (index === 0 && _.$slides.length === 0) {
                $(markup).appendTo(_.$slideTrack);
            } else if (addBefore) {
                $(markup).insertBefore(_.$slides.eq(index));
            } else {
                $(markup).insertAfter(_.$slides.eq(index));
            }
        } else {
            if (addBefore === true) {
                $(markup).prependTo(_.$slideTrack);
            } else {
                $(markup).appendTo(_.$slideTrack);
            }
        }

        _.$slides = _.$slideTrack.children(this.options.slide);

        _.$slideTrack.children(this.options.slide).detach();

        _.$slideTrack.append(_.$slides);

        _.$slides.each(function(index, element) {
            $(element).attr("index",index);
        });

        _.$slidesCache = _.$slides;

        _.reinit();

    };

    Slick.prototype.animateSlide = function(targetLeft, callback) {

        var animProps = {}, _ = this;

        if(_.options.slidesToShow === 1 && _.options.adaptiveHeight === true && _.options.vertical === false) {
            var targetHeight = _.$slides.eq(_.currentSlide).outerHeight(true);
            _.$list.animate({height: targetHeight},_.options.speed);
        }

        if (_.options.rtl === true && _.options.vertical === false) {
            targetLeft = -targetLeft;
        }
        if (_.transformsEnabled === false) {
            if (_.options.vertical === false) {
                _.$slideTrack.animate({
                    left: targetLeft
                }, _.options.speed, _.options.easing, callback);
            } else {
                _.$slideTrack.animate({
                    top: targetLeft
                }, _.options.speed, _.options.easing, callback);
            }

        } else {

            if (_.cssTransitions === false) {

                $({
                    animStart: _.currentLeft
                }).animate({
                    animStart: targetLeft
                }, {
                    duration: _.options.speed,
                    easing: _.options.easing,
                    step: function(now) {
                        if (_.options.vertical === false) {
                            animProps[_.animType] = 'translate(' +
                                now + 'px, 0px)';
                            _.$slideTrack.css(animProps);
                        } else {
                            animProps[_.animType] = 'translate(0px,' +
                                now + 'px)';
                            _.$slideTrack.css(animProps);
                        }
                    },
                    complete: function() {
                        if (callback) {
                            callback.call();
                        }
                    }
                });

            } else {

                _.applyTransition();

                if (_.options.vertical === false) {
                    animProps[_.animType] = 'translate3d(' + targetLeft + 'px, 0px, 0px)';
                } else {
                    animProps[_.animType] = 'translate3d(0px,' + targetLeft + 'px, 0px)';
                }
                _.$slideTrack.css(animProps);

                if (callback) {
                    setTimeout(function() {

                        _.disableTransition();

                        callback.call();
                    }, _.options.speed);
                }

            }

        }

    };

    Slick.prototype.asNavFor = function(index) {
        var _ = this, asNavFor = _.options.asNavFor != null ? $(_.options.asNavFor).getSlick() : null;
        if(asNavFor != null) asNavFor.slideHandler(index, true);
    };

    Slick.prototype.applyTransition = function(slide) {

        var _ = this,
            transition = {};

        if (_.options.fade === false) {
            transition[_.transitionType] = _.transformType + ' ' + _.options.speed + 'ms ' + _.options.cssEase;
        } else {
            transition[_.transitionType] = 'opacity ' + _.options.speed + 'ms ' + _.options.cssEase;
        }

        if (_.options.fade === false) {
            _.$slideTrack.css(transition);
        } else {
            _.$slides.eq(slide).css(transition);
        }

    };

    Slick.prototype.autoPlay = function() {

        var _ = this;

        if (_.autoPlayTimer) {
            clearInterval(_.autoPlayTimer);
        }

        if (_.slideCount > _.options.slidesToShow && _.paused !== true) {
            _.autoPlayTimer = setInterval(_.autoPlayIterator,
                _.options.autoplaySpeed);
        }

    };

    Slick.prototype.autoPlayClear = function() {

        var _ = this;
        if (_.autoPlayTimer) {
            clearInterval(_.autoPlayTimer);
        }

    };

    Slick.prototype.autoPlayIterator = function() {

        var _ = this;

        if (_.options.infinite === false) {

            if (_.direction === 1) {

                if ((_.currentSlide + 1) === _.slideCount -
                    1) {
                    _.direction = 0;
                }

                _.slideHandler(_.currentSlide + _.options.slidesToScroll);

            } else {

                if ((_.currentSlide - 1 === 0)) {

                    _.direction = 1;

                }

                _.slideHandler(_.currentSlide - _.options.slidesToScroll);

            }

        } else {

            _.slideHandler(_.currentSlide + _.options.slidesToScroll);

        }

    };

    Slick.prototype.buildArrows = function() {

        var _ = this;

        if (_.options.arrows === true && _.slideCount > _.options.slidesToShow) {

            _.$prevArrow = $(_.options.prevArrow);
            _.$nextArrow = $(_.options.nextArrow);

            if (_.htmlExpr.test(_.options.prevArrow)) {
                _.$prevArrow.appendTo(_.options.appendArrows);
            }

            if (_.htmlExpr.test(_.options.nextArrow)) {
                _.$nextArrow.appendTo(_.options.appendArrows);
            }

            if (_.options.infinite !== true) {
                _.$prevArrow.addClass('slick-disabled');
            }

        }

    };

    Slick.prototype.buildDots = function() {

        var _ = this,
            i, dotString;

        if (_.options.dots === true && _.slideCount > _.options.slidesToShow) {

            dotString = '<ul class="' + _.options.dotsClass + '">';

            for (i = 0; i <= _.getDotCount(); i += 1) {
                dotString += '<li>' + _.options.customPaging.call(this, _, i) + '</li>';
            }

            dotString += '</ul>';

            _.$dots = $(dotString).appendTo(
                _.options.appendDots);

            _.$dots.find('li').first().addClass(
                'slick-active');

        }

    };

    Slick.prototype.buildOut = function() {

        var _ = this;

        _.$slides = _.$slider.children(_.options.slide +
            ':not(.slick-cloned)').addClass(
            'slick-slide');
        _.slideCount = _.$slides.length;

        _.$slides.each(function(index, element) {
            $(element).attr("index",index);
        });

        _.$slidesCache = _.$slides;

        _.$slider.addClass('slick-slider');

        _.$slideTrack = (_.slideCount === 0) ?
            $('<div class="slick-track"/>').appendTo(_.$slider) :
            _.$slides.wrapAll('<div class="slick-track"/>').parent();

        _.$list = _.$slideTrack.wrap(
            '<div class="slick-list"/>').parent();
        _.$slideTrack.css('opacity', 0);

        if (_.options.centerMode === true) {
            _.options.slidesToScroll = 1;
        }

        $('img[data-lazy]', _.$slider).not('[src]').addClass('slick-loading');

        _.setupInfinite();

        _.buildArrows();

        _.buildDots();

        _.updateDots();

        if (_.options.accessibility === true) {
            _.$list.prop('tabIndex', 0);
        }

        _.setSlideClasses(typeof this.currentSlide === 'number' ? this.currentSlide : 0);

        if (_.options.draggable === true) {
            _.$list.addClass('draggable');
        }

    };

    Slick.prototype.checkResponsive = function() {

        var _ = this,
            breakpoint, targetBreakpoint, respondToWidth;
        var sliderWidth = _.$slider.width();
        var windowWidth = window.innerWidth || $(window).width();
        if (_.respondTo === "window") {
          respondToWidth = windowWidth;
        } else if (_.respondTo === "slider") {
          respondToWidth = sliderWidth;
        } else if (_.respondTo === "min") {
          respondToWidth = Math.min(windowWidth, sliderWidth);
        }

        if (_.originalSettings.responsive && _.originalSettings
            .responsive.length > -1 && _.originalSettings.responsive !== null) {

            targetBreakpoint = null;

            for (breakpoint in _.breakpoints) {
                if (_.breakpoints.hasOwnProperty(breakpoint)) {
                    if (respondToWidth < _.breakpoints[breakpoint]) {
                        targetBreakpoint = _.breakpoints[breakpoint];
                    }
                }
            }

            if (targetBreakpoint !== null) {
                if (_.activeBreakpoint !== null) {
                    if (targetBreakpoint !== _.activeBreakpoint) {
                        _.activeBreakpoint =
                            targetBreakpoint;
                        _.options = $.extend({}, _.originalSettings,
                            _.breakpointSettings[
                                targetBreakpoint]);
                        _.refresh();
                    }
                } else {
                    _.activeBreakpoint = targetBreakpoint;
                    _.options = $.extend({}, _.originalSettings,
                        _.breakpointSettings[
                            targetBreakpoint]);
                    _.refresh();
                }
            } else {
                if (_.activeBreakpoint !== null) {
                    _.activeBreakpoint = null;
                    _.options = _.originalSettings;
                    _.refresh();
                }
            }

        }

    };

    Slick.prototype.changeSlide = function(event, dontAnimate) {

        var _ = this,
            $target = $(event.target),
            indexOffset, slideOffset, unevenOffset,navigables, prevNavigable;

        // If target is a link, prevent default action.
        $target.is('a') && event.preventDefault();

        unevenOffset = (_.slideCount % _.options.slidesToScroll !== 0);
        indexOffset = unevenOffset ? 0 : (_.slideCount - _.currentSlide) % _.options.slidesToScroll;

        switch (event.data.message) {

            case 'previous':
                slideOffset = indexOffset === 0 ? _.options.slidesToScroll : _.options.slidesToShow - indexOffset;
                if (_.slideCount > _.options.slidesToShow) {
                    _.slideHandler(_.currentSlide  - slideOffset, false, dontAnimate);
                }
                break;

            case 'next':
                slideOffset = indexOffset === 0 ? _.options.slidesToScroll : indexOffset;
                if (_.slideCount > _.options.slidesToShow) {
                    _.slideHandler(_.currentSlide + slideOffset, false, dontAnimate);
                }
                break;

            case 'index':
                var index = event.data.index === 0 ? 0 :
                    event.data.index || $(event.target).parent().index() * _.options.slidesToScroll;

                navigables = _.getNavigableIndexes();
                prevNavigable = 0;
                if(navigables.indexOf(index) === -1) {
                    if(index > navigables[navigables.length -1]){
                        index = navigables[navigables.length -1];
                    } else {
                        for(var n in navigables) {
                            if(index < navigables[n]) {
                                index = prevNavigable;
                                break;
                            }
                            prevNavigable = navigables[n];
                        }
                    }
                }
                _.slideHandler(index, false, dontAnimate);

            default:
                return;
        }

    };

    Slick.prototype.clickHandler = function(event) {

        var _ = this;

        if(_.shouldClick === false) {
            event.stopImmediatePropagation();
            event.stopPropagation();
            event.preventDefault();
        }

    }

    Slick.prototype.destroy = function() {

        var _ = this;

        _.autoPlayClear();

        _.touchObject = {};

        $('.slick-cloned', _.$slider).remove();
        if (_.$dots) {
            _.$dots.remove();
        }
        if (_.$prevArrow && (typeof _.options.prevArrow !== 'object')) {
            _.$prevArrow.remove();
        }
        if (_.$nextArrow && (typeof _.options.nextArrow !== 'object')) {
            _.$nextArrow.remove();
        }
        if (_.$slides.parent().hasClass('slick-track')) {
            _.$slides.unwrap().unwrap();
        }

        _.$slides.removeClass(
            'slick-slide slick-active slick-center slick-visible')
            .removeAttr('index')
            .css({
                position: '',
                left: '',
                top: '',
                zIndex: '',
                opacity: '',
                width: ''
            });

        _.$slider.removeClass('slick-slider');
        _.$slider.removeClass('slick-initialized');

        _.$list.off('.slick');
        $(window).off('.slick-' + _.instanceUid);
        $(document).off('.slick-' + _.instanceUid);

    };

    Slick.prototype.disableTransition = function(slide) {

        var _ = this,
            transition = {};

        transition[_.transitionType] = "";

        if (_.options.fade === false) {
            _.$slideTrack.css(transition);
        } else {
            _.$slides.eq(slide).css(transition);
        }

    };

    Slick.prototype.fadeSlide = function(oldSlide, slideIndex, callback) {

        var _ = this;

        if (_.cssTransitions === false) {

            _.$slides.eq(slideIndex).css({
                zIndex: 1000
            });

            _.$slides.eq(slideIndex).animate({
                opacity: 1
            }, _.options.speed, _.options.easing, callback);

            _.$slides.eq(oldSlide).animate({
                opacity: 0
            }, _.options.speed, _.options.easing);

        } else {

            _.applyTransition(slideIndex);
            _.applyTransition(oldSlide);

            _.$slides.eq(slideIndex).css({
                opacity: 1,
                zIndex: 1000
            });

            _.$slides.eq(oldSlide).css({
                opacity: 0
            });

            if (callback) {
                setTimeout(function() {

                    _.disableTransition(slideIndex);
                    _.disableTransition(oldSlide);

                    callback.call();
                }, _.options.speed);
            }

        }

    };

    Slick.prototype.filterSlides = function(filter) {

        var _ = this;

        if (filter !== null) {

            _.unload();

            _.$slideTrack.children(this.options.slide).detach();

            _.$slidesCache.filter(filter).appendTo(_.$slideTrack);

            _.reinit();

        }

    };

    Slick.prototype.getCurrent = function() {

        var _ = this;

        return _.currentSlide;

    };

    Slick.prototype.getDotCount = function() {

        var _ = this;

        var breakPoint = 0;
        var counter = 0;
        var pagerQty = 0;

        if(_.options.infinite === true) {
            pagerQty = Math.ceil(_.slideCount / _.options.slidesToScroll);
        } else {
            while (breakPoint < _.slideCount){
                ++pagerQty;
                breakPoint = counter + _.options.slidesToShow;
                counter += _.options.slidesToScroll <= _.options.slidesToShow ? _.options.slidesToScroll  : _.options.slidesToShow;
            }
        }

        return pagerQty - 1;

    };

    Slick.prototype.getLeft = function(slideIndex) {

        var _ = this,
            targetLeft,
            verticalHeight,
            verticalOffset = 0,
            slideWidth,
            targetSlide;

        _.slideOffset = 0;
        verticalHeight = _.$slides.first().outerHeight();

        if (_.options.infinite === true) {
            if (_.slideCount > _.options.slidesToShow) {
                _.slideOffset = (_.slideWidth * _.options.slidesToShow) * -1;
                verticalOffset = (verticalHeight * _.options.slidesToShow) * -1;
            }
            if (_.slideCount % _.options.slidesToScroll !== 0) {
                if (slideIndex + _.options.slidesToScroll > _.slideCount && _.slideCount > _.options.slidesToShow) {
                    if(slideIndex > _.slideCount) {
                        _.slideOffset = ((_.options.slidesToShow - (slideIndex - _.slideCount)) * _.slideWidth) * -1;
                        verticalOffset = ((_.options.slidesToShow - (slideIndex - _.slideCount)) * verticalHeight) * -1;
                    } else {
                        _.slideOffset = ((_.slideCount % _.options.slidesToScroll) * _.slideWidth) * -1;
                        verticalOffset = ((_.slideCount % _.options.slidesToScroll) * verticalHeight) * -1;
                    }
                }
            }
        } else {
            if(slideIndex + _.options.slidesToShow > _.slideCount) {
                _.slideOffset = ((slideIndex + _.options.slidesToShow) - _.slideCount) * _.slideWidth;
                verticalOffset = ((slideIndex + _.options.slidesToShow) - _.slideCount) * verticalHeight;
            }
        }

        if (_.slideCount <= _.options.slidesToShow){
            _.slideOffset = 0;
            verticalOffset = 0;
        }

        if (_.options.centerMode === true && _.options.infinite === true) {
            _.slideOffset += _.slideWidth * Math.floor(_.options.slidesToShow / 2) - _.slideWidth;
        } else if (_.options.centerMode === true) {
            _.slideOffset = 0;
            _.slideOffset += _.slideWidth * Math.floor(_.options.slidesToShow / 2);
        }

        if (_.options.vertical === false) {
            targetLeft = ((slideIndex * _.slideWidth) * -1) + _.slideOffset;
        } else {
            targetLeft = ((slideIndex * verticalHeight) * -1) + verticalOffset;
        }

        if (_.options.variableWidth === true) {

            if(_.slideCount <= _.options.slidesToShow || _.options.infinite === false) {
                targetSlide = _.$slideTrack.children('.slick-slide').eq(slideIndex);
            } else {
                targetSlide = _.$slideTrack.children('.slick-slide').eq(slideIndex + _.options.slidesToShow);
            }
            targetLeft = targetSlide[0] ? targetSlide[0].offsetLeft * -1 : 0;
            if (_.options.centerMode === true) {
                if(_.options.infinite === false) {
                    targetSlide = _.$slideTrack.children('.slick-slide').eq(slideIndex);
                } else {
                    targetSlide = _.$slideTrack.children('.slick-slide').eq(slideIndex + _.options.slidesToShow + 1);
                }
                targetLeft = targetSlide[0] ? targetSlide[0].offsetLeft * -1 : 0;
                targetLeft += (_.$list.width() - targetSlide.outerWidth()) / 2;
            }
        }

         // 1680

        return targetLeft;

    };

    Slick.prototype.getNavigableIndexes = function() {

        var _ = this;

        var breakPoint = 0;
        var counter = 0;
        var indexes = [];

        while (breakPoint < _.slideCount){
            indexes.push(breakPoint);
            breakPoint = counter + _.options.slidesToScroll;
            counter += _.options.slidesToScroll <= _.options.slidesToShow ? _.options.slidesToScroll  : _.options.slidesToShow;
        }

        return indexes;

    };

    Slick.prototype.getSlideCount = function() {

        var _ = this, slidesTraversed;

        if(_.options.swipeToSlide === true) {
            var swipedSlide = null;
            _.$slideTrack.find('.slick-slide').each(function(index, slide){
                if (slide.offsetLeft + ($(slide).outerWidth() / 2) > (_.swipeLeft * -1)) {
                    swipedSlide = slide;
                    return false;
                }
            });
            slidesTraversed = Math.abs($(swipedSlide).attr('index') - _.currentSlide);
            return slidesTraversed;
        } else {
            return _.options.slidesToScroll;
        }

    };

    Slick.prototype.init = function() {

        var _ = this;

        if (!$(_.$slider).hasClass('slick-initialized')) {

            $(_.$slider).addClass('slick-initialized');
            _.buildOut();
            _.setProps();
            _.startLoad();
            _.loadSlider();
            _.initializeEvents();
            _.updateArrows();
            _.updateDots();
        }

        if (_.options.onInit !== null) {
            _.options.onInit.call(this, _);
        }

    };

    Slick.prototype.initArrowEvents = function() {

        var _ = this;

        if (_.options.arrows === true && _.slideCount > _.options.slidesToShow) {
            _.$prevArrow.on('click.slick', {
                message: 'previous'
            }, _.changeSlide);
            _.$nextArrow.on('click.slick', {
                message: 'next'
            }, _.changeSlide);
        }

    };

    Slick.prototype.initDotEvents = function() {

        var _ = this;

        if (_.options.dots === true && _.slideCount > _.options.slidesToShow) {
            $('li', _.$dots).on('click.slick', {
                message: 'index'
            }, _.changeSlide);
        }

        if (_.options.dots === true && _.options.pauseOnDotsHover === true && _.options.autoplay === true) {
            $('li', _.$dots)
                .on('mouseenter.slick', function(){
                    _.paused = true;
                    _.autoPlayClear();
                })
                .on('mouseleave.slick', function(){
                    _.paused = false;
                    _.autoPlay();
                });
        }

    };

    Slick.prototype.initializeEvents = function() {

        var _ = this;

        _.initArrowEvents();

        _.initDotEvents();

        _.$list.on('touchstart.slick mousedown.slick', {
            action: 'start'
        }, _.swipeHandler);
        _.$list.on('touchmove.slick mousemove.slick', {
            action: 'move'
        }, _.swipeHandler);
        _.$list.on('touchend.slick mouseup.slick', {
            action: 'end'
        }, _.swipeHandler);
        _.$list.on('touchcancel.slick mouseleave.slick', {
            action: 'end'
        }, _.swipeHandler);

        _.$list.on('click.slick', _.clickHandler);

        if (_.options.pauseOnHover === true && _.options.autoplay === true) {
            _.$list.on('mouseenter.slick', function(){
                _.paused = true;
                _.autoPlayClear();
            });
            _.$list.on('mouseleave.slick', function(){
                _.paused = false;
                _.autoPlay();
            });
        }

        if(_.options.accessibility === true) {
            _.$list.on('keydown.slick', _.keyHandler);
        }

        if(_.options.focusOnSelect === true) {
            $(_.options.slide, _.$slideTrack).on('click.slick', _.selectHandler);
        }

        $(window).on('orientationchange.slick.slick-' + _.instanceUid, function() {
            _.checkResponsive();
            _.setPosition();
        });

        $(window).on('resize.slick.slick-' + _.instanceUid, function() {
            if ($(window).width() !== _.windowWidth) {
                clearTimeout(_.windowDelay);
                _.windowDelay = window.setTimeout(function() {
                    _.windowWidth = $(window).width();
                    _.checkResponsive();
                    _.setPosition();
                }, 50);
            }
        });

        $('*[draggable!=true]', _.$slideTrack).on('dragstart', function(e){ e.preventDefault(); })

        $(window).on('load.slick.slick-' + _.instanceUid, _.setPosition);
        $(document).on('ready.slick.slick-' + _.instanceUid, _.setPosition);

    };

    Slick.prototype.initUI = function() {

        var _ = this;

        if (_.options.arrows === true && _.slideCount > _.options.slidesToShow) {

            _.$prevArrow.show();
            _.$nextArrow.show();

        }

        if (_.options.dots === true && _.slideCount > _.options.slidesToShow) {

            _.$dots.show();

        }

        if (_.options.autoplay === true) {

            _.autoPlay();

        }

    };

    Slick.prototype.keyHandler = function(event) {

        var _ = this;

        if (event.keyCode === 37 && _.options.accessibility === true) {
            _.changeSlide({
                data: {
                    message: 'previous'
                }
            });
        } else if (event.keyCode === 39 && _.options.accessibility === true) {
            _.changeSlide({
                data: {
                    message: 'next'
                }
            });
        }

    };

    Slick.prototype.lazyLoad = function() {

        var _ = this,
            loadRange, cloneRange, rangeStart, rangeEnd;

        function loadImages(imagesScope) {
            $('img[data-lazy]', imagesScope).each(function() {
                var image = $(this),
                    imageSource = $(this).attr('data-lazy');

                image
                  .load(function() { image.animate({ opacity: 1 }, 200); })
                  .css({ opacity: 0 })
                  .attr('src', imageSource)
                  .removeAttr('data-lazy')
                  .removeClass('slick-loading');
            });
        }

        if (_.options.centerMode === true) {
            if (_.options.infinite === true) {
                rangeStart = _.currentSlide + (_.options.slidesToShow/2 + 1);
                rangeEnd = rangeStart + _.options.slidesToShow + 2;
            } else {
                rangeStart = Math.max(0, _.currentSlide - (_.options.slidesToShow/2 + 1));
                rangeEnd = 2 + (_.options.slidesToShow/2 + 1) + _.currentSlide;
            }
        } else {
            rangeStart = _.options.infinite ? _.options.slidesToShow + _.currentSlide : _.currentSlide;
            rangeEnd = rangeStart + _.options.slidesToShow;
            if (_.options.fade === true ) {
                if(rangeStart > 0) rangeStart--;
                if(rangeEnd <= _.slideCount) rangeEnd++;
            }
        }

        loadRange = _.$slider.find('.slick-slide').slice(rangeStart, rangeEnd);
        loadImages(loadRange);

          if (_.slideCount <= _.options.slidesToShow){
              cloneRange = _.$slider.find('.slick-slide')
              loadImages(cloneRange)
          }else
        if (_.currentSlide >= _.slideCount - _.options.slidesToShow) {
            cloneRange = _.$slider.find('.slick-cloned').slice(0, _.options.slidesToShow);
            loadImages(cloneRange)
        } else if (_.currentSlide === 0) {
            cloneRange = _.$slider.find('.slick-cloned').slice(_.options.slidesToShow * -1);
            loadImages(cloneRange);
        }

    };

    Slick.prototype.loadSlider = function() {

        var _ = this;

        _.setPosition();

        _.$slideTrack.css({
            opacity: 1
        });

        _.$slider.removeClass('slick-loading');

        _.initUI();

        if (_.options.lazyLoad === 'progressive') {
            _.progressiveLazyLoad();
        }

    };

    Slick.prototype.postSlide = function(index) {

        var _ = this;

        if (_.options.onAfterChange !== null) {
            _.options.onAfterChange.call(this, _, index);
        }

        _.animating = false;

        _.setPosition();

        _.swipeLeft = null;

        if (_.options.autoplay === true && _.paused === false) {
            _.autoPlay();
        }

    };

    Slick.prototype.progressiveLazyLoad = function() {

        var _ = this,
            imgCount, targetImage;

        imgCount = $('img[data-lazy]', _.$slider).length;

        if (imgCount > 0) {
            targetImage = $('img[data-lazy]', _.$slider).first();
            targetImage.attr('src', targetImage.attr('data-lazy')).removeClass('slick-loading').load(function() {
                targetImage.removeAttr('data-lazy');
                _.progressiveLazyLoad();
            })
         .error(function () {
          targetImage.removeAttr('data-lazy');
          _.progressiveLazyLoad();
         });
        }

    };

    Slick.prototype.refresh = function() {

        var _ = this,
            currentSlide = _.currentSlide;

        _.destroy();

        $.extend(_, _.initials);

        _.init();

        _.changeSlide({
            data: {
                message: 'index',
                index: currentSlide,
            }
        }, true);

    };

    Slick.prototype.reinit = function() {

        var _ = this;

        _.$slides = _.$slideTrack.children(_.options.slide).addClass(
            'slick-slide');

        _.slideCount = _.$slides.length;

        if (_.currentSlide >= _.slideCount && _.currentSlide !== 0) {
            _.currentSlide = _.currentSlide - _.options.slidesToScroll;
        }

        if (_.slideCount <= _.options.slidesToShow) {
            _.currentSlide = 0;
        }

        _.setProps();

        _.setupInfinite();

        _.buildArrows();

        _.updateArrows();

        _.initArrowEvents();

        _.buildDots();

        _.updateDots();

        _.initDotEvents();

        if(_.options.focusOnSelect === true) {
            $(_.options.slide, _.$slideTrack).on('click.slick', _.selectHandler);
        }

        _.setSlideClasses(0);

        _.setPosition();

        if (_.options.onReInit !== null) {
            _.options.onReInit.call(this, _);
        }

    };

    Slick.prototype.removeSlide = function(index, removeBefore, removeAll) {

        var _ = this;

        if (typeof(index) === 'boolean') {
            removeBefore = index;
            index = removeBefore === true ? 0 : _.slideCount - 1;
        } else {
            index = removeBefore === true ? --index : index;
        }

        if (_.slideCount < 1 || index < 0 || index > _.slideCount - 1) {
            return false;
        }

        _.unload();

        if(removeAll === true) {
            _.$slideTrack.children().remove();
        } else {
            _.$slideTrack.children(this.options.slide).eq(index).remove();
        }

        _.$slides = _.$slideTrack.children(this.options.slide);

        _.$slideTrack.children(this.options.slide).detach();

        _.$slideTrack.append(_.$slides);

        _.$slidesCache = _.$slides;

        _.reinit();

    };

    Slick.prototype.setCSS = function(position) {

        var _ = this,
            positionProps = {}, x, y;

        if (_.options.rtl === true) {
            position = -position;
        }
        x = _.positionProp == 'left' ? position + 'px' : '0px';
        y = _.positionProp == 'top' ? position + 'px' : '0px';

        positionProps[_.positionProp] = position;

        if (_.transformsEnabled === false) {
            _.$slideTrack.css(positionProps);
        } else {
            positionProps = {};
            if (_.cssTransitions === false) {
                positionProps[_.animType] = 'translate(' + x + ', ' + y + ')';
                _.$slideTrack.css(positionProps);
            } else {
                positionProps[_.animType] = 'translate3d(' + x + ', ' + y + ', 0px)';
                _.$slideTrack.css(positionProps);
            }
        }

    };

    Slick.prototype.setDimensions = function() {

        var _ = this;

        if (_.options.vertical === false) {
            if (_.options.centerMode === true) {
                _.$list.css({
                    padding: ('0px ' + _.options.centerPadding)
                });
            }
        } else {
            _.$list.height(_.$slides.first().outerHeight(true) * _.options.slidesToShow);
            if (_.options.centerMode === true) {
                _.$list.css({
                    padding: (_.options.centerPadding + ' 0px')
                });
            }
        }

        _.listWidth = _.$list.width();
        _.listHeight = _.$list.height();


        if(_.options.vertical === false && _.options.variableWidth === false) {
            _.slideWidth = Math.ceil(_.listWidth / _.options.slidesToShow);
            _.$slideTrack.width(Math.ceil((_.slideWidth * _.$slideTrack.children('.slick-slide').length)));

        } else if (_.options.variableWidth === true) {
            var trackWidth = 0;
            _.slideWidth = Math.ceil(_.listWidth / _.options.slidesToShow);
            _.$slideTrack.children('.slick-slide').each(function(){
                trackWidth += Math.ceil($(this).outerWidth(true));
            });
            _.$slideTrack.width(Math.ceil(trackWidth) + 1);
        } else {
            _.slideWidth = Math.ceil(_.listWidth);
            _.$slideTrack.height(Math.ceil((_.$slides.first().outerHeight(true) * _.$slideTrack.children('.slick-slide').length)));
        }

        var offset = _.$slides.first().outerWidth(true) - _.$slides.first().width();
        if (_.options.variableWidth === false) _.$slideTrack.children('.slick-slide').width(_.slideWidth - offset);

    };

    Slick.prototype.setFade = function() {

        var _ = this,
            targetLeft;

        _.$slides.each(function(index, element) {
            targetLeft = (_.slideWidth * index) * -1;
            if (_.options.rtl === true) {
                $(element).css({
                    position: 'relative',
                    right: targetLeft,
                    top: 0,
                    zIndex: 800,
                    opacity: 0
                });
            } else {
                $(element).css({
                    position: 'relative',
                    left: targetLeft,
                    top: 0,
                    zIndex: 800,
                    opacity: 0
                });
            }
        });

        _.$slides.eq(_.currentSlide).css({
            zIndex: 900,
            opacity: 1
        });

    };

    Slick.prototype.setHeight = function() {

        var _ = this;

        if(_.options.slidesToShow === 1 && _.options.adaptiveHeight === true && _.options.vertical === false) {
            var targetHeight = _.$slides.eq(_.currentSlide).outerHeight(true);
            _.$list.css('height', targetHeight);
        }

    };

    Slick.prototype.setPosition = function() {

        var _ = this;

        _.setDimensions();

        _.setHeight();

        if (_.options.fade === false) {
            _.setCSS(_.getLeft(_.currentSlide));
        } else {
            _.setFade();
        }

        if (_.options.onSetPosition !== null) {
            _.options.onSetPosition.call(this, _);
        }

    };

    Slick.prototype.setProps = function() {

        var _ = this,
            bodyStyle = document.body.style;

        _.positionProp = _.options.vertical === true ? 'top' : 'left';

        if (_.positionProp === 'top') {
            _.$slider.addClass('slick-vertical');
        } else {
            _.$slider.removeClass('slick-vertical');
        }

        if (bodyStyle.WebkitTransition !== undefined ||
            bodyStyle.MozTransition !== undefined ||
            bodyStyle.msTransition !== undefined) {
            if(_.options.useCSS === true) {
                _.cssTransitions = true;
            }
        }

        if (bodyStyle.OTransform !== undefined) {
            _.animType = 'OTransform';
            _.transformType = "-o-transform";
            _.transitionType = 'OTransition';
            if (bodyStyle.perspectiveProperty === undefined && bodyStyle.webkitPerspective === undefined) _.animType = false;
        }
        if (bodyStyle.MozTransform !== undefined) {
            _.animType = 'MozTransform';
            _.transformType = "-moz-transform";
            _.transitionType = 'MozTransition';
            if (bodyStyle.perspectiveProperty === undefined && bodyStyle.MozPerspective === undefined) _.animType = false;
        }
        if (bodyStyle.webkitTransform !== undefined) {
            _.animType = 'webkitTransform';
            _.transformType = "-webkit-transform";
            _.transitionType = 'webkitTransition';
            if (bodyStyle.perspectiveProperty === undefined && bodyStyle.webkitPerspective === undefined) _.animType = false;
        }
        if (bodyStyle.msTransform !== undefined) {
            _.animType = 'msTransform';
            _.transformType = "-ms-transform";
            _.transitionType = 'msTransition';
            if (bodyStyle.msTransform === undefined) _.animType = false;
        }
        if (bodyStyle.transform !== undefined && _.animType !== false) {
            _.animType = 'transform';
            _.transformType = "transform";
            _.transitionType = 'transition';
        }
        _.transformsEnabled = (_.animType !== null && _.animType !== false);

    };


    Slick.prototype.setSlideClasses = function(index) {

        var _ = this,
            centerOffset, allSlides, indexOffset, remainder;

        _.$slider.find('.slick-slide').removeClass('slick-active').removeClass('slick-center');
        allSlides = _.$slider.find('.slick-slide');

        if (_.options.centerMode === true) {

            centerOffset = Math.floor(_.options.slidesToShow / 2);

            if(_.options.infinite === true) {

                if (index >= centerOffset && index <= (_.slideCount - 1) - centerOffset) {
                    _.$slides.slice(index - centerOffset, index + centerOffset + 1).addClass('slick-active');
                } else {
                    indexOffset = _.options.slidesToShow + index;
                    allSlides.slice(indexOffset - centerOffset + 1, indexOffset + centerOffset + 2).addClass('slick-active');
                }

                if (index === 0) {
                    allSlides.eq(allSlides.length - 1 - _.options.slidesToShow).addClass('slick-center');
                } else if (index === _.slideCount - 1) {
                    allSlides.eq(_.options.slidesToShow).addClass('slick-center');
                }

            }

            _.$slides.eq(index).addClass('slick-center');

        } else {

            if (index >= 0 && index <= (_.slideCount - _.options.slidesToShow)) {
                _.$slides.slice(index, index + _.options.slidesToShow).addClass('slick-active');
            } else if ( allSlides.length <= _.options.slidesToShow ) {
                allSlides.addClass('slick-active');
            } else {
                remainder = _.slideCount%_.options.slidesToShow;
                indexOffset = _.options.infinite === true ? _.options.slidesToShow + index : index;
                if(_.options.slidesToShow == _.options.slidesToScroll && (_.slideCount - index) < _.options.slidesToShow) {
                    allSlides.slice(indexOffset-(_.options.slidesToShow-remainder), indexOffset + remainder).addClass('slick-active');
                } else {
                    allSlides.slice(indexOffset, indexOffset + _.options.slidesToShow).addClass('slick-active');
                }
            }

        }

        if (_.options.lazyLoad === 'ondemand') {
            _.lazyLoad();
        }

    };

    Slick.prototype.setupInfinite = function() {

        var _ = this,
            i, slideIndex, infiniteCount;

        if (_.options.fade === true) {
            _.options.centerMode = false;
        }

        if (_.options.infinite === true && _.options.fade === false) {

            slideIndex = null;

            if (_.slideCount > _.options.slidesToShow) {

                if (_.options.centerMode === true) {
                    infiniteCount = _.options.slidesToShow + 1;
                } else {
                    infiniteCount = _.options.slidesToShow;
                }

                for (i = _.slideCount; i > (_.slideCount -
                    infiniteCount); i -= 1) {
                    slideIndex = i - 1;
                    $(_.$slides[slideIndex]).clone(true).attr('id', '')
                        .attr('index', slideIndex-_.slideCount)
                        .prependTo(_.$slideTrack).addClass('slick-cloned');
                }
                for (i = 0; i < infiniteCount; i += 1) {
                    slideIndex = i;
                    $(_.$slides[slideIndex]).clone(true).attr('id', '')
                        .attr('index', slideIndex+_.slideCount)
                        .appendTo(_.$slideTrack).addClass('slick-cloned');
                }
                _.$slideTrack.find('.slick-cloned').find('[id]').each(function() {
                    $(this).attr('id', '');
                });

            }

        }

    };

    Slick.prototype.selectHandler = function(event) {

        var _ = this;
        var index = parseInt($(event.target).parents('.slick-slide').attr("index"));
        if(!index) index = 0;

        if(_.slideCount <= _.options.slidesToShow){
            _.$slider.find('.slick-slide').removeClass('slick-active');
            _.$slides.eq(index).addClass('slick-active');
            if(_.options.centerMode === true) {
                _.$slider.find('.slick-slide').removeClass('slick-center');
                _.$slides.eq(index).addClass('slick-center');
            }
            _.asNavFor(index);
            return;
        }
        _.slideHandler(index);

    };

    Slick.prototype.slideHandler = function(index,sync,dontAnimate) {

        var targetSlide, animSlide, oldSlide, slideLeft, unevenOffset, targetLeft = null,
            _ = this;

        sync = sync || false;

        if (_.animating === true && _.options.waitForAnimate === true) {
            return;
        }

        if (_.options.fade === true && _.currentSlide === index) {
            return;
        }

        if (_.slideCount <= _.options.slidesToShow) {
            return;
        }

        if (sync === false) {
            _.asNavFor(index);
        }

        targetSlide = index;
        targetLeft = _.getLeft(targetSlide);
        slideLeft = _.getLeft(_.currentSlide);

        _.currentLeft = _.swipeLeft === null ? slideLeft : _.swipeLeft;

        if (_.options.infinite === false && _.options.centerMode === false && (index < 0 || index > _.getDotCount() * _.options.slidesToScroll)) {
            if(_.options.fade === false) {
                targetSlide = _.currentSlide;
                if(dontAnimate!==true) {
                    _.animateSlide(slideLeft, function() {
                        _.postSlide(targetSlide);
                    });
                } else {
                    _.postSlide(targetSlide);
                }
            }
            return;
        } else if (_.options.infinite === false && _.options.centerMode === true && (index < 0 || index > (_.slideCount - _.options.slidesToScroll))) {
            if(_.options.fade === false) {
                targetSlide = _.currentSlide;
                if(dontAnimate!==true) {
                    _.animateSlide(slideLeft, function() {
                        _.postSlide(targetSlide);
                    });
                } else {
                    _.postSlide(targetSlide);
                }
            }
            return;
        }

        if (_.options.autoplay === true) {
            clearInterval(_.autoPlayTimer);
        }

        if (targetSlide < 0) {
            if (_.slideCount % _.options.slidesToScroll !== 0) {
                animSlide = _.slideCount - (_.slideCount % _.options.slidesToScroll);
            } else {
                animSlide = _.slideCount + targetSlide;
            }
        } else if (targetSlide >= _.slideCount) {
            if (_.slideCount % _.options.slidesToScroll !== 0) {
                animSlide = 0;
            } else {
                animSlide = targetSlide - _.slideCount;
            }
        } else {
            animSlide = targetSlide;
        }

        _.animating = true;

        if (_.options.onBeforeChange !== null && index !== _.currentSlide) {
            _.options.onBeforeChange.call(this, _, _.currentSlide, animSlide);
        }

        oldSlide = _.currentSlide;
        _.currentSlide = animSlide;

        _.setSlideClasses(_.currentSlide);

        _.updateDots();
        _.updateArrows();

        if (_.options.fade === true) {
            if(dontAnimate!==true) {
                _.fadeSlide(oldSlide,animSlide, function() {
                    _.postSlide(animSlide);
                });
            } else {
                _.postSlide(animSlide);
            }
            return;
        }

        if(dontAnimate!==true) {
            _.animateSlide(targetLeft, function() {
                _.postSlide(animSlide);
            });
        } else {
            _.postSlide(animSlide);
        }

    };

    Slick.prototype.startLoad = function() {

        var _ = this;

        if (_.options.arrows === true && _.slideCount > _.options.slidesToShow) {

            _.$prevArrow.hide();
            _.$nextArrow.hide();

        }

        if (_.options.dots === true && _.slideCount > _.options.slidesToShow) {

            _.$dots.hide();

        }

        _.$slider.addClass('slick-loading');

    };

    Slick.prototype.swipeDirection = function() {

        var xDist, yDist, r, swipeAngle, _ = this;

        xDist = _.touchObject.startX - _.touchObject.curX;
        yDist = _.touchObject.startY - _.touchObject.curY;
        r = Math.atan2(yDist, xDist);

        swipeAngle = Math.round(r * 180 / Math.PI);
        if (swipeAngle < 0) {
            swipeAngle = 360 - Math.abs(swipeAngle);
        }

        if ((swipeAngle <= 45) && (swipeAngle >= 0)) {
            return (_.options.rtl === false ? 'left' : 'right');
        }
        if ((swipeAngle <= 360) && (swipeAngle >= 315)) {
            return (_.options.rtl === false ? 'left' : 'right');
        }
        if ((swipeAngle >= 135) && (swipeAngle <= 225)) {
            return (_.options.rtl === false ? 'right' : 'left');
        }

        return 'vertical';

    };

    Slick.prototype.swipeEnd = function(event) {

        var _ = this, slideCount;

        _.dragging = false;

        _.shouldClick = (_.touchObject.swipeLength > 10) ? false : true;

        if (_.touchObject.curX === undefined) {
            return false;
        }

        if (_.touchObject.swipeLength >= _.touchObject.minSwipe) {

            switch (_.swipeDirection()) {
                case 'left':
                    _.slideHandler(_.currentSlide + _.getSlideCount());
                    _.currentDirection = 0;
                    _.touchObject = {};
                    break;

                case 'right':
                    _.slideHandler(_.currentSlide - _.getSlideCount());
                    _.currentDirection = 1;
                    _.touchObject = {};
                    break;
            }
        } else {
            if(_.touchObject.startX !== _.touchObject.curX) {
                _.slideHandler(_.currentSlide);
                _.touchObject = {};
            }
        }

    };

    Slick.prototype.swipeHandler = function(event) {

        var _ = this;

        if ((_.options.swipe === false) || ('ontouchend' in document && _.options.swipe === false)) {
           return;
        } else if (_.options.draggable === false && event.type.indexOf('mouse') !== -1) {
           return;
        }

        _.touchObject.fingerCount = event.originalEvent && event.originalEvent.touches !== undefined ?
            event.originalEvent.touches.length : 1;

        _.touchObject.minSwipe = _.listWidth / _.options
            .touchThreshold;

        switch (event.data.action) {

            case 'start':
                _.swipeStart(event);
                break;

            case 'move':
                _.swipeMove(event);
                break;

            case 'end':
                _.swipeEnd(event);
                break;

        }

    };

    Slick.prototype.swipeMove = function(event) {

        var _ = this,
            curLeft, swipeDirection, positionOffset, touches;

        touches = event.originalEvent !== undefined ? event.originalEvent.touches : null;

        if (!_.dragging || touches && touches.length !== 1) {
            return false;
        }

        curLeft = _.getLeft(_.currentSlide);

        _.touchObject.curX = touches !== undefined ? touches[0].pageX : event.clientX;
        _.touchObject.curY = touches !== undefined ? touches[0].pageY : event.clientY;

        _.touchObject.swipeLength = Math.round(Math.sqrt(
            Math.pow(_.touchObject.curX - _.touchObject.startX, 2)));

        swipeDirection = _.swipeDirection();

        if (swipeDirection === 'vertical') {
            return;
        }

        if (event.originalEvent !== undefined && _.touchObject.swipeLength > 4) {
            event.preventDefault();
        }

        positionOffset = (_.options.rtl === false ? 1 : -1) * (_.touchObject.curX > _.touchObject.startX ? 1 : -1);

        if (_.options.vertical === false) {
            _.swipeLeft = curLeft + _.touchObject.swipeLength * positionOffset;
        } else {
            _.swipeLeft = curLeft + (_.touchObject
                .swipeLength * (_.$list.height() / _.listWidth)) * positionOffset;
        }

        if (_.options.fade === true || _.options.touchMove === false) {
            return false;
        }

        if (_.animating === true) {
            _.swipeLeft = null;
            return false;
        }

        _.setCSS(_.swipeLeft);

    };

    Slick.prototype.swipeStart = function(event) {

        var _ = this,
            touches;

        if (_.touchObject.fingerCount !== 1 || _.slideCount <= _.options.slidesToShow) {
            _.touchObject = {};
            return false;
        }

        if (event.originalEvent !== undefined && event.originalEvent.touches !== undefined) {
            touches = event.originalEvent.touches[0];
        }

        _.touchObject.startX = _.touchObject.curX = touches !== undefined ? touches.pageX : event.clientX;
        _.touchObject.startY = _.touchObject.curY = touches !== undefined ? touches.pageY : event.clientY;

        _.dragging = true;

    };

    Slick.prototype.unfilterSlides = function() {

        var _ = this;

        if (_.$slidesCache !== null) {

            _.unload();

            _.$slideTrack.children(this.options.slide).detach();

            _.$slidesCache.appendTo(_.$slideTrack);

            _.reinit();

        }

    };

    Slick.prototype.unload = function() {

        var _ = this;

        $('.slick-cloned', _.$slider).remove();
        if (_.$dots) {
            _.$dots.remove();
        }
        if (_.$prevArrow && (typeof _.options.prevArrow !== 'object')) {
            _.$prevArrow.remove();
        }
        if (_.$nextArrow && (typeof _.options.nextArrow !== 'object')) {
            _.$nextArrow.remove();
        }
        _.$slides.removeClass(
            'slick-slide slick-active slick-visible').css('width', '');

    };

    Slick.prototype.updateArrows = function() {

        var _ = this, centerOffset;

        centerOffset = Math.floor(_.options.slidesToShow / 2)

        if (_.options.arrows === true && _.options.infinite !==
            true && _.slideCount > _.options.slidesToShow) {
            _.$prevArrow.removeClass('slick-disabled');
            _.$nextArrow.removeClass('slick-disabled');
            if (_.currentSlide === 0) {
                _.$prevArrow.addClass('slick-disabled');
                _.$nextArrow.removeClass('slick-disabled');
            } else if (_.currentSlide >= _.slideCount - _.options.slidesToShow && _.options.centerMode === false) {
                _.$nextArrow.addClass('slick-disabled');
                _.$prevArrow.removeClass('slick-disabled');
            } else if (_.currentSlide > _.slideCount - _.options.slidesToShow + centerOffset  && _.options.centerMode === true) {
                _.$nextArrow.addClass('slick-disabled');
                _.$prevArrow.removeClass('slick-disabled');
            }
        }

    };

    Slick.prototype.updateDots = function() {

        var _ = this;

        if (_.$dots !== null) {

            _.$dots.find('li').removeClass('slick-active');
            _.$dots.find('li').eq(Math.floor(_.currentSlide / _.options.slidesToScroll)).addClass('slick-active');

        }

    };

    $.fn.slick = function(options) {
        var _ = this;
        return _.each(function(index, element) {

            element.slick = new Slick(element, options);

        });
    };

    $.fn.slickAdd = function(slide, slideIndex, addBefore) {
        var _ = this;
        return _.each(function(index, element) {

            element.slick.addSlide(slide, slideIndex, addBefore);

        });
    };

    $.fn.slickCurrentSlide = function() {
        var _ = this;
        return _.get(0).slick.getCurrent();
    };

    $.fn.slickFilter = function(filter) {
        var _ = this;
        return _.each(function(index, element) {

            element.slick.filterSlides(filter);

        });
    };

    $.fn.slickGoTo = function(slide, dontAnimate) {
        var _ = this;
        return _.each(function(index, element) {

            element.slick.changeSlide({
                data: {
                    message: 'index',
                    index: parseInt(slide)
                }
            }, dontAnimate);

        });
    };

    $.fn.slickNext = function() {
        var _ = this;
        return _.each(function(index, element) {

            element.slick.changeSlide({
                data: {
                    message: 'next'
                }
            });

        });
    };

    $.fn.slickPause = function() {
        var _ = this;
        return _.each(function(index, element) {

            element.slick.autoPlayClear();
            element.slick.paused = true;

        });
    };

    $.fn.slickPlay = function() {
        var _ = this;
        return _.each(function(index, element) {

            element.slick.paused = false;
            element.slick.autoPlay();

        });
    };

    $.fn.slickPrev = function() {
        var _ = this;
        return _.each(function(index, element) {

            element.slick.changeSlide({
                data: {
                    message: 'previous'
                }
            });

        });
    };

    $.fn.slickRemove = function(slideIndex, removeBefore) {
        var _ = this;
        return _.each(function(index, element) {

            element.slick.removeSlide(slideIndex, removeBefore);

        });
    };

    $.fn.slickRemoveAll = function() {
        var _ = this;
        return _.each(function(index, element) {

            element.slick.removeSlide(null, null, true);

        });
    };

    $.fn.slickGetOption = function(option) {
        var _ = this;
        return _.get(0).slick.options[option];
    };

    $.fn.slickSetOption = function(option, value, refresh) {
        var _ = this;
        return _.each(function(index, element) {

            element.slick.options[option] = value;

            if (refresh === true) {
                element.slick.unload();
                element.slick.reinit();
            }

        });
    };

    $.fn.slickUnfilter = function() {
        var _ = this;
        return _.each(function(index, element) {

            element.slick.unfilterSlides();

        });
    };

    $.fn.unslick = function() {
        var _ = this;
        return _.each(function(index, element) {

          if (element.slick) {
            element.slick.destroy();
          }

        });
    };

    $.fn.getSlick = function() {
        var s = null;
        var _ = this;
        _.each(function(index, element) {
            s = element.slick;
        });

        return s;
    };

}));
```



What are the three events that can change a slide?

Does this image carousel support mobile swiping?

When determining the swipe direction, does this carousel support vertical swiping?

Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data

















-----------
```js
/*
 * jQuery Notebook 0.5
 *
 * Copyright (c) 2014
 * Raphael Cruzeiro - http://raphaelcruzeiro.eu/
 * Otávio Soares
 *
 * Released under the MIT License
 * http://opensource.org/licenses/MIT
 *
 * Github https://github.com/raphaelcruzeiro/jquery-notebook
 * Version 0.5
 *
 * Some functions of this plugin were based on Jacob Kelley's Medium.js
 * https://github.com/jakiestfu/Medium.js/
 */

(function($, d, w) {

    /*
     * This module deals with the CSS transforms. As it is not possible to easily
     * combine the transform functions with JavaScript this module abstract those
     * functions and generates a raw transform matrix, combining the new transform
     * with the others that were previously applied to the element.
     */

    var transform = (function() {
        var matrixToArray = function(str) {
            if (!str || str == 'none') {
                return [1, 0, 0, 1, 0, 0];
            }
            return str.match(/(-?[0-9\.]+)/g);
        };

        var getPreviousTransforms = function(elem) {
            return elem.css('-webkit-transform') || elem.css('transform') || elem.css('-moz-transform') ||
                elem.css('-o-transform') || elem.css('-ms-transform');
        };

        var getMatrix = function(elem) {
            var previousTransform = getPreviousTransforms(elem);
            return matrixToArray(previousTransform);
        };

        var applyTransform = function(elem, transform) {
            elem.css('-webkit-transform', transform);
            elem.css('-moz-transform', transform);
            elem.css('-o-transform', transform);
            elem.css('-ms-transform', transform);
            elem.css('transform', transform);
        };

        var buildTransformString = function(matrix) {
            return 'matrix(' + matrix[0] +
                ', ' + matrix[1] +
                ', ' + matrix[2] +
                ', ' + matrix[3] +
                ', ' + matrix[4] +
                ', ' + matrix[5] + ')';
        };

        var getTranslate = function(elem) {
            var matrix = getMatrix(elem);
            return {
                x: parseInt(matrix[4]),
                y: parseInt(matrix[5])
            };
        };

        var scale = function(elem, _scale) {
            var matrix = getMatrix(elem);
            matrix[0] = matrix[3] = _scale;
            var transform = buildTransformString(matrix);
            applyTransform(elem, transform);
        };

        var translate = function(elem, x, y) {
            var matrix = getMatrix(elem);
            matrix[4] = x;
            matrix[5] = y;
            var transform = buildTransformString(matrix);
            applyTransform(elem, transform);
        };

        var rotate = function(elem, deg) {
            var matrix = getMatrix(elem);
            var rad1 = deg * (Math.PI / 180);
            var rad2 = rad1 * -1;
            matrix[1] = rad1;
            matrix[2] = rad2;
            var transform = buildTransformString(matrix);
            applyTransform(elem, transform);
        };

        return {
            scale: scale,
            translate: translate,
            rotate: rotate,
            getTranslate: getTranslate
        };
    })();

    var isMac = w.navigator.platform == 'MacIntel',
        mouseX = 0,
        mouseY = 0,
        cache = {
            command: false,
            shift: false,
            isSelecting: false
        },
        modifiers = {
            66: 'bold',
            73: 'italic',
            85: 'underline',
            112: 'h1',
            113: 'h2',
            122: 'undo'
        },
        options,
        utils = {
            keyboard: {
                isCommand: function(e, callbackTrue, callbackFalse) {
                    if (isMac && e.metaKey || !isMac && e.ctrlKey) {
                        callbackTrue();
                    } else {
                        callbackFalse();
                    }
                },
                isShift: function(e, callbackTrue, callbackFalse) {
                    if (e.shiftKey) {
                        callbackTrue();
                    } else {
                        callbackFalse();
                    }
                },
                isModifier: function(e, callback) {
                    var key = e.which,
                        cmd = modifiers[key];
                    if (cmd) {
                        callback.call(this, cmd);
                    }
                },
                isEnter: function(e, callback) {
                    if (e.which === 13) {
                        callback();
                    }
                },
                isArrow: function(e, callback) {
                    if (e.which >= 37 || e.which <= 40) {
                        callback();
                    }
                }
            },
            html: {
                addTag: function(elem, tag, focus, editable) {
                    var newElement = $(d.createElement(tag));
                    newElement.attr('contenteditable', Boolean(editable));
                    newElement.append(' ');
                    elem.append(newElement);
                    if (focus) {
                        cache.focusedElement = elem.children().last();
                        utils.cursor.set(elem, 0, cache.focusedElement);
                    }
                    return newElement;
                }
            },
            cursor: {
                set: function(editor, pos, elem) {
                    var range;
                    if (d.createRange) {
                        range = d.createRange();
                        var selection = w.getSelection(),
                            lastChild = editor.children().last(),
                            length = lastChild.html().length - 1,
                            toModify = elem ? elem[0] : lastChild[0],
                            theLength = typeof pos !== 'undefined' ? pos : length;
                        range.setStart(toModify, theLength);
                        range.collapse(true);
                        selection.removeAllRanges();
                        selection.addRange(range);
                    } else {
                        range = d.body.createTextRange();
                        range.moveToElementText(elem);
                        range.collapse(false);
                        range.select();
                    }
                }
            },
            selection: {
                save: function() {
                    if (w.getSelection) {
                        var sel = w.getSelection();
                        if (sel.rangeCount > 0) {
                            return sel.getRangeAt(0);
                        }
                    } else if (d.selection && d.selection.createRange) { // IE
                        return d.selection.createRange();
                    }
                    return null;
                },
                restore: function(range) {
                    if (range) {
                        if (w.getSelection) {
                            var sel = w.getSelection();
                            sel.removeAllRanges();
                            sel.addRange(range);
                        } else if (d.selection && range.select) { // IE
                            range.select();
                        }
                    }
                },
                getText: function() {
                    var txt = '';
                    if (w.getSelection) {
                        txt = w.getSelection().toString();
                    } else if (d.getSelection) {
                        txt = d.getSelection().toString();
                    } else if (d.selection) {
                        txt = d.selection.createRange().text;
                    }
                    return txt;
                },
                clear: function() {
                    if (window.getSelection) {
                        if (window.getSelection().empty) { // Chrome
                            window.getSelection().empty();
                        } else if (window.getSelection().removeAllRanges) { // Firefox
                            window.getSelection().removeAllRanges();
                        }
                    } else if (document.selection) { // IE?
                        document.selection.empty();
                    }
                },
                getContainer: function(sel) {
                    if (w.getSelection && sel && sel.commonAncestorContainer) {
                        return sel.commonAncestorContainer;
                    } else if (d.selection && sel && sel.parentElement) {
                        return sel.parentElement();
                    }
                    return null;
                },
                getSelection: function() {
                    if (w.getSelection) {
                        return w.getSelection();
                    } else if (d.selection && d.selection.createRange) { // IE
                        return d.selection;
                    }
                    return null;
                }
            },
            validation: {
                isUrl: function(url) {
                    return (/^(https?:\/\/)?([\da-z\.-]+)\.([a-z\.]{2,6})([\/\w \.-]*)*\/?$/).test(url);
                }
            }
        },
        bubble = {
            /*
             * This is called to position the bubble above the selection.
             */
            updatePos: function(editor, elem) {
                var sel = w.getSelection(),
                    range = sel.getRangeAt(0),
                    boundary = range.getBoundingClientRect(),
                    bubbleWidth = elem.width(),
                    bubbleHeight = elem.height(),
                    offset = editor.offset().left,
                    pos = {
                        x: (boundary.left + boundary.width / 2) - (bubbleWidth / 2),
                        y: boundary.top - bubbleHeight - 8 + $(document).scrollTop()
                    };
                transform.translate(elem, pos.x, pos.y);
            },
            /*
             * Updates the bubble to set the active formats for the current selection.
             */
            updateState: function(editor, elem) {
                elem.find('button').removeClass('active');
                var sel = w.getSelection(),
                    formats = [];
                bubble.checkForFormatting(sel.focusNode, formats);
                var formatDict = {
                    'b': 'bold',
                    'i': 'italic',
                    'u': 'underline',
                    'h1': 'h1',
                    'h2': 'h2',
                    'a': 'anchor',
                    'ul': 'ul',
                    'ol': 'ol'
                };
                for (var i = 0; i < formats.length; i++) {
                    var format = formats[i];
                    elem.find('button.' + formatDict[format]).addClass('active');
                }
            },
            /*
             * Recursively navigates upwards in the DOM to find all the format
             * tags enclosing the selection.
             */
            checkForFormatting: function(currentNode, formats) {
                var validFormats = ['b', 'i', 'u', 'h1', 'h2', 'ol', 'ul', 'li', 'a'];
                if (currentNode.nodeName === '#text' ||
                    validFormats.indexOf(currentNode.nodeName.toLowerCase()) != -1) {
                    if (currentNode.nodeName != '#text') {
                        formats.push(currentNode.nodeName.toLowerCase());
                    }
                    bubble.checkForFormatting(currentNode.parentNode, formats);
                }
            },
            buildMenu: function(editor, elem) {
                var ul = utils.html.addTag(elem, 'ul', false, false);
                for (var cmd in options.modifiers) {
                    var li = utils.html.addTag(ul, 'li', false, false);
                    var btn = utils.html.addTag(li, 'button', false, false);
                    btn.attr('editor-command', options.modifiers[cmd]);
                    btn.addClass(options.modifiers[cmd]);
                }
                elem.find('button').click(function(e) {
                    e.preventDefault();
                    var cmd = $(this).attr('editor-command');
                    events.commands[cmd].call(editor, e);
                });
                var linkArea = utils.html.addTag(elem, 'div', false, false);
                linkArea.addClass('link-area');
                var linkInput = utils.html.addTag(linkArea, 'input', false, false);
                linkInput.attr({
                    type: 'text'
                });
                var closeBtn = utils.html.addTag(linkArea, 'button', false, false);
                closeBtn.click(function(e) {
                    e.preventDefault();
                    var editor = $(this).closest('.editor');
                    $(this).closest('.link-area').hide();
                    $(this).closest('.bubble').find('ul').show();
                });
            },
            show: function() {
                var tag = $(this).parent().find('.bubble');
                if (!tag.length) {
                    tag = utils.html.addTag($(this).parent(), 'div', false, false);
                    tag.addClass('jquery-notebook bubble');
                }
                tag.empty();
                bubble.buildMenu(this, tag);
                tag.show();
                bubble.updateState(this, tag);
                if (!tag.hasClass('active')) {
                    tag.addClass('jump');
                } else {
                    tag.removeClass('jump');
                }
                bubble.updatePos($(this), tag);
                tag.addClass('active');
            },
            update: function() {
                var tag = $(this).parent().find('.bubble');
                bubble.updateState(this, tag);
            },
            clear: function() {
                var elem = $(this).parent().find('.bubble');
                if (!elem.hasClass('active')) return;
                elem.removeClass('active');
                bubble.hideLinkInput.call(this);
                bubble.showButtons.call(this);
                setTimeout(function() {
                    if (elem.hasClass('active')) return;
                    elem.hide();
                }, 500);
            },
            hideButtons: function() {
                $(this).parent().find('.bubble').find('ul').hide();
            },
            showButtons: function() {
                $(this).parent().find('.bubble').find('ul').show();
            },
            showLinkInput: function(selection) {
                bubble.hideButtons.call(this);
                var editor = this;
                var elem = $(this).parent().find('.bubble').find('input[type=text]');
                var hasLink = elem.closest('.jquery-notebook').find('button.anchor').hasClass('active');
                elem.unbind('keydown');
                elem.keydown(function(e) {
                    var elem = $(this);
                    utils.keyboard.isEnter(e, function() {
                        e.preventDefault();
                        var url = elem.val();
                        if (utils.validation.isUrl(url)) {
                            e.url = url;
                            events.commands.createLink(e, selection);
                            bubble.clear.call(editor);
                        } else if (url === '' && hasLink) {
                            events.commands.removeLink(e, selection);
                            bubble.clear.call(editor);
                        }
                    });
                });
                elem.bind('paste', function(e) {
                    var elem = $(this);
                    setTimeout(function() {
                        var text = elem.val();
                        if (/http:\/\/https?:\/\//.test(text)) {
                            text = text.substring(7);
                            elem.val(text);
                        }
                    }, 1);
                });
                var linkText = 'http://';
                if (hasLink) {
                    var anchor = $(utils.selection.getContainer(selection)).closest('a');
                    linkText = anchor.prop('href') || linkText;
                }
                $(this).parent().find('.link-area').show();
                elem.val(linkText).focus();
            },
            hideLinkInput: function() {
                $(this).parent().find('.bubble').find('.link-area').hide();
            }
        },
        actions = {
            bindEvents: function(elem) {
                elem.keydown(rawEvents.keydown);
                elem.keyup(rawEvents.keyup);
                elem.focus(rawEvents.focus);
                elem.bind('paste', events.paste);
                elem.mousedown(rawEvents.mouseClick);
                elem.mouseup(rawEvents.mouseUp);
                elem.mousemove(rawEvents.mouseMove);
                elem.blur(rawEvents.blur);
                $('body').mouseup(function(e) {
                    if (e.target == e.currentTarget && cache.isSelecting) {
                        rawEvents.mouseUp.call(elem, e);
                    }
                });
            },
            setPlaceholder: function(e) {
                if (/^\s*$/.test($(this).text())) {
                    $(this).empty();
                    var placeholder = utils.html.addTag($(this), 'p').addClass('placeholder');
                    placeholder.append($(this).attr('editor-placeholder'));
                    utils.html.addTag($(this), 'p', typeof e.focus != 'undefined' ? e.focus : false, true);
                } else {
                    $(this).find('.placeholder').remove();
                }
            },
            removePlaceholder: function(e) {
                $(this).find('.placeholder').remove();
            },
            preserveElementFocus: function() {
                var anchorNode = w.getSelection() ? w.getSelection().anchorNode : d.activeElement;
                if (anchorNode) {
                    var current = anchorNode.parentNode,
                        diff = current !== cache.focusedElement,
                        children = this.children,
                        elementIndex = 0;
                    if (current === this) {
                        current = anchorNode;
                    }
                    for (var i = 0; i < children.length; i++) {
                        if (current === children[i]) {
                            elementIndex = i;
                            break;
                        }
                    }
                    if (diff) {
                        cache.focusedElement = current;
                        cache.focusedElementIndex = elementIndex;
                    }
                }
            },
            setContentArea: function(elem) {
                var id = $('body').find('.jquery-editor').length + 1;
                elem.attr('data-jquery-notebook-id', id);
                var body = $('body');
                contentArea = $('<textarea></textarea>');
                contentArea.css({
                    position: 'absolute',
                    left: -1000
                });
                contentArea.attr('id', 'jquery-notebook-content-' + id);
                body.append(contentArea);
            },
            prepare: function(elem, customOptions) {
                options = customOptions;
                actions.setContentArea(elem);
                elem.attr('editor-mode', options.mode);
                elem.attr('editor-placeholder', options.placeholder);
                elem.attr('contenteditable', true);
                elem.css('position', 'relative');
                elem.addClass('jquery-notebook editor');
                actions.setPlaceholder.call(elem, {});
                actions.preserveElementFocus.call(elem);
                if (options.autoFocus === true) {
                    var firstP = elem.find('p:not(.placeholder)');
                    utils.cursor.set(elem, 0, firstP);
                }
            }
        },
        rawEvents = {
            keydown: function(e) {
                var elem = this;
                if (cache.command && e.which === 65) {
                    setTimeout(function() {
                        bubble.show.call(elem);
                    }, 50);
                }
                utils.keyboard.isCommand(e, function() {
                    cache.command = true;
                }, function() {
                    cache.command = false;
                });
                utils.keyboard.isShift(e, function() {
                    cache.shift = true;
                }, function() {
                    cache.shift = false;
                });
                utils.keyboard.isModifier.call(this, e, function(modifier) {
                    if (cache.command) {
                        events.commands[modifier].call(this, e);
                    }
                });

                if (cache.shift) {
                    utils.keyboard.isArrow.call(this, e, function() {
                        setTimeout(function() {
                            var txt = utils.selection.getText();
                            if (txt !== '') {
                                bubble.show.call(elem);
                            } else {
                                bubble.clear.call(elem);
                            }
                        }, 100);
                    });
                } else {
                    utils.keyboard.isArrow.call(this, e, function() {
                        bubble.clear.call(elem);
                    });
                }

                if (e.which === 13) {
                    events.enterKey.call(this, e);
                }
                if (e.which === 27) {
                    bubble.clear.call(this);
                }
                if (e.which === 86 && cache.command) {
                    events.paste.call(this, e);
                }
                if (e.which === 90 && cache.command) {
                    events.commands.undo.call(this, e);
                }
            },
            keyup: function(e) {
                utils.keyboard.isCommand(e, function() {
                    cache.command = false;
                }, function() {
                    cache.command = true;
                });
                actions.preserveElementFocus.call(this);
                actions.removePlaceholder.call(this);

                /*
                 * This breaks the undo when the whole text is deleted but so far
                 * it is the only way that I fould to solve the more serious bug
                 * that the editor was losing the p elements after deleting the whole text
                 */
                if (/^\s*$/.test($(this).text())) {
                    $(this).empty();
                    utils.html.addTag($(this), 'p', true, true);
                }
                events.change.call(this);
            },
            focus: function(e) {
                cache.command = false;
                cache.shift = false;
            },
            mouseClick: function(e) {
                var elem = this;
                cache.isSelecting = true;
                if ($(this).parent().find('.bubble:visible').length) {
                    var bubbleTag = $(this).parent().find('.bubble:visible'),
                        bubbleX = bubbleTag.offset().left,
                        bubbleY = bubbleTag.offset().top,
                        bubbleWidth = bubbleTag.width(),
                        bubbleHeight = bubbleTag.height();
                    if (mouseX > bubbleX && mouseX < bubbleX + bubbleWidth &&
                        mouseY > bubbleY && mouseY < bubbleY + bubbleHeight) {
                        return;
                    }
                }
            },
            mouseUp: function(e) {
                var elem = this;
                cache.isSelecting = false;
                setTimeout(function() {
                    var s = utils.selection.save();
                    if (s) {
                        if (s.collapsed) {
                            bubble.clear.call(elem);
                        } else {
                            bubble.show.call(elem);
                            e.preventDefault();
                        }
                    }
                }, 50);
            },
            mouseMove: function(e) {
                mouseX = e.pageX;
                mouseY = e.pageY;
            },
            blur: function(e) {
                actions.setPlaceholder.call(this, {
                    focus: false
                });
            }
        },
        events = {
            commands: {
                bold: function(e) {
                    e.preventDefault();
                    d.execCommand('bold', false);
                    bubble.update.call(this);
                    events.change.call(this);
                },
                italic: function(e) {
                    e.preventDefault();
                    d.execCommand('italic', false);
                    bubble.update.call(this);
                    events.change.call(this);
                },
                underline: function(e) {
                    e.preventDefault();
                    d.execCommand('underline', false);
                    bubble.update.call(this);
                    events.change.call(this);
                },
                anchor: function(e) {
                    e.preventDefault();
                    var s = utils.selection.save();
                    bubble.showLinkInput.call(this, s);
                    events.change.call(this);
                },
                createLink: function(e, s) {
                    utils.selection.restore(s);
                    d.execCommand('createLink', false, e.url);
                    bubble.update.call(this);
                    events.change.call(this);
                },
                removeLink: function(e, s) {
                    var el = $(utils.selection.getContainer(s)).closest('a');
                    el.contents().first().unwrap();
                    events.change.call(this);
                },
                h1: function(e) {
                    e.preventDefault();
                    if ($(window.getSelection().anchorNode.parentNode).is('h1')) {
                        d.execCommand('formatBlock', false, '<p>');
                    } else {
                        d.execCommand('formatBlock', false, '<h1>');
                    }
                    bubble.update.call(this);
                    events.change.call(this);
                },
                h2: function(e) {
                    e.preventDefault();
                    if ($(window.getSelection().anchorNode.parentNode).is('h2')) {
                        d.execCommand('formatBlock', false, '<p>');
                    } else {
                        d.execCommand('formatBlock', false, '<h2>');
                    }
                    bubble.update.call(this);
                    events.change.call(this);
                },
                ul: function(e) {
                    e.preventDefault();
                    d.execCommand('insertUnorderedList', false);
                    bubble.update.call(this);
                    events.change.call(this);
                },
                ol: function(e) {
                    e.preventDefault();
                    d.execCommand('insertOrderedList', false);
                    bubble.update.call(this);
                    events.change.call(this);
                },
                undo: function(e) {
                    e.preventDefault();
                    d.execCommand('undo', false);
                    var sel = w.getSelection(),
                        range = sel.getRangeAt(0),
                        boundary = range.getBoundingClientRect();
                    $(document).scrollTop($(document).scrollTop() + boundary.top);
                    events.change.call(this);
                }
            },
            enterKey: function(e) {
                if ($(this).attr('editor-mode') === 'inline') {
                    e.preventDefault();
                    e.stopPropagation();
                    return;
                }

                var sel = utils.selection.getSelection();
                var elem = $(sel.focusNode.parentElement);
                var nextElem = elem.next();
                if(!nextElem.length && elem.prop('tagName') != 'LI') {
                    var tagName = elem.prop('tagName');
                    if(tagName === 'OL' || tagName === 'UL') {
                        var lastLi = elem.children().last();
                        if(lastLi.length && lastLi.text() === '') {
                            lastLi.remove();
                        }
                    }
                    utils.html.addTag($(this), 'p', true, true);
                    e.preventDefault();
                    e.stopPropagation();
                }
                events.change.call(this);
            },
            paste: function(e) {
                var elem = $(this),
                    id = 'jqeditor-temparea',
                    range = utils.selection.save(),
                    tempArea = $('#' + id);
                if (tempArea.length < 1) {
                    var body = $('body');
                    tempArea = $('<textarea></textarea>');
                    tempArea.css({
                        position: 'absolute',
                        left: -1000
                    });
                    tempArea.attr('id', id);
                    body.append(tempArea);
                }
                tempArea.focus();

                setTimeout(function() {
                    var clipboardContent = '',
                        paragraphs = tempArea.val().split('\n');
                    for(var i = 0; i < paragraphs.length; i++) {
                        clipboardContent += ['<p>', paragraphs[i], '</p>'].join('');
                    }
                    tempArea.val('');
                    utils.selection.restore(range);
                    d.execCommand('delete');
                    d.execCommand('insertHTML', false, clipboardContent);
                    events.change.call(this);
                }, 500);
            },
            change: function(e) {
                var contentArea = $('#jquery-notebook-content-' + $(this).attr('data-jquery-notebook-id'));
                contentArea.val($(this).html());
                var content = contentArea.val();
                var changeEvent = new CustomEvent('contentChange', { 'detail': { 'content' : content }});
                this.dispatchEvent(changeEvent);
            }
        };

    $.fn.notebook = function(options) {
        options = $.extend({}, $.fn.notebook.defaults, options);
        actions.prepare(this, options);
        actions.bindEvents(this);
        return this;
    };

    $.fn.notebook.defaults = {
        autoFocus: false,
        placeholder: 'Your text here...',
        mode: 'multiline',
        modifiers: ['bold', 'italic', 'underline', 'h1', 'h2', 'ol', 'ul', 'anchor']
    };

})(jQuery, document, window);
```

What does the above code do?
  - makes an image carousel
  - makes a WISYWIG text editor
  - parses text for Arnold schwarzenegger sayings

Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data


What command is not supported?
  - bold
  - underline
  - strikethrough
  - italic

When checking for formatting, the code above uses a recursive call.  On what line does it call itself?












---------


```rb
require 'active_support/concern'

module Mailchimpable
  extend ActiveSupport::Concern

  #                      ### Marketing Pipeline ###
  #                      last updated: Mar. 04 2014

  #        :unbucketed                                                 Day 0
  #         /         \
  #    purchase      no-action                                         Day 1
  #       /                  \
  # :welcome_to_ds           :your_surprise
  #    /      \             /   |    |     \
  # :diy_for :vips  *purchase cart diyemail no-action                  Day 3
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
  #          /   \
  #  :diy_for     :vips    <-- depends on purchase of diy_for box

  # * Designer applicants ommitted from the flow

  included do
    state_machine :mailchimp_list, initial: :unbucketed do
      state :unbucketed,
            :unsubscribed,
            :designer,
            :darby_girl,
            :welcome_to_ds,
            :your_surprise,
            :diy_for,
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
      after_transition any => any, do: :mailchimp_list_subscribe

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
        transition all - [:unbucketed] => :diy_for,     if: :purchased_diy_for?
        transition all - [:unbucketed] => :vips,    unless: :purchased_diy_for?
        transition unbucketed: :welcome_to_ds,          if: :t0?
      end

      event :added_item_to_cart do
        transition your_surprise: :lets_get_crafty
      end

      event :interested_in_diy_for do
        transition your_surprise: :reveal
      end
    end

    after_create :subscribe_to_darby_girl_list, if: :darby_girl_role?
    after_create :subscribe_to_designer_list, if: :designer_applicant?
    after_create :mailchimp_list_subscribe
  end

  def darby_girl_role?
    roles.to_a.include?(:darby_girl)
  end

  def mailchimp_list_subscribe
    if $rollout.active?(:subscribe_to_mailchimp)
      Resque.enqueue(SubscribeUserToMailchimpList, id, mailchimp_list)
    end
  end

  def mailchimp_list_unsubscribe
    if $rollout.active?(:subscribe_to_mailchimp)
      Resque.enqueue(UnsubscribeUserFromMailchimpList, id, mailchimp_list)
    end
  end

  def purchased_diy_for?
    subscribers.active.any?{|subscriber| subscriber.offer.name == 'DIY For'}
  end

  def has_incomplete_diy_for?
    subscribers.any?{|subscriber| subscriber.offer.name == 'DIY For'}
  end

  def subscribe_to_darby_girl_list
    update_attributes(mailchimp_list: :darby_girl)
  end

  def subscribe_to_designer_list
    update_attributes(mailchimp_list: :designer)
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

Before and after each state transition what happens?

After creation, they are assigned to the designer list only if what?

What are the five events that transition the customer's state?

After making a purchase, an unbucketed customer less than 4 days old is placed on the what email list?


Where is this code executed?
  - the browser
  - the server
  - the command line
  - this is not code, this is just encoded data

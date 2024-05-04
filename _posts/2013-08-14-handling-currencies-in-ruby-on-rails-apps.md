---
layout: post
title: "Handling currencies in Ruby on Rails apps"
tags: [tutorial, Ruby on Rails, currency]
---
{% include JB/setup %}

Many Ruby on Rails applications handle money and currencies, from prices for products or subscriptions to money values that must be converted to various currencies and be updated over time for changing currency rates.  In this tutorial I'll show you the steps to storing, manipulating, and formatting currencies in your Rails applications that is both easy and flexible.  The method described in this tutorial is one I used in my [open source project CollectorMetric](https://github.com/themetric/collectormetric) which makes extensive use of many different currencies and money operations.  

### STEP 1 currency in the database 

Probably the most important part of handling curriencies in Rails applications is how you store them in your database so that you don't lose any information about the money (like the cents!).  The data types integer, decimal, and float immediately come to mind as feasible options for storing our currency.  Integers don't have decimal or fractional components so we will be losing the cents portion of our money.  Decimals seem like a good option and could be defined in the Rails schema as having a certain `precision` (total number of digits, including decimal portion) and a `scale` (number of digits past decimal point, which is 2 for simple currencies).  Rails returns a decimal column attribute as the Ruby datatype BigDecimal.  As long as we use the BigDecimal datatype correctly, it's pretty hassle-free but be sure to look over how floating point operations and comparisons work in [the Ruby documentation](http://www.ruby-doc.org/stdlib-2.0/libdoc/bigdecimal/rdoc/BigDecimal.html). Speaking of floating points, the float datatype is definitely one to avoid when dealing with currencies unless you want to take on a datatype full of mathematic subtleties that will likely result in the loss of precision and lots of headache!  The example below shows one of these headaches: 

    1.9.2p320 :028 > 0.5 - 0.45 - 0.05
     => -1.3877787807814457e-17 

Above we subtracted two floats from another float and the result wasn't 0 as we would expect.  Why does this happen?  It has to do with float being approximated for arithmatic that we're not used to dealing with in everyday math.  Because of these subtleties, we're going to choose the integer data type to store our currency. Oh no, what about the cents?!? Well, we're going to be storing them in_cents (in pennies) and manually do the conversion to a currency (divide by 100).  This way we're dealing with the integer datatype which is easy to grasp and we know we won't ever lose the cents portion of our money because we're storing the amount in cents!  A Rails database migration might look something like this: 

{% highlight ruby %}
class AddPriceToProducts < ActiveRecord::Migration
  def change
    add_column :products, :price_in_cents, :integer
    add_column :products, :currency, :string 
  end
end
{% endhighlight %}

### STEP 2 handle representation of the money datatype 

Now that we're storing our product price in cents and also currency as a string ("USD" "EUR" etc), we want to handle the representation of our cents as actual money, formatted by the kind of currency.  If all of your money attributes are going to be in US dollars or any single currency, you can skip the currency string in the database and hardcode your default currency.  Since our converter from cents to actual money can operate any cents attribute, we'll want to make a Ruby library that can be added to any model whether we're storing a price for a product, tax for an item, or revenue for a business.  We'll create a file that looks like this: 

{% highlight ruby %}
# lib/money_attributes.rb
module MoneyAttributes
  extend ActiveSupport::Concern
  module ClassMethods
    def money_attributes *args      
      args.each do |attribute|
        cents_attribute = "#{attribute}_in_cents"
        define_method attribute do
          send(cents_attribute).try("/", 100.0)
        end
        define_method "#{attribute}=" do |value| 
          value.gsub!(/[^\d.]/,"") if value.is_a? String           
          send("#{cents_attribute}=", value.try(:to_f).try("*", 100))
        end
        define_method "#{attribute}_money" do                                
          Money.new(send("#{attribute}_in_cents").to_f || 0, send("currency")).format 
        end 
        attr_accessible attribute if accessible_attributes.include?(cents_attribute)
      end
    end
  end
end
{% endhighlight %}

The above module add three handy methods to any class with attributes that are stored in cents.  If we added it to our Product model that we defined previously, we would have access to these three new methods: 

{% highlight ruby %}
require "#{Rails.root}/lib/money_attributes"
class Product < ActiveRecord::Base
  include MoneyAttributes   
  money_attributes :price
end 
{% endhighlight %}

After we load the library file with the require statement, we include it into the model so that our Product class now has the three class methods defined, namely `price` which returns a float in dollars in cents (2.56), `price=` so that we have a way of setting our price.  Notice that we're setting the price, and not the price in cents, which is how the data is actually stored.  Here we experience the wonderful benefit of data abstraction, we don't have to worry about how the data is stored, we just have to know how to set the price and our class method handles the rest (conversion to cents and storage).  Notice that we also have a line in there that uses `gsub!` that modifies the input price is from the String class.  If a user adds commas or other string text, it's stripped out and we just have the number which we convert to a float for multiplication into cents.  

{% highlight ruby %}
1.9.2p320 :008 > value = "USD 2,123.45"
1.9.2p320 :009 > value.to_f
 => 0.0 # Not good 
1.9.2p320 :010 > value = "2,123.45" 
1.9.2p320 :011 > value.to_f
 => 2.0 # Also not good
1.9.2p320 :012 > value.gsub!(/[^\d.]/,"") 
1.9.2p320 :013 > value
 => "2123.45" 
1.9.2p320 :014 > value.to_f
 => 2123.45 # Perfect  
{% endhighlight %}

The last method that our handy money attribute library adds to our Product model is `price_money` that autmatically returns our money as a formatted string.  To do this we'll use the popular [Ruby money gem](http://money.rubyforge.org/).  Initializing a new money instance from the Money class expects the amount in cents of the money as the first argument and the currency as the second argument.  The currency can be left blank if you're using a default currency ("USD") or we can get the currency from the Product instance by sending the `currency` method to be called on our product.  The currency is stored in the database as a string currency code.  Later in this tutorial we'll cover how to get all the currency codes and have them be selected easily.  The last line of our library defines these new class methods and attributes as accessible to mass assignment if the original cents attribute (like `price_in_cents`) was originally defined as accessible in the Product class.  

### STEP 3 views for allowing user to set prices and currencies 

Although steps 1 & 2 cover all the necessary producedures for handling currencies in the internal Rails code, in many cases we'll want the user to set prices and currencies through a simple web form.  To do this, we'll create an input for `price` and `currency`.  We're not using `price_in_cents` because users expect to enter prices in dollars, but pennies!  Because of our the data abstraction and converter setup in previous steps, we don't have to worry about prices in cents, just prices in dollars that we can get and set through methods.  A simple_form for a product with a price and a currency might look like this: 

{% highlight ruby %}
# app/views/products/_form.html.erb
<%= simple_form_for @product do |f| %>
  <%= f.input :price %>
  <%= f.input :currency, collection: major_currencies(Money::Currency::TABLE), include_blank: "Select a Currency" %>
<% end %> 
{% endhighlight %}

We get these currencies by using the Money class (previously installed as a gem) and some handy helper methods: 

{% highlight ruby %}
def curr_code_to_sym code 
  # TODO Find a better way to get the symbol 
  Money.new(0, code).symbol 
end 
  
def major_currencies(hash)
  hash.inject([]) do |array, (id, attributes)|
    priority = attributes[:priority]
    if priority && priority < 10
      array ||= []
      array << [attributes[:name], attributes[:iso_code]]
    end
    array
  end
end

def all_currencies(hash)
  hash.inject([]) do |array, (id, attributes)|
    array ||= []
    array << [attributes[:name], attributes[:iso_code]]
    array
  end
end
{% endhighlight %}

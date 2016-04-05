#Integration Guide
This guide includes all information necessary to allow your users to utilize Klarna's on-demand for digital goods offering. In this guide, you will see how to embed our solution into your site in order to offer registration for future payments, as well as one-time payments for a specific amount.

While we've done our best to make these examples easy to dive into, you are encouraged to give the [process overview](process_overview.md) a look as it will provide better context for the contents of this guide.

**Note:** at various points during this guide an API key will be presented. This API key works with our playground environment and may be used freely. In order to acquire a production key, [contact us](mailto:digital.goods.dev@klarna.com).

**Note:** the backend samples in this document are written in Ruby. They will be kept simple and should be clear even without any prior knowledge in Ruby.

###Table of Contents
- [Embedding on-demand for digital goods](#embedding-on-demand-for-digital-goods)
- [Receiving one-time payments](#receiving-one-time-payments)
  - [Placing a payment form on your page](#placing-a-payment-form-on-your-page)
  - [Interacting with the form on your backend](#interacting-with-the-form-on-your-backend)
  - [Updating the frontend](#updating-the-frontend)
- [Recurring payments](#recurring-payments)
  - [Placing a registration form on your page](#placing-a-registration-form-on-your-page)
  - [Interacting with the form on your backend](#interacting-with-the-form-on-your-backend-1)
  - [Updating the frontend](#updating-the-frontend-1)
  - [Using a recurring payment reference](#using-a-recurring-payment-reference)
- [Custom form attributes](#custom_form_attributes)

##Embedding on-demand for digital goods
While the specifics vary slightly depending on your use case, there are three main things you will need to do in order to embed the solution in your site:

1. Use a script tag pointing to Klarna's SDK. There are two versions of the SDK, one that works with our playground environment and one that works with our production environment. Other than the environment they are geared towards, both are functionally equivalent.

   For the playground version, use:

   ```html
   <script async src="https://ondemand-dg.playground.klarna.com/web/js/sdk.min.js"></script>
   ```

   For production, use:

   ```html
   <script async src="https://ondemand.klarna.com/web/js/sdk.min.js"></script>
   ```

   **Note:** We deferred the SDK's execution by adding the `async` attribute to the tag. Use this or any other similar measure.

2. Place a form on the page and mark it with the `klarna-form` class. While we don't want to dive into the specifics just yet, here is an example of such a form:

   ```html
   <form action="/some_action" method="POST" class="klarna-form" data-api-key="test_d4ca9ed6-489b-42b3-8011-dacdcee0fdb6" data-flow="purchase" data-locale="sv" data-merchant-context="daily pass">
      <input type="hidden" name="article_id" value="4553" />
</form>
   ```

   Once the SDK loads, it will seek out forms thusly marked and change their contents to present Klarna's on-demand for digital goods offering. We will see how to interact with such forms later on. Do note the custom attributes used above, the meaning of which you can find in [this table](#custom_form_attributes).

   The hidden input field will be submitted along with the form and is the preferred method for passing data to your backend when the user interacts with Klarna. We will see this in action later on.

3. <a name='define_callback'></a>In an additional script tag, define a callback to be invoked when the form is successfully submitted (no need for a failure callback, as the form will display errors on its own). As before, the exact nature of this callback depends on the use case, but here is an example of such a script tag:

   ```html
   <script>
      onKlarnaSuccess = function(data) {
        // Do all sorts of things
      }
   </script>
   ```

## Receiving one-time payments
Perhaps the nature of your site favors singular transactions over return business, or maybe you wish to offer your users a more casual form of payment. Regardless of the reason, this section will show you how to set up a form that will allow users to perform a single purchase at a specified cost. The code samples in this section are based on the sample site, specifically [the page that demonstrates one-time payments](http://ondemand-dg-playground.herokuapp.com/article/1).

### Placing a payment form on your page
Assuming you've already included the SDK itself on your page, you now need to place the actual payment form somewhere:

```html
<form action="/purchase" method="POST" class="klarna-form" data-api-key="test_d4ca9ed6-489b-42b3-8011-dacdcee0fdb6" data-flow="purchase" data-amount="9900" data-currency="SEK" data-locale="sv" data-merchant-context="daily pass">
  <input type="hidden" name="article_id" value="1">
</form>
```

Note how this form is different than the example we've shown previously. Specifically, the values of some data attributes are different and there are some additional attributes we have not previously seen. As before, consult [the table](#custom_form_attributes) as to their meaning.

### Interacting with the form on your backend
We've previously stated that the forms presented by Klarna are eventually posted as if they were a completely regular form. In the sample above, we expect data to be posted to the `/purchase` endpoint. Let us see how such an endpoint might be set up on the site's backend.

```ruby
# Handle POST requests to '/purchase'
post '/purchase' do
  purchase = Klarna::Purchase.new(
    user_token:    params[:userToken],
    reference:     "IA-#{params[:article_id]}",
    name:          'Interesting Article',
    amount:        9900,
    tax:           600,
    origin_proof:  params[:origin_proof]
  )

  authorization_response = purchase.authorize!
  if authorization_response.success?
    article = Article.find(params[:article_id])
    return json data: article.paid_content, klarna_response: authorization_response.data
  else
    status 500
    return json klarna_response: authorization_response.data
  end
end
```

Let us go over the above code and see what it does.
First, a new purchase is created using the information posted with the form. Should you care to delve into the specifics of the `Klarna::Purchase` object you are welcome to [look at the source code](https://github.com/klarna/sample-digital-goods-backend/blob/master/models/klarna/purchase.rb), but simply put it is a wrapper object for [Klarna's on Demand API](http://docs.inapp.apiary.io/). Specifically, for the creation of [orders](http://docs.inapp.apiary.io/#orders).

Once created, we attempt to authorize the purchase. If everything goes smoothly, we will then respond with the article's paid contents and purchase info. Otherwise, we will return an indicative error.

There are a few things worth noticing in the example above:

* Other than the normal form data submitted (which is where we get `params[:article_id]` from), Klarna on-demand forms supply a few extra parameters.
  * **userToken:** a token that identifies the user's token for the sake of the purchase. For one-time purchases the "user" is transient and is mainly used to carry over the user's preferences as they were filled out in the form.
  * **origin_proof:** A unique string used to secure the purchase. We discuss the specifics of this mechanism in the [process overview](process_overview.md).
  * **email:** As part of the form, the user is asked to supply an email address and that address is made available as a parameter.
* In the samples above the price of the article is hard coded, both in the front and backend. This is, naturally, done for the sake of simplicity and is not an encouraged practice.
* The JSON objects returned by the backend are specifically formatted to interact with the callbacks introduced to the page as part of embedding on-demand for digital goods. We will look at these callbacks in the next section.

### Updating the frontend
Having handled the one-time purchase on your backend, you will most likely want to update the page in some way, perform a redirect or any other appropriate action. Klarna on-demand for digital goods makes this task extremely easy by invoking a specific callback that is expected to have been defined ([as we've seen](#define_callback)). Building upon the previous sections, this callback will probably look like the following:

```javascript
onKlarnaSuccess = function(paid_content) {
  document.getElementById('paid_content').innerHTML = paid_content;
}
```

Note that `onKlarnaSuccess` will be called with the contents of the `data` key we returned from the backend when the purchase succeeds. What we did here is alter the page to display the article's paid content once the purchase went through.

As for errors, they will be displayed by Klarna's form automatically as long as they are returned in the manner show in the backend example.

## Recurring payments
Recurring payments offer more versatility in the way payments are collected from your end users. Setting up recurring payments involves the generation of a reference that can be used to charge customers at any given time. This is perfect for services that have a monthly fee, or for creating a virtually frictionless buying experience. In this section will show you how to set up a form that will allow users to register for recurring payments.

### Placing a registration form on your page
Assuming you've already included the SDK itself on your page, you now need to place the recurring payments form somewhere. This form will allow users to register for recurring payments:

```html
<form action="/subscription" method="POST" class="klarna-form" data-api-key="test_d4ca9ed6-489b-42b3-8011-dacdcee0fdb6" data-flow="recurring-purchase" data-locale="sv" data-merchant-context="monthly subscription">
  <input type="hidden" name="article_id" value="1">
</form>
```

As you can see, this is nearly identical to the way you would set up a one-time payment form, the main difference being the value of the `data-flow` attribute which must be set to `recurring-purchase`.

### Interacting with the form on your backend
Similar to what we've seen for one-time purchases, the form shown above is posted normally to the `/registration` endpoint. Let us see how such an endpoint might be set up on the site's backend.

```ruby
# Handle POST requests to '/subscription'
post '/subscription' do
  recurring_payment_reference = Klarna::Recurring.new(
    user_token:    params[:userToken],
    origin_proof:  params[:origin_proof]
  )

  if recurring_payment_reference.valid?
    User.find_or_create_by_email(params[:email])
    User.recurring_payment_reference = recurring_payment_reference.to_string
    User.save!

    article = Article.find(params[:article_id])
    return json data: article.paid_content
  else
    status recurring_payment_reference.error.status
    return json klarna_response: recurring_payment_reference.error.message
  end
end
```

Let us go over the above code and see what it does.
First, a new recurring payment reference is created using the information posted with the form. For simplicity's sake, imagine that the `Klarna::Recurring` object is a wrapper for [the relevant Klarna API call](http://docs.inapp.apiary.io/#recurringpayments).

What happens next depends on whether or not we managed to secure a recurring payment reference. If we have a reference, we store it in the user associated with the email supplied in the form. We then return the paid content that spurred the user to set up for recurring payments. Otherwise, we return an indicative error.

There are a few things worth noticing in the example above:

* The same extra parameters that were available upon submitting a one-time purchase form are available here.
* The origin proof seen here may *only* be used to acquire a recurring payment reference, and so it is different from a one-time purchase origin proof that is used for securing a purchase.
* There is no necessity in charging the user straight away. In this case, the reference is stored in the user object to be used at a later stage.
* The JSON objects return are identical in structure to the ones seen in the one-time purchase example.

### Updating the frontend
Updating the frontend in this scenario is virtually identical to what we did in the one-time purchase example, utilizing the same callback mechanism.

### Using a recurring payment reference
In essence, a recurring payment reference acts like the origin proof for one-time purchases, except you can use it over and over again (see the [order API documentation](http://docs.inapp.apiary.io/#post-%2Fapi%2Fv1%2Fusers%2F%7Buser_token%7D%2Forders) which shows how it can be supplied in lieu of an origin-proof).

Suppose you wanted to charge all your users on a monthly basis. You could accomplish that using a piece of code as follows:

```ruby
every 1.day do
  users = User.all_with_monthly_payment_due_for(Date.today)
  users.each do |user|
    purchase = Klarna::Purchase.new(
      user_token:    params[:userToken],
      reference:     "MNT-SUB",
      name:          'Monthly subscription fee',
      amount:        20000,
      tax:           1200,
      origin_proof:  user.recurring_payment_reference
    )

    authorization_response = purchase.authorize!
    if authorization_response.succeed?
      user.monthly_payments.create(Date.today, authorization_response)
    else
      send_notification_email(user)
    end
  end
end
```

## <a name="custom_form_attributes"></a>Custom form attributes
In this section we will present and clarify the various custom attributes you may use to configure the Klarna on-demand for digital goods forms you embed in your site.

Attribute | Description
:---: | :---
data-api-key | This is where you must supply your API key for the form to function properly.
data-flow | The type of on-demand form to display. The supported values are `purchase` and `recurring-purchase`
data-locale | The locale in which the form should be presented.
data-amount | This attribute is valid only for purchase forms and denotes the purchase's total price. The value should be multiplied by 100, so if your price is "9,90" it should be sent as "990".
data-currency | This attribute is valid only for purchase forms and denotes the purchase's currency, as an [ISO 4217 code](https://en.wikipedia.org/wiki/ISO_4217).
data-merchant-context | This attribute is optional. We recommend sending information about the product. Klarna will use this to segment your traffic and will be able to offer ideas that may help your conversion as a merchant. A practical example is to divide the traffic between "daily pass" and "weekly pass".

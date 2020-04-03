---
layout: post
title:  "Patterns For Building Records In Salesforce"
date:   2020-03-22
categories: Apex Design Patterns
---

What is the best way to build a record in Salesforce?

Sometimes it is as simple as:
	
	Account a = new Account(Name = 'test');

But more often there is complex logic behind record creation. Setting the right field value, creating the right relationships, & bulkification are all common complexities that must be solved.

Let's go through some scenarios to evaluate patterns for building records.

## A simple build

Suppose we're building a community for a bio tech company. Part of the revenue for company comes from selling license to farmers to grow a GMO apple. Farmers have to log into the community and buy a licence. Here's the data model we'll be working with.

![a simple data model](/assets/images/simple_data_model.PNG)

So when a farmer logs into the community & selects a licence to buy we need to create a cart, an apple licence, and a cart item for the farmer.

Let's assume the cart & the apple licence are created for us & all we have to build is the cart item. How might we create builder for that job?

	AppleLicenceCartItemBuilder
		Cart 
		AppleLicenceProduct
		AppleLicence

		build()
			return new CartItem (
				Cart = this.Cart
				Licence = this.AppleLicence
				Price = this.AppleLicenceProduct.Price
			)

Pretty easy! Now to create apple licence cart items all we have to do is:

- get the cart
- get the apple licence
- get the apple licence product
- create an `AppleLicenceCartItemBuilder` and call `build()`

![a simple builder](/assets/images/simple_builder.png)

This will continue to work even if the bio tech company expands and starts selling different types of apple licences.

## A more complex build

The bio tech company has expanded in a way we didn't expect! Now it is selling carrot licences. Here is the revised data model. We could maybe improve the data model & make it really easy to build cart items. For the sake of argument let's say that we have to work with this data model.

![a more complicated data model](/assets/images/data_model_2.png)

Now the builder must support accepting two different types of products & two different types of licences. This seems like an ideal case for inheritance!

	abstract CartItemBuilder
		Cart
		Product

		build()
			return new CartItem (
				Cart = this.Cart
				Price = this.Product.Price
			)

	AppleLicenceCartItemBuilder extends CartItemBuilder
		AppleLicence

		build()
			return super.build().withAppleLicence(
				this.AppleLicence
			)

	CarrotLicenceCartItemBuilder extends CartItemBuilder
		CarrotLicence

		build()
			return super.build().withCarrotLicence(
				this.CarrotLicence
			)

Now to create cart items all we have to do is:

- get the cart
- get the specific licence, carrot or apple
- get the product associated to that licence
- create the correct `CartItemBuilder` and call `build()` on it

![a more complicated builder](/assets/images/builder_2.png)

This is great because it encapsulates the complexity of building generic cart items. There's a little bit of code duplication in the build methods; but there's also a lot of good here. Now we can ignore how a generic cart item is built when building a specific cart item. For example, if the cart item needed to copy a description from the product then only the CartItemBuilder would need to change.

	abstract CartItemBuilder
		Cart
		Product

		build()
			return new CartItem (
				Cart = this.Cart
				Description = this.Product.Description
				Price = this.Product.Price
			)

With this single change both apple & carrot cart items are being built correctly.

Similarly, this encapulates the complexity of building each specific cart item. For example, if the description were actually dependent on the specific licence being bought then that could be added to each specific cart item builder.

	AppleLicenceCartItemBuilder extends CartItemBuilder
		AppleLicence

		build()
			return super.build().withAppleLicence(
				this.AppleLicence
			).withDescription(
				this.AppleLicence.Description
			)

	CarrotLicenceCartItemBuilder extends CartItemBuilder
		CarrotLicence

		build()
			return super.build().withCarrotLicence(
				this.CarrotLicence
			).withDescription(
				this.CarrotLicence.Description
			)

## Even more complicated!

The client comes to us with a final requirement: licence cancellation. Cancellation can comprise a charge or a credit; so a cart item is still a good representation of this operation. So we'll update our data model with a 'type' field on the cart item. When the farmer is buying a licence we'll build a 'purchase' cart item, and when the farmer cancels a licence we'll build a 'cancel' cart item.

![most complex data model](/assets/images/data_model_3.png)

The addition of a single type of cart item has doubled the number of distinct cart items we have to build. Things will get worse if the client adds more 'types' to the cart item (such as a freeze) or more types of licences (like banana licences). The number of distinct cart items our system can build is N*M where N is the number of licences and M is the number of cart item types.

Here are some options.

We could add a method to each of the builders for each type of licence.

	abstract CartItemBuilder
		Cart
		Product

		buildPurchase()
			return new CartItem (
				Cart = this.Cart
				Price = this.Product.Price
				Type = Purchase
			)

		buildCancel()
			return new CartItem (
				Cart = this.Cart
				Price = this.Product.Price
				Type = Cancel
			)

	AppleLicenceCartItemBuilder extends CartItemBuilder
		AppleLicence

		buildPurchase()
			return super.buildPurchase().withAppleLicence(
				this.AppleLicence
			)

		buildCancel()
			return super.buildCancel().withAppleLicence(
				this.AppleLicence
			)

	CarrotLicenceCartItemBuilder extends CartItemBuilder
		CarrotLicence

		buildPurchase()
			return super.buildPurchase().withCarrotLicence(
				this.CarrotLicence
			)

		buildCancel()
			return super.buildCancel().withCarrotLicence(
				this.CarrotLicence
			)

![complicated builder option 1](/assets/images/builder_3_option_1.png)

Hmm... This makes the code duplication problem worse. What if we made our builders 'type' oriented instead of licence oriented? 

	abstract CartItemBuilder
		Cart
		Product
		AppleLicence
		CarrotLicence

		build()
			if (AppleLicence != null) 
				return new CartItem (
					Cart = this.Cart
					Price = this.Product.Price
					Licence = AppleLicence
				)
			else if (CarrortLicence != null) 
				return new CartItem (
					Cart = this.Cart
					Price = this.Product.Price
					Licence = CarrortLicence
				)

	PurchaseCartItemBuilder extends CartItemBuilder
		build()
			return super().build().withType('Purchase')

	CancelCartItemBuilder extends CartItemBuilder
		build()
			return super().build().withType('Cancel')

There's less code duplication in this design but now there's a complexity choke point in the 'CartItemBuilder'. It's got multiple responsiblities that will grow with the number of licence types. This won't scale!

![complicated builder option 2](/assets/images/builder_3_option_2.png)

We can simplify further by abstracting the idea of a licence.  

	Licence interface
		getLicenceId

		getLicenceCartItemLookupField


	AppleLicence implements Licence
		getLicenceId
			return this.Id

		getLicenceCartItemLookupField
			return CartItem.AppleLicence

		getLicencePrice
			return this.AppleProduct.Product.Price

	CarrotLicence implements Licence
		getLicenceId
			return this.Id

		getLicenceCartItemLookupField
			return CartItem.CarrotLicence

		getLicencePrice
			return this.CarrotProduct.Product.Price

	abstract CartItemBuilder
		Cart
		Licence

		build()
			return new CartItem (
				Cart = this.Cart
				Price = Licence.getLicencePrice()
				Licence.getLicenceCartItemLookupField() = Licence.getLicenceId()
			)

	PurchaseCartItemBuilder extends CartItemBuilder
		build()
			return super().build().withType('Purchase')

	CancelCartItemBuilder extends CartItemBuilder
		build()
			return super().build().withType('Cancel')

Now to create cart items all we have to do is:

- get the cart
- get the specific licence, carrot or apple
- contain the licence in either a 'CarrotLicence' or 'AppleLicence' class
- build a `PurchaseCartItemBuilder` or a `CancelCartItemBuilder` depending on what is needed
- call `build()` on it

![complicated builder option 3](/assets/images/builder_3_option_3.png)

By moving all the licence-specific logic into the 'Licence' class we've reduced the scope of responsiblity of our builders. Now the builders can focus on just building the correct type of cart item without caring about the specific type of licence being built. 

Other things to try out here:

- encapsulate the 'type' into an 'operation' class similar to how the 'licence' was encapsulated in a 'licence' class
- create PurchaseCartItemBuilder and an AppleLicenceCartItemBuilder & pass the cart item through both builders
- move the 'operation' information into its own table separate from the 'CartItem' table

## Summary

Containment and inheritence are powerful tools for making your classes ~~completely illegible~~ encapsulated and simple. The best way to find the right balance between too-many and too few classes is to:
	
1. be aware of the data structures, algorithms and design patterns at your disposal
2. try out a few different designs at a high level before writing any code
3. ideally get somebody to review your design before writing any code

Iterating on pseudocode is much faster than iterating on code. It's also much easier for your coworkers to find problems in your pseudocode than it is for them to find problems in your code. Can you imagine the amount of time wasted if you implemented one of the inferior designs above, only to figure out at the end that it wasn't the ideal approach? You'd be faced with the difficult decision of either throwing away what you'd written or committing to an inferior design.

It is especially important to quickly iterate on designs when working on Salesforce because the complexity of above design challenges would be compounded by the restrictions of Apex transactions limits. When building dozens of CartItems in a single transaction:

- how would you ensure that the product information is available within the builders? 
- would you bulkify the builder or create many individual builders?
- when is the appropriate time to execute DML?

The answer may be a builder for your builders. Oy...
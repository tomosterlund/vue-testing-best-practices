# Vue testing best practices

This is a collection of best practices for testing Vue components. All examples illustrate usage
of [Vue test utils](https://github.com/vuejs/test-utils), but the principles should apply to
other testing libraries as well.

Most of these best practices are inspired
by [JavaScript Testing Best Practices](https://github.com/goldbergyoni/javascript-testing-best-practices)
by renowned tech author Yoni Goldberg, whose Book has a whooping 22k stars on GitHub. The goal of
this document is to take some of the ideas from his book, and apply in a Vue-setting.

## Section 1: General code hygiene

### 1. The Golden Rule: Design for lean testing

From Yoni Goldberg:

> Testing code is not production-code - Design it to be short, dead-simple, flat, and delightful to
> work with. One should look at a test and get the intent instantly.
>
> See, our minds are already occupied with our main job - the production code. There is no '
> headspace' for additional complexity.
> Should we try to squeeze yet another sus-system into our poor brain it will slow the team down
> which works against the reason we do testing. Practically this is where many teams just abandon
> testing.

<br/>

### 👏 Doing It Right Example: a short, intent-revealing test

```ts
import { shallowMount } from '@vue/test-utils'

describe('Product', () => {
  describe('adding a product to the cart', () => {
    it('should add product with quantity 1 to the cart, when stock is greater than 0', async () => {
      const wrapper = shallowMount(Product, {
        props: {
          stock: 1,
        }
      })

      await wrapper.find('[data-test-id="cart-button"]').trigger('click')

      expect(wrapper.emitted('add-to-cart')).toEqual([[{ quantity: 1 }]])
    })
  })
})

```

### 2. Structure tests by the AAA pattern

Yoni Goldberg says:
> ✅ Do: Structure your tests with 3 well-separated sections Arrange, Act & Assert (AAA). Following
> this structure guarantees that the reader spends no brain-CPU on understanding the test plan:
>
> 1st A - Arrange: All the setup code to bring the system to the scenario the test aims to simulate.
> This might include instantiating the unit under test constructor, adding DB records,
> mocking/stubbing on objects, and any other preparation code
>
> 2nd A - Act: Execute the unit under test. Usually 1 line of code
>
> 3rd A - Assert: Ensure that the received value satisfies the expectation. Usually 1 line of code


<br/>

### 👏 Doing It Right Example: a well-structured AAA test

```ts
import { shallowMount } from '@vue/test-utils'

describe('Product', () => {
  describe('adding a product to the cart', () => {
    it('should add product with quantity 1 to the cart, when stock is greater than 0', async () => {
      // Arrange
      const wrapper = shallowMount(Product, {
        props: {
          stock: 1,
        }
      })

      // Act
      await wrapper.find('[data-test-id="cart-button"]').trigger('click')

      // Assert
      expect(wrapper.emitted('add-to-cart')).toEqual([[{ quantity: 1 }]])
    })
  })
})

```

### 👎 Anti-pattern example: No separation. Harder to interpret.

```ts
import { shallowMount } from '@vue/test-utils'

describe('Product', () => {
  describe('adding a product to the cart', () => {
    it('should add product with quantity 1 to the cart, when stock is greater than 0', async () => {
      const wrapper = shallowMount(Product, {
        props: {
          stock: 1,
        }
      })
      await wrapper.find('[data-test-id="cart-button"]').trigger('click')
      expect(wrapper.emitted('add-to-cart')).toEqual([[{ quantity: 1 }]])
    })
  })
})

```

### 3. For larger arrange-sections, use utility-methods for mounting the component

✅ Do:

Write utility-methods for mounting the component, every time the arrange-section becomes too
bloated. An intent-revealing function name, will be much faster to understand, than reading the
entire setup code. These utility methods can be reused across multiple tests in a test-suite.

❌ Otherwise:

Even if you adhere to the AAA-pattern, your tests can quickly become unreadable when the setup
section becomes bloated. If your arrange-section becomes too lengthy, this causes two significant
problems:

- A person reading the test, cannot effortlessly grasp the intent. They have to spend brain-CPU
  on understanding the test plan.
- The setup cannot be reused between tests.

### 👏 Doing It Right Example: arranging the test in a utility-method

```ts
// cart-spec-utils.ts
export const whenCartHasNoItems = props => {
  return shallowMount(Cart, {
    props: {
      ...props,
      items: [],
      isSpecialOfferActive: false,
    },
    global: {
      mocks: {
        $store: {
          getters: {
            'cart/total': 0,
          }
        }
      }
    }
  })
}
```

```ts
// cart.spec.ts
import { whenCartHasNoItems } from './cart-spec-utils'

describe('Cart', () => {
  describe('when cart has no items', () => {
    it('should show a message saying the cart is empty', () => {
      // Arrange
      const wrapper = whenCartHasNoItems()

      // Act
      // ...

      // Assert
      // ...
    })
  })
})
```

### 👎 Anti-pattern example: large arrange-section. The intent is harder to grasp.

```ts
describe('Cart', () => {
  describe('when cart has no items', () => {
    it('should show a message saying the cart is empty', () => {
      // Arrange
      const wrapper = shallowMount(Cart, {
        props: {
          items: [],
          isSpecialOfferActive: false,
        },
        global: {
          mocks: {
            $store: {
              getters: {
                'cart/total': 0,
              }
            }
          }
        }
      })

      // Act
      // ...

      // Assert
      // ...
    })
  })
})
```

### 4. Query HTML elements based on attributes that are unlikely to change

✅ Do:

Query HTML-elements based on attributes that will not change, even if other things in the
implementation do. For example, you could settle on always using `data-test-id`.

❌ Otherwise:

Tests might fail, even though functionality stayed the same, but someone threw out a CSS class that
was no longer needed for styling.

💡 Tip:

Use a utility method in your project, for querying elements based on `data-test-id`. This will
make it easier to change the attribute in the future if needed, and prevent people from having to
debug tests due to misspelling your data attribute. For example:

```ts
// test-utils.ts
export const testId = testId => {
  return `[data-test-id="${testId}"]`
}
```

```ts
// cart.spec.ts
import { testId } from './test-utils'

describe('Cart', () => {
  describe('when cart has no items', () => {
    it('should show a message saying the cart is empty', () => {
      // Arrange
      const wrapper = shallowMount(Cart)

      // Act
      // ...

      // Assert
      expect(wrapper.find(testId('empty-cart-message')).exists()).toBe(true)
    })
  })
})
```

## Section 2: Black-box testing

✅ Do:

Test the external APIs of the component. This is what is often referred to as black-box testing. In
comparison to testing a class with public methods, figuring out what these APIs are might not be as
straight forward. However, a (probably not conclusive) list of APIs that you can test would be:

- User interaction with DOM elements
- Props
- Custom events
- Global state, defined and set outside of component
- Side effects: things that have consequences outside the component
- Effect of other APIs on the DOM

Avoid testing component internals, commonly referred to as implementation details.

❌ Otherwise:

Your tests will be very fragile and break easily. Refactoring & renaming will be a pain. Though
functionality is still fine, your tests will sometimes fail, slowing down the team.

### 5. Test user interaction with DOM elements

✅ Do:

Test user interactions with buttons & different inputs. Whenever you see a `@click`, `@change`
or `@input` in your templates, you probably enable some user behavior that can be tested.

❌ Otherwise:

The wanted consequences of user interactions might break, without you noticing it.

### 👏 Doing It Right Example: testing the effect of interacting with a button

```ts
describe('Cart', () => {
  describe('Moving on to checkout', () => {
    it('should move on to checkout when clicking the "Checkout button"', async () => {
      const wrapper = shallowMount(Cart)
      const checkoutButton = wrapper.find('[data-test-id="checkout-button"]')

      await checkoutButton.trigger('click')

      expect(wrapper.emitted().checkout).toBeTruthy()
    })
  })
})
```

### 6. Test outcomes of different prop values

✅ Do:

Test that your component implements the desired behavior, depending on the values you pass as props.
For example, one might pass a prop `isInteractionDisabled` to a component "ProductListing", and
expect that the component disabled some interactive behavior, when this prop is set to `true`.
If you enjoy writing parametrized tests, this is a prime candidate for doing so.

❌ Otherwise:

Another developer might come along and change something in the implementation, breaking the desired
effect of your props. Another scenario, which happens more often than one might think: someone might
misunderstand the intent of the prop and misuse it, since are no tests to display the use case of
it.

### 👏 Doing It Right Example: testing the effect of different prop values

```ts
describe('ProductListing', () => {
  describe('Interaction with listing', () => {
    it('should not open the product details when clicking on a product image, if interaction is disabled', () => {
      const wrapper = shallowMount(ProductListing, {
        props: {
          isInteractionDisabled: true,
        }
      })
      const imageComponent = wrapper.findComponent(ProductListingImage)

      imageComponent.vm.$emit('open-details', { id: 1 })

      expect(wrapper.vm.$router.push).not.toHaveBeenCalled()
    })
  })
})
```

### 7. Test API to child components (receiving custom events)

✅ Do:

Test that your component reacts the way you would expect it to, on input from a child component. For
example, a component under test: "ProductListing", might receive an event "open-details" from a
child component, and as a response, you want to route to a different page.

❌ Otherwise:

Same consequences as in point 5: Your handling of input might break, not being noticed by anyone,
until a customer comes along and complains.

### 👏 Doing It Right Example: testing the effect of a custom event from a child component

```ts
describe('ProductListing', () => {
  describe('Interaction with listing', () => {
    it('should open the product details when clicking on a product image', () => {
      const wrapper = shallowMount(ProductListing, {
        global: {
          mocks: {
            $router: {
              push: jest.fn(),
            }
          }
        }
      })
      const imageComponent = wrapper.findComponent(ProductListingImage)

      imageComponent.vm.$emit('open-details', { id: 1 })

      expect(wrapper.vm.$router.push).toHaveBeenCalledWith('/product-details/1')
    })
  })
})
```

### 8. Test API to parent components (emitting custom events)

✅ Do:

Test that your component emits the correct events, when you want it to. For example, a component
under test: "CheckoutPayment", might emit an event "payment-successful" to its parent component
"Checkout", when the payment was successful.

❌ Otherwise:

Parent components that implement your component under test, and depend on its API, are more likely
to break when
refactoring.

### 👏 Doing It Right Example: testing that an event is emitted

```ts
describe('CheckoutPayment', () => {
  describe('Payment', () => {
    it('should emit a "payment-successful" event when the payment is successful', async () => {
      const wrapper = shallowMount(CheckoutPayment)
      const paymentComponent = wrapper.findComponent(Payment)

      await paymentComponent.vm.$emit('payment-successful')

      expect(wrapper.emitted('payment-successful')).toBeTruthy()
    })
  })
})
```

### 9. Test effect of global state on component

✅ Do:

Test that your component reacts the way you would expect it to when given a certain global state. In
Vue, state from Pinia or Vuex would the most common thing to test.

### 👏 Doing It Right Example: testing the effect of global state on a component

```ts
describe('Cart', () => {
  describe('Displaying items that a customer has selected', () => {
    it('should show a message saying the cart is empty, when the cart is empty', () => {
      const wrapper = shallowMount(ProductListing, {
        global: {
          mocks: {
            $store: {
              getters: {
                'cart/items': [],
              }
            }
          }
        }
      })

      expect(wrapper.find(testId('empty-cart-message')).exists()).toBe(true)
    })
  })
})
```

### 10. Test side effects

✅ Do:

Test that your component has the desired side effects. For example, you might have a globally
available `TrackingService` object, whose method `purchaseCanceled` should be called when a user
cancels their purchase.

❌ Otherwise:

Other components or services that depend on your component, might break without anyone taking
notice.

### 👏 Doing It Right Example: testing side effects

```ts
describe('Checkout', () => {
  describe('Canceling the purchase', () => {
    it('should notify the tracking service when the purchase is canceled', async () => {
      const wrapper = shallowMount(Checkout, {
        global: {
          mocks: {
            $trackingService: {
              purchaseCanceled: jest.fn(),
            }
          }
        }
      })
      const checkoutComponent = wrapper.findComponent(CheckoutPayment)
      const cancelSpy = jest.spyOn(wrapper.vm.$trackingService, 'purchaseCanceled')

      await checkoutComponent.vm.$emit('purchase-canceled')

      expect(cancelSpy).toHaveBeenCalled()
    })
  })
})
```

### 11. Test effects on DOM

✅ Do:

Test that interactions with any of the component APIs, result in the desired effect on the DOM. For
example, given different prop values, should the DOM react in a certain way? Or: if a dialog is
initially hidden on mounting the component, should it be shown as a reaction to a certain user
input?

❌ Otherwise:

Customer: "The product listing is broken."
Dev or PM: "Can you be more specific"
Customer: "When I go to the product listing, and hover over a product image, I don't get the popup with
all the info like I used to"

### 👏 Doing It Right Example: testing the effect on the DOM
```ts
describe('ProductListing', () => {
  describe('Interaction with listing', () => {
    it('should show the product details when hovering over a product image', async () => {
      const wrapper = shallowMount(ProductListing)
      const imageElement = wrapper.find('[data-test-id="product-image"]')

      await imageElement.vm.$emit('mouseenter')

      expect(wrapper.find(testId('product-details')).exists()).toBe(true)
    })
  })
})
```

roadmap:

~~- do & dont~~

~~- Example 1: Do test user interaction with DOM elements~~  
~~- Example 2: Do test outcomes of different prop values~~  
~~- Example 3: Do test API to child components (receiving custom events)~~  
~~- Example 4: Do test API to parent components (emitting custom events)~~  
~~- Example 5: Do test effect of global state on component~~  
~~- Example 6: Do test side effects~~  
~~- Example 6: Do test effect of local state on DOM~~  
- Example 7: Do not test the resulting local state
- Example 8: Do not invoke component methods, unless this is how the component is implemented
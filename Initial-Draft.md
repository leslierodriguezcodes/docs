# Cypress Best Practices

This is a best practices guide for writing integration tests with Cypress at Affirm. It draws from the recommendations set [here](https://docs.cypress.io/guides/references/best-practices/) in the Cypress official best practice documentation and include some Affirm-specific examples. 

## File structure and naming conventions

### Anti patterns for test organizations and naming convention
- Naming folders after team names is not idea since those team names are garaunteed to stay the same. 

Example: https://github.com/Affirm/web-ux/tree/develop/integration-tests-v2/cypress/specs/user-portal/us affirm-anywhere-adoption is no longer a team name.

### Best Practice 
- Name folders after artifacts (ex: https://github.com/Affirm/web-ux/tree/develop/integration-tests-v2/cypress/specs/user-portal/us/settings)

This makes it easier to determine ownership of artifacts and directories no matter what team changes may happen to the team that owns it now.  

## Cypress Commands 

Cypress commands can be queries, actions, or assertions.

In general, the structure of your test should flow query -> query -> command or assertion(s). **It's best practice not to chain anything after an action command.**

### Anti Pattern: Incorrectly chaining commands
```js
cy.get('.new-todo')
  .type('todo A{enter}') // action
  .type('todo B{enter}') // action after another action - bad
  .should('have.class', 'active') // assertion after an action - bad
```
To avoid these issues entirely, it is better to split up the above chain of commands.

### Best Practice: Correctly ending chains after an action
```js
cy.get('.new-todo').type('todo A{enter}')
cy.get('.new-todo').type('todo B{enter}')
cy.get('.new-todo').should('have.class', 'active')
```

## Timeouts


### Anti Pattern: Using arbitrary custom timeouts

Lets look at the following test. The tests sets some server configurations for axp, and stubbs out a response for a merchant details endpoint. The test then waits for the response and in the then block we have an increase timeout for querying a list of results. 

```js
describe('Shop search', () => {
  beforeEach(() => {
  // stub out search results endpoint
  })

  describe('Affirm Anywhere merchant', () => {
    describe('Not prequalified user', () => {
      it('opens merchant detail page and shows Prequal VCN Flow Modal without prequal amount', () => {
        // Sets up experiment configuratons
        // stub out merchant details endpoint

        cy.visit('/u/shop');
        cy.findByTestId('shop-search-input').type(queryString);

        cy.wait(['@searchResults']).then(() => { // cy.wait() actually uses 2 different timeouts. When waiting for a routing alias, we wait for a matching request for 5000ms, and then additionally for the server's response for 30000ms
          cy.findByRole('listbox', { timeout: 30000 }) // query for the element with a role of `listbox` and wait ✨up to 30 seconds for it to exist in the DOM✨
            .then(($listbox) => {  // if the listbox is found before 30 seconds is up, we go into this block
              const firstSearchResult = $listbox.children().first();
              return cy.wrap(firstSearchResult);
            })
            .click();
        });
      });
    })
  })
});
```
We should keep in mind that future developers looking at this test code won't know how the `30000` was derived, increasing the timeout is only a band-aid that will lead to test flakiness, and it increases the overall runtime of integration tests in both CI and the pipeline.

The above custom timeout was also added inside a `.then` block, which will only happen after the `.wait` resolves. According to cypress docs, 
> cy.wait() actually uses 2 different timeouts. When waiting for a routing alias, we wait for a matching request for 5000ms, and then additionally for the server's response for 30000ms. That means above code will wait for ✨up to 35 seconds for the servers response✨, before the timeout will cause the test to fail. 

We don't really want to give our test 30 seconds to find the listbox element. We know at this point in our code, we already have the data we need so populating it the UI should not take 30 seconds. 

### Best Practice: Use Cypress assertions to wait for something to be true

Lets see how we can accomplish waiting for the search results to populate the drowpdown without increasing the timeout.

```js
cy.wait(['@searchResults']).then(() => { // wait up to the default 35 seconds to get results and then continue with our test
    cy.get('#search-results') // Now wait up to the default timeout for the dropdown to exist
      .find('[role="option"]') // AND contain options inside of it 
      .should('have.length', 4) // AND for those options to the expected length of the our stubbed response
      .then(($listbox) => {
        const firstSearchResult = $listbox.children().first();
        return cy.wrap(firstSearchResult);
      })
      .click();
})
```
When we add the `.should()` then cypress will query for the or the element with a role of option and wait **✨up to 4 seconds for it to exist in the DOM✨** **✨and for it to be populated with 4 options**. 

Then you can proceed with selecting the option you want. 

## Selectors


### Anti Pattern: Using FindByText 
If the copy changes, should the test fail? If not, don't use findbytext.
```js
cy.findByText('Pay with a one time use virtual card').click();
```

### Anti Pattern: Using uninterpretable things like `div > div > div` 
```js
this.selectors = {
  LOAN_TERMS_LINK: '.loan-event__terms-link > div > a',
};
```

### Best practice: Use a data-testid selector when possible

Sometimes you won’t be able to put a data attribute on your element because you are using the components core library. When compiled the data attributes may change, just like the classnames. You may run into the situation where you cannot directly place a data attribute on the element you want to select.  
For example, let’s say you are writing a test for an input that is using the [AmountInput](https://storybook.affirm-dev.com/components-core/6.9.0/?path=/docs/components-forms-generic-numbers-amountinput--amount-input) reusable component. 
If you were to place a data attribute on this component, the library might compile this attribute to something foreign at build time. 

In this case you can try placing a data attribute on the parent element containing this component. 


## Conditional Testing


### Anti-pattern: Conditionals based on the existence of DOM elements

While possible, this is not recommended because tests end up being non-deterministic, meaning different test runs may behave differently.  

Example: Sometimes there is a Modal in front of the element you need to access for your test. So you may be thinking: “IF this modal exists, then close it, else just click my button…”. This is tempting since we deal with a lot of experiments, feature flags, country-based features, even cookies and local storage can affect what the user sees. 


### Best Practices: 

Remove the need to ever do conditional testing.

Force your application to behave deterministically.

According to Cypress docs: "The only way to do conditional testing on the DOM is if you are 100% sure that the state has "settled" and there is no possible way for it to change.... Unfortunately, it is not possible for you to use the DOM to do conditional testing. To do this would require you to know with 100% guarantee that your application has finished all asynchronous rendering and that there are no pending network requests, setTimeouts, intervals, postMessage, or async/await code."

Example 1: 
```js
  cy.intercept('GET', shopSavingsEndpoints.eligible_instruments).as(
    'getEligibleInstruments',
  );

  const getFirstInstrument = () => {
    return cy.wait('@getEligibleInstruments').then(({ response }) => {
      const { instruments }: { instruments: Instrument[] } = response?.body;
      const activeInstrument = instruments.find(
        (instrument: Instrument) => instrument.verification_status === 'active',
      );

      if (activeInstrument) {
        cy.wrap(activeInstrument.title).as('instrumentTitle');
        return cy.findByText(activeInstrument.title);
      }
      return false;
    });
  };
```
  
The condional here is not based on the existence of a dom element, it is based on the stubbed out server resonse for eligible_instruments endpoint.


Example 2:
```js
  if (Cypress.browser.name !== 'chrome') {
    console.log('SKIPPING TEST. ONLY RUNS FOR CHROME.');
    return;
  }
```

While .cy is async, Cypress.browser is not. This method returns the browser object with properties like `name`. This is safe to do. 

See:

[https://docs.cypress.io/guides/core-concepts/conditional-testing#The-strategies](https://docs.cypress.io/guides/core-concepts/conditional-testing#The-strategies) 

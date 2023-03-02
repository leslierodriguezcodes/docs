# Cypress Best Practices

This is a best practices guide for writing integration tests at Affirm. It extends on the recommendations set [here](https://docs.cypress.io/guides/references/best-practices/) in the Cypress official best practice documentation. 

## Cypress Commands 

Queries, actions, & assertions // todo add brief explanation of each

The cy.get() query starts searching the DOM root node as the scope for finding elements (In most cases, it is the document element, unless used inside the .within() command.)
The cy.find() query starts its search from the current scope selected 

In general, the structure of your test should flow query -> query -> command or assertion(s). **It's best practice not to chain anything after an action command.**


## Timeouts


### Anti Pattern: Using arbitrary custom timeouts


todo: find examples in repo

These examples are using `wait()` and `{ timeout: 7000 }` incorrectly. 


The listbox element (dropdown) renders before the options inside of it do. The tests above are written so that it would pick the first option of the table as soon as the dropdown is rendered before all options even exist.

todo: add example from repo


### Best Practice: Use Cypress assertions to wait for something to be true

You can solve this by instead doing the following which retries until the assertion is true. So it will pass once the options are all present. 

```
cy.get('[role="option"]').should('have.length', 2);
```

cy.get() has a default timeout of 4 seconds. If we were to just use .get here, `cy.get('[role="option"]')`, Cypress will query for the element with a role of option and wait **✨up to 4 seconds for it to exist in the DOM✨**.
When we add the .should(), then cypress will query for the or the element with a role of option and wait **✨up to 4 seconds for it to exist in the DOM✨** **✨and for it to be populated with two options**. 

 

Then you can proceed with selecting the option you want.

Cypress commands have default timeouts that can be changed in cypress.config.js. The timeout determins how long Cypress will wait for the element to exist in the DOM. For more info please see: https://docs.cypress.io/guides/core-concepts/retry-ability 

todo: add example from repo

## Selectors


### Anti Pattern: Using FindByText 

If the copy for your feature changes, which is likely to happen, your tests will fail. 

// todo find example of this in our repo

### Best practice: Use a data-test selector whenever possible. 

Sometimes you won’t be able to put a data attribute on your element because you are using the components core library. When compiled the data attributes will change, just like the classnames. You may run into the situation where you cannot directly place a data attribute on the element you want to select.  
For example, let’s say you are writing a test for an input that is really the [AmountInput](https://storybook.affirm-dev.com/components-core/6.9.0/?path=/docs/components-forms-generic-numbers-amountinput--amount-input) reusable component. 
If you were to place a data attribute on this component, the library might compile this attribute to something foreign at build time. This is intentional because libraries don’t want to clash with app classNames or attributes. 

In this case you can try placing a data attribute on the parent element containing this component. 

// todo find example of this in our repo


### Understanding The Differences in selectors

* getBy will throw errors if it can't find the element
* findBy will return and reject a Promise if it doesn't find an element


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
  
Why does this work? 
The condional here is not based on the existence of a dom element, it is based on the stubbed out server resonse for eligible_instruments endpoint.
The .wait() has a default timeout
Addionally, everything inside of the .then block will run *after* the .then executes. If we were to place an if statement outside the .then block, it would rub *before* the .then executes. 

Example 2:
```js
  if (Cypress.browser.name !== 'chrome') {
    console.log('SKIPPING TEST. ONLY RUNS FOR CHROME.');
    return;
  }
```

Why does this work? 

Cypress.browser returns properties of the browser. 




See:

[https://docs.cypress.io/guides/core-concepts/conditional-testing#The-strategies](https://docs.cypress.io/guides/core-concepts/conditional-testing#The-strategies) 

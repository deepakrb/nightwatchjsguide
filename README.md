# NightwatchJS Best Practices

NightwatchJS can be used to run User Acceptance tests which guarantee that certain features run as we expect them. This best practices guide details how we write those tests. It is written as a list of rules.

## Tests

### Rule 1: Make Tests Transactional

Each test should be in its own file and should aim to leave the overall state of the underlying peristence layer the way it found it. Sometimes tests may need to be grouped together as a series of steps. In that case a complete test should be able to fully revert its test changes. Take the following example:

```javascript
// tests/integration_name.js

const addIntegration = (browser) => {
    // Add integration_name code
};
 
const removeIntegration = (browser) => {
    // Remove integration_name code
};

module.exports = {
    "@tags": ["integrations", "integration_name"],
    "Add Integration Name Account": addIntegration,
    "Remove Integration Name Account": removeIntegration,
    "Re-add Integration Name account": addIntegration,
    "End Integration Name Integration test": (browser) => { browser.end(); }
};
``` 

> **Remember**: Always remember to call .end() to ensure the Selenium session is properly closed and multiple browsers don't remain open once your test suite has finished running

Here we execute a test to add and remove an integration account. The implementation is not important, but when the test file (`tests/integration_name.js`) is run the final state of the whole system the same as when it began. Tests are defined outside of the 'exports' statement ensuring assertions are easy to read, disable, change and re-use. If there were an interruption to the system (say an account was added but not removed) you need to guarnatee that running the initial assertion "Add Integration Name Account" would not error or fail.

### Rule 2: Split up a large non-dependent test suites into multiple smaller test suites

You might be tempted to create large Test Suites (a single test file) that contains multiple test cases relating to one 'user story' or 'aspect' that is typical in your organization. Using our previous example, you might want to write a single test suite that involves sharing to an integration then updating a social page as many users would do. 

You should avoid doing this because Nightwatch does not guarnatee that every test case (a single test in a test suite) will run given a failure in the test cases upstream of it. In the following example we have multiple non-dependent tests (test that don't rely on eachother) in a single file. When a test fails the tests that were intended to be run after it will be skipped:

```javascript
✖ integration_name
   - Test Integration Name connection (10.928s)
   - If Integration Name is a success, show success message to user (1.312s)
   SKIPPED:
   - Share To Integration Name
   - Navigate to Social Page
   - Post Tweet 'Integration Name King'   
```

In this case it's quite easy to see that this file (`integration_name.js`) should be split into at least 2 files:

```javascript

share_integration_name.js
social_share.js

```

In this case we could guarantee that these test cases (regardless of the overall user story they were in) could operate indepdendantly of eachtoher. The benefit of splitting these tests is that we now  guarnatee that `Navigate to Social Page` (which represents the first test case in the `social_share.js` page) will be started. As a plus both tests could even potentially be run in parallel.

## Pages 

Pages are an abstraction that helps group bits of code that help tests interact with webpages in a reusable and predictable manner.

### Rule 3: Use waitForPage after calls to navigate

It's common practice to call waitForElementPresent after a page's navigate command. 

```javascript
// tests/home.js

const homePage = browser.page.home();
 
home
    .navigate()
    .waitForElement(@homePageIndentifier, 5000);

```

Because waiting for a page to load is tied to 'navigation completing' you should create the command `waitForPage` command which always proceeds a call to `navigate`.

```javascript
// pages/home.js

commands: [
    {
        waitForPage: function() {
            this.waitForElementVisible("@homePageIdentifier", 5000)
  
            return this;
    }
]
```

This allows you to keep remove page specific selectors from your test file and into the page where they are more likely to be seen and changed with the pages schema.

```javascript
// tests/home.js

const homePage = browser.page.home();
 
home
    .navigate()
    .waitForPage();

```

### Rule 4: Break up complex page elements into sections where possible

Single page apps are often subject to large and complicated `page` files. Sections should be used to represent parts of a page. This provides an additional level of namespacing vs storing all commands and selectors at the top page level. 

They help break up complex page files into small chunks. 

```javascript
// pages/home.js

{
    props: {
        email: "example@example.com",
        password: "example"
    },
    sections: {
        hero: {
            selector: "#hero",
            elements: {
                "signUpButton": "#signUpButton"
            }
            commands: [
                {
                    clickSignUp: function() {
                        this
                            .waitForSelector("@signUpButton", 5000)
                            .click("@signUpButton")
 
                        return this;
                    }
                }
            ]
        }
    }
}
```

In this case the selector `.signUpButton` is the equivalent of `#hero .signUpButton`

### Rule 5: Return this in most cases
Return this from all commands that do not need a return statement. This allows you to chain commands predictably

```javascript
// pages/home.js

commands: [
    {
        waitForPage: function() {
            this.openModal()
                .waitForElementVisible("@homePageIdentifier", 5000)
 
            return this;
    }
]
```

### Rule 6: Always return a useCss browser

When returning this from a command revert the browser to useCss instead of xPath so commands which are chained function predictably.

```javascript
pages/home.js
{
    commands: [
        {
            findAndClickElementWithName(text) {
                this.api
                    .useXpath()
                    .waitForSelector(`//text()[contains(.,'${text}')][1]/ancestor::tr`, 5000)
                    .click(`//text()[contains(.,'${text}')][1]/ancestor::tr`)
                    .useCss()
                 
                return this;
            }
        }
    ]
}
```

## Commands 

### Rule 7: Favour using generic commands

Generic commands allow for bits of code which are executed all across the test suite to be reused. When it comes to components that get used in many pages (like alerts or modals) it is helpful to use commands to limit the amount of duplicate commands we have across pages.

```javascript
pages/home.js
const { homeSection } = browser.page.home().section;
 
homeSection
    .waitForElementVisible("@homeShare", 5000)
    .click(@homeShare)
    .acceptModal(); // Accept modal is a globally available command  (-test/nightwatch/command/acceptModal.js)
```


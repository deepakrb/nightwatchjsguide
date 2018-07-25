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

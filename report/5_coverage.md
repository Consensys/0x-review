# 5 Test Coverage Measurement

Testing is implemented using the Truffle Framework.

The [Solidity-Coverage](https://github.com/sc-forks/solidity-coverage) tool was used to measure the portion of the code base exercised by the test suite, and identify areas with little or no coverage. Specific sections of the code where necessary test coverage is missing are included in section [4 Specific Findings](./4_specific_findings.md).

It's important to note that "100% test coverage" is not a silver bullet. Our review also included a inspection of the test suite, to ensure that testing included important edge cases. On the other hand, we also do not expect to see full test coverage on all `/base` contracts, as they should already have been well tested and reviewed prior to use here, or `/test` contracts, as they are not a permanent part of the system.

The state of test coverage prior to and after our review can be viewed in html rendered from the Github repo, or by opening the `index.html` file from the report directory in a browser.

## Initial coverage

* [Report directory](../test_coverage/initial_coverage/coverage)
* [Github rendered HTML report](https://htmlpreview.github.io/?https://raw.githubusercontent.com/ConsenSys/0x_report/master/test_coverage/initial_coverage/coverage/index.html?token=AV93pbb4wD05MqJirx_cCMINwq0Orkpjks5ZjMjjwA%3D%3D)

## Updated coverage

* [Report directory](../test_coverage/updated_coverage/coverage)
* [Github rendered HTML report](https://htmlpreview.github.io/?https://raw.githubusercontent.com/ConsenSys/0x_report/master/test_coverage/updated_coverage/coverage/index.html?token=AV93pbb4wD05MqJirx_cCMINwq0Orkpjks5ZjMjjwA%3D%3D)

# Quality Tests
A quality test (or 'qtest') is a piece of code that operates on a monitor element to verify the data's "quality" or "goodness" against some statistical measure. E.g. comparing the mean of a histogram to an expected value, or comparing spectra to a reference produced using Monte Carlo methods.

Quality tests are an important part of the framework, allowing users and shifters to quickly identify statistical properties or features in data that may not be obvious "by eye", improving their ability to identify any issues or anomalies with data-taking or the device under test.

DQM4HEP comes with several quality tests ready to use and users may implement their own. Included quality tests are:
* Property within range (`PropertyWithinExpectedTest`) – this obtains one of: the mean, mean90, RMS, RMS90, or median of either a histogram or graph, and checks the result against user-defined criteria. This criteria can be either a range, above a threshold, or below a threshold.
* Exact reference comparison (`ExactRefCompareTest`) – this compares a histogram or graph directly with a user-defined reference, looking for an exact match.
* Fit parameter within range (`FitParamInRangeTest`) – this plots a user-defined function onto a histogram or graph, obtains one of the parameters of the function, and checks that the result falls within a user-defined range.

The following quality tests are planned but not yet implemented as of 2018-06-06:
* Fitted chi-squared within range
* Likelihood fit
* Kolmogorov-Smirnov test
* Scalar distance between values

## Running a quality test
Quality tests can be run using the `dqm4hep-run-qtests` executable, found in `dqm4hep-core/bin/`. The only required input for this executable is an XML steering file, which defines the quality tests to perform, their parameters, and which monitor elements to test. An example of running is:

`dqm4hep-run-qtests -i steeringFile.xml`

As with all DQM4HEP executables, running with the `--help` argument shows the possible arguments with their meanings and usage. An example steering file that runs some default quality tests can be found within `dqm4hep-core/tests/test_samples.xml`.

### Steering files
Steering files use XML to store all the information needed to execute a qtest. An example steering file can be found in `dqm4hep-core/tests/test_samples.xml`.

There are two main sections: the `<qtests>` block and the `<monitorElements>` block, both of which must be within  the `<dqm4hep>` XML tag.

The `<qtests>` block defines the qtests to execute along with their settings or parameters, without reference to what they will be run on:

    <qtests>

      <qtest type="ExampleQualityTest" name="ExampleTest">
        <parameter name="NumericalProperty" value="10"/>
        <parameter name="StringProperty"> SomeString </parameter>
      </qtest>

    </qtests>

The `type` is which qtest type to use and `name` is the name of this instance of the qtest.

The `<monitorElements>` block opens a file using the `<file>` tag, within which each monitor element is opened with `<fileElement>`. Inside this tag, all of the qtests to execute on this monitor element are given.

    <monitorElements>

      <file name="test_samples.root">
        <fileElement path="\TestDirectory" name="TestHistogram">
          <qtest name="ExampleTest1" />
          <qtest name="ExampleTest2" />
        </fileElement>
      </file>

    </monitorElements>

## Writing quality tests
Users can create their own quality tests if the included tests do not satisfy their requirements. The code for existing quality tests can be found within `dqm4hep-core/source/src/plugins/` and can be used as references or templates. Files for quality tests should be given a descriptive name in CamelCase, and end with 'Test', e.g. `ExactRefCompareTest.cc`.

The code for a quality test requires only a single .cc file, which comprises four functions: the constructor, the destructor, readSettings, and userRun. Each will be discussed in a separate subsection below. If required, further functions can be implemented as needed. This is left to the discretion of the user.

Code should be written so as to catch common errors and throw an appropriate exception, especially for errors that may cause segmentation faults. This will help to avoid a qtest halting the execution of other qtests should an error occur. Errors are reported using the `report.m_message()` function, and the error message will appear in the summary of the qtest after execution.

#### Constructor and destructor
For the constructor and destructor, it is sufficient to copy existing code, changing the name of the qtest, and change the variables to be initialised in the qtest. The executable that runs qtests handles everything else. Care should be taken to initialise variables appropriately and to give the qtest a clelar and accurate description:

    ExampleTest::ExampleTest(const std::string &qname)
        : QualityTest("ExampleTest", qname),
          m_someFloatParameter(0.f),
          m_someIntParameter(0)
    {
      m_description = "A description of the test's functionality, as well as the meaning of the quality statistic it outputs.";
    }

#### readSettings
The `readSettings` function initialises the variables of the qtest from the XML steering file which is loaded into memory via `xmlHandle`. This function should be used to read in information from the XML file and validate it to ensure the test can be run. Variables are read in using a combination of one of the `RETURN_RESULT` macros and the `XmlHelper`:

    RETURN_RESULT_IF(STATUS_CODE_SUCCESS, !=, XmlHelper::readParameter(xmlHandle, "PropertyName", m_property))`

In this case, the `RETURN_RESULT_IF` macro will succeed only if the call to the `XmlHelper::readParameter` function is successful (i.e. it returns `STATUS_CODE_SUCCESS`). If it is not successful for any reason, the macro will return that statuscode and the program will exit.

Note that if the specified XML attribute isn't found, the `RETURN_RESULT_IF` macro will receive a `STATUS_CODE_NOT_FOUND` and exit. If a variable is *optional*, use the `RETURN_RESULT_IF_AND_IF` macro instead. This allows for the evaluated expression to return either of two statuscodes, given by the first and second arguments, e.g.:

    RETURN_RESULT_IF_AND_IF(STATUS_CODE_SUCCESS, STATUS_CODE_NOT_FOUND, !=, XmlHelper::readParameter(xmlHandle, "PropertyName", m_property))`

In this case, the macro succeeds provided the expression returns either `STATUS_CODE_SUCCESS` or `STATUS_CODE_NOT_FOUND`, so if the variable is not found in the XML file, the qtest can continue. An example of this behaviour for optional parameters can be found in `PropertyWithinExpectedTest::readSettings`, where it is used to allow the user to specify either only one, or both of the thresholds.

Variables should be checked to ensure that the qtest can be run and that the result is meaningful. While the `XmlHelper::readParameter` and similar functions can take an optional fourth argument for a validator delta function, users are encouraged to make code clear and readable by using if-else statements. This is especially important when checking against more complicated criteria.

#### userRun
The `userRun` function defines the process of the qtest itself, using the monitor element. The result must be a float between 0.0 and 1.0 that represents the "quality" or "goodness" of the test. The meaning of this quality statistic will vary depending on the qtest but will often take the form of a chi-squared or p-value. At absolute minimum, it should represent a pass-fail case, so that a passing qtest reports a quality of 1 and a failing qtest a quality of 0.

The monitor element must first be cast to an appropriate class. This is best accomplished using the provided `objectTo` function. For example, if the expected monitor element is a TH1:

    TH1* myHistogram = pMonitorElement->objectTo<TH1>();

After this, the object can be accessed using it's normal methods, and the qtest can be written as normal C++ code for ROOT objects.

It is useful to include a check for whether the monitor element exists and is the correct type, to prevent segmentation faults, using a comparison to `nullptr`:

    if (nullptr == pMonitorElement->objectTo<TH1>())
    {
      report.m_message("Object does not exist or is of unrecognised type!");
      throw StatusCodeException(STATUS_CODE_INVALID_PTR);
    }

Once the quality statistic of the test is known, it is output using `report.m_quality`. Any other information can be output via `report.m_message` – this is useful for including comments on the result.

### Compiling quality tests
Once the code for a new qtest is complete, it must be compiled and tested. To make sure that the build tool compiles newly-implemented quality tests, navigate to the `dqm4hep-core/build/` directory and repeat the installation commands:

    cmake ..
    make install

#### Unit testing
If you intend to contribute your new qtest to the DQM4HEP repositories on Github, it must also have a unit test. Examples can be found in `dqm4hep-core/source/tests/`. A unit test should test both valid tests as well as common failure modes, such as:
* Valid test with incorrect parameters
* Not enough parameters in steering file
* Empty monitor element
* Incorrect monitor element type


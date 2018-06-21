# Quality Tests
A quality test (or 'qtest') is a piece of code that operates on a monitor element to check the data's "quality" or "goodness" against some criteria. E.g. comparing the mean of a histogram to an expected value, or comparing spectra to a reference produced using Monte Carlo methods.

Quality tests are an important part of the framework, allowing users and shifters to quickly identify properties or features in data that may not be obvious "by eye", improving their ability to identify any issues or anomalies with data-taking or the device under test.

DQM4HEP comes with several quality tests ready to use, detailed below. Users may also implement their own (see [here](#writing-quality-tests) for more information).

| Quality test | ROOT Objects | Required parameters | Optional parameters |
|---|---|---|---|
| [PropertyWithinExpectedTest](#propertywithinexpectedtest) | TH1,<br>TGraph | Property,<br>Method,<br>&ensp;plus others (see below) | |
| [ExactRefCompareTest](#exactrefcomparetest)  | TH1,<br>TGraph,<br>TGraph2D | None | CompareUnderflow,<br>CompareOverflow |
| [FitParamInRangeTest](#fitparaminrangetest) | TH1,<br>TGraph,<br>TGraph2D | FitFormula, TestParameter, DeviationLower, DeviationUpper | GuessParameters,<br>FunctionRange, UseLogLikelihood, UsePearsonChi2, ImproveFitResult |
| [KolmogorovTest](#kolmogorovtest) | TH1,<br>TGraph | None | UseUnderflow,<br>UseOverflow |
| [Chi2Test](#chi2test) | TH1 | None | ComparisonType, UseUnderflow,<br>UseOverflow |

### PropertyWithinExpectedTest
This test takes a monitor element and determines one of the following properties:

- Mean
- Mean90
- Root mean square (RMS)
- Root mean square 90 (RMS90)
- Median

It then compares this property against user-defined criteria. These criteria are one of:

- within a specified range
- above a threshold value
- below a threshold value

The result depends on the comparison type. For a range, the result is a p-value of the property being within the required range. For a threshold, the result is 1 if the property passes the threshold, 0 otherwise.

```xml
<qtest type="PropertyWithinExpectedTest" name="MyMeanTest">
  <parameter name="Property"> Mean </parameter>
  <parameter name="ExpectedValue" value="10"/>
  <parameter name="DeviationLower" value="8"/>
  <parameter name="DeviationHigher" value="12"/>
</qtest>
```

The required parameter `Property` determines which property of the object to calculate. Possible properties are: `Mean`, `Mean90`, `RMS`, `RMS90`, and `Median`. The other three arguments determine the expected value of the property, and the lower and upper bounds, and must be numbers. Which arguments are required depends on the type of test being done. If doing a *within range* test, all three are required. If doing an *above threshold* test, only `DeviationLower` is required. If doing a *below threshold* test, only `DeviationUpper` is required. 

### ExactRefCompareTest
This takes a monitor element and an attached reference, and compares them to see if they are an exact match. These objects can be either TH1s, TGraphs, or TGraph2Ds, but both the object and it's reference must be of the same type. The result is 1 if the two objects are exactly identical, 0 otherwise.

```xml
<qtest type="ExactRefCompareTest" name="MyExactRefTest"/>
```

This test has no parameters. The reference is defined in the `<monitorElement>` section of the XML file, explained [here](#steering-files).

### FitParamInRangeTest
This takes a monitor element and plots a user-defined function onto it, then gets one of the parameters of the function and checks whether it falls within a user-defined range. 

```xml
<qtest type="FitParamInRangeTest" name="MyFitParamTest"/>
  <parameter name ="FitFormula"> gaus(0) </>
  <parameter name ="GuessParameters" value=""/>
  <parameter name ="TestParameter" value="1"/>
  <parameter name ="DeviationLower" value="-0.5"/>
  <parameter name ="DeviationUpper" value="0.5"/>
  <parameter name ="FunctionRange" value=""/>
  <parameter name ="UseLogLikelihood" value="false"/>
  <parameter name ="UsePearsonChi2" value="false"/>
  <parameter name ="ImproveFitResult" value="false"/>
</qtest>
```

The required parameter `FitFormula` defines the formula used for the fit. This must be [...]. The optional argument `UsePearsonChi2` [...]

[...]

### KolmogorovTest
This test takes a monitor element and an attached reference, and performs the Kolmogorov-Smirnov test on the two objects. These objects can be either TGraphs or TH1s, but both the object and it's reference must be of the same type. The result is the p-value output by the Kolmogorov test.

<div class="info-msg">
  <i class="fa fa-info"></i>
  The Kolmogorov-Smirnov test is intended for unbinned data, not histograms. However, ROOT provides a function for performing the Kolmogorov-Smirnov test on histograms, so this is functionlity is also included.
</div>

```xml
<qtest type="KolmogorovTest" name="MyKolmogorovTest">
  <parameter name="UseUnderflow" value="false"/>
  <parameter name="UseOverflow" value="false"/>
</qtest>
```

The optional arguments `UseUnderflow` and  `UseOverflow` control whether the overflow and underflow bins are used to calculate the chi-squared. These may be either `false <string>` or `true`. By default, both of these are `false`. 

### Chi2Test
This test takes a monitor element and an attached reference, and performs the Pearson chi-squared test on the two objects. Both objects must be TH1s. The result is the p-value output by the chi-squared test. This is analogous to the Kolmorogov-Smirnov test (above), but is designed for binned data in histograms.

Here is an example definition of a Chi2Test in XML:

```xml
<qtest type="Chi2Test" name="MyChi2Test">
  <parameter name="ComparisonType" value="UU"/>
  <parameter name="UseUnderflow" value="false"/>
  <parameter name="UseOverflow" value="false"/>
</qtest>
```

The optional argument `ComparisonType` determines the comparison type, based on whether the histograms are weighted or unweighted (see [the ROOT documentation](https://root.cern.ch/doc/master/classTH1.html#a6c281eebc0c0a848e7a0d620425090a5) for more information). This may be either `UU`, `UW`, `WW`, or `NORM`. By default, this is `UU`. The optional arguments `UseUnderflow` and  `UseOverflow` control whether the overflow and underflow bins are used to calculate the chi-squared. These may be either `false <string>` or `true`. By default, both of these are `false`.

## Running a quality test
Quality tests can be run using the `dqm4hep-run-qtests` executable, found in `dqm4hep-core/bin/`. This executable handles the running of the actual binaries for each qtest, as well as obtaining monitor elements from the ROOT file and the setting of parameters. This executable has one required arguments and several optional ones, detailed below. 

###Arguments

```bash
-h
--help
```

Displays usage information, then exits.

```bash
-i <string>
--input-qtest-file <string>
```

(Required) Gives the path to the XML steering file that defines what quality tests to run, their parameters, and what monitor elements to run them on. See the [section below](#steering-files) for more information on these steering files.

```bash
-c
--compress-json
```

Turns on compression for the JSON qtest report output file. Off by default.

```bash
-w
--write-monitor-elements
```

Turns on writing of monitor elements in the qtest report. Off by default.

```bash
-p <string>
--print-only <string>
```

Prints only the quality reports of the given flag. Options are undefined, invalid, insuf_stat, success, warning, and error.

```bash
-e <string>
--exit-on <string>
```

Forces the program to exit if any qtest results in the given code. or greater. Options are `ignore`, `failure`, `warning`, and `error`. This is `failure` by default.

```bash
-v <string>
--verbosity <string>
```

The verbosity of the logger. Options are `trace`, `debug`, `info`, `warning`, `error`, `critical`, and `off`. This is `warning` by default [?].

```bash
-q <string>
--qreport-file <string>
```

Gives the path of the qtest report output file (in JSON) format.

```bash
-o <string>
--root-output <string>
```

Gives the path of a ROOT output file to save the processed mmonitor elements.

```bash
--
--ignore_rest
```

Ignores any arguments following this flag.

```bash
--version
```

Displays version information, then exits.

### Steering files
Steering files use XML to store all the information needed to execute a qtest. An example steering file can be found in `dqm4hep-core/tests/test_samples.xml`.

There are two main sections: the `<qtests>` block and the `<monitorElements>` block, both of which must be within  the `<dqm4hep>` XML tag.

The `<qtests>` block defines the qtests to execute along with their settings or parameters, without reference to what they will be run on. The structure and parameters of these is highly dependent upon the qtest being used – see the section for each qtest [above](#quality-tests).

The `<monitorElements>` block opens a file using the `<file>` tag, within which each monitor element is opened with `<fileElement>`. Inside this tag, all of the qtests to execute on this monitor element are given. In this example below, the qtests `ExampleTest1` and `ExampleTest2` are both performed on the monitor element `TestHistogram`:

```xml
<monitorElements>

  <file name="test_samples.root">
    <fileElement path="\TestDirectory" name="TestHistogram">
    <qtest name="ExampleTest1" />
    <qtest name="ExampleTest2" />
    </fileElement>
  </file>

</monitorElements>
```

Some kinds of qtests require reference objects to compare against, which must be declared in the `<references>` block. References have a `name` parameter which gives the path to the file used as a reference, and an `id` which is a short tag for referring to them later in the XML file. For example:

```xml
<references>
  <file id="mc-ref" name="montecarlo_reference_samples.root"/>
  <file id="ex-ref" name="experiment_reference_samples.root"/>
</references>
```

When a qtest that requires a reference is declared, the reference is given within the `<fileElement>` tag:

```xml
<fileElement path="\TestDirectory" name="TestHistogram">
  <reference id="MyReference"/>
  <qtest name="ExampleTest1"/>
</fileElement>
```

This performs the qtest `ExampleTest1` on the monitor element `TestHistogram`, looking for another ROOT object of the same name within the file `MyReference` points to. It is also possible to use a specific object in a file as the reference:

```xml
<fileElement path="\TestDirectory" name="TestHistogram">
  <reference id="MyReference" path="/path/to/the/reference/file" name="ReferenceHistogram"/>
  <qtest name="ExampleTest2"/>
</fileElement>
```

In this case, this performs `ExampleTest2` on the monitor element `TestHistogram`, using the object `ReferenceHistogram` as the reference. 

## Writing quality tests
Users can create their own quality tests if the included tests do not satisfy their requirements. Quality tests are a type of plugin – see [here](plugin-system.md) for more information on plugins, including how to write and compile them.

The code for the built-in quality tests can be found in `dqm4hep-core/source/src/plugins/` and can be used as references or templates. Files for quality tests should be given a descriptive name in CamelCase, and end with 'Test', e.g. `ExactRefCompareTest.cc`.

The code for a quality test requires only a single .cc file, which has four functions: the constructor, the destructor, readSettings, and userRun. Each is discussed in a separate subsection below. If required, further functions can be implemented as needed. This is left for the user to decide.

Code should be written to catch common errors and throw appropriate exceptions, especially for errors that cause segmentation faults. This will help to avoid a qtest preventing other qtests from running should an error occur. Errors are reported using the `report.m_message()` function, and the error message will appear in the summary of the qtest after it has run.

### Constructor and destructor
For the constructor and destructor, it's enough to copy existing code, changing the name of the qtest and the variables to be initialised. The program that runs qtests handles everything else. Care should be taken to initialise variables properly and to give the qtest a good description:

```cpp
ExampleTest::ExampleTest(const std::string &qname)
    : QualityTest("ExampleTest", qname),
      m_someFloatParameter(0.f),
      m_someIntParameter(0)
{
  m_description = "A description of the test's functionality, as well as the meaning of the quality statistic it outputs.";
}
```

### readSettings
The `readSettings` function initialises the variables of the qtest from the XML steering file, which is loaded into memory via `xmlHandle`. This function should be used to read in information from the XML file and validate it to make sure the test can be run. Variables are read in using a combination of pre-processor macros and XmlHelper. For example:

```cpp
RETURN_RESULT_IF(STATUS_CODE_SUCCESS, !=, XmlHelper::readParameter(xmlHandle, "PropertyName", m_property)) <string>`
```

Note that the above example will fail if the parameter is not present in the XML file, so should only be used for parameters that are required. If a parameter is optional, use the `RETURN_RESULT_IF_AND_IF` macro instead. This allows the parameter to be returned if it is found, or does nothing if it is not. For example:

```cpp
    RETURN_RESULT_IF_AND_IF(STATUS_CODE_SUCCESS, STATUS_CODE_NOT_FOUND, !=, XmlHelper::readParameter(xmlHandle, "PropertyName", m_property)) <string>`
```

For more information on status codes, pre-processor macros, and XML parsing with XmlHelper, see the [core tools section](core-tools.md).

Parameters should be checked to make sure that the qtest can be run and that the result is meaningful. While `XmlHelper::readParameter` and similar functions can take an optional fourth argument for a validator delta function, users should make code clear and readable by using `if-else` statements. This is especially important when checking against more complicated criteria.

### userRun
The `userRun` function defines the process of the qtest itself, using the monitor element. The result *must* be a float between 0.0 and 1.0 that represents the "quality" or "goodness" of the test. The meaning of this quality statistic depends on the test but is often a p-value. At absolute minimum, it should represent a pass-fail case, so that a passing qtest gives a quality of 1 and a failing qtest gives a quality of 0.

The monitor element must first be cast to an appropriate class. This should be done using the `objectTo` function. For example, if the monitor element is a TH1:

```cpp
TH1* myHistogram = pMonitorElement->objectTo<TH1>();
```

After this, the object can be accessed using it's normal methods, and the qtest can be written using normal C++ code for ROOT objects.

It is useful to include a check for whether the monitor element exists and is the correct type, to prevent segmentation faults, using a comparison to `nullptr` and throwing an appropriate status code if the check fails. For example:

```cpp
if (nullptr == pMonitorElement->objectTo<TH1>())
{
  report.m_message("Object does not exist or is of unrecognised type!");
  throw StatusCodeException(STATUS_CODE_INVALID_PTR);
}
```

The meaning of these status codes is documented under *Status codes and useful preprocessor macros* in the [core tools section](core-tools.md).

Once the quality statistic of the test is known, it is output using `report.m_quality`. Any other information can be output using `report.m_message` – this is useful for including comments on the result.

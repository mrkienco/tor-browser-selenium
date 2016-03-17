# tor-browser-selenium [![Build Status](https://travis-ci.org/webfp/tor-browser-selenium.svg?branch=master)](https://travis-ci.org/webfp/tor-browser-selenium)

![DISCLAIMER](https://upload.wikimedia.org/wikipedia/commons/thumb/d/d7/Dialog-warning-orange.svg/40px-Dialog-warning-orange.svg.png "experimental")  **experimental - PLEASE BE CAREFUL**


A Python library to automate Tor Browser with Selenium. Our implementation started as a fork of [tor-browser-selenium](https://github.com/isislovecruft/tor-browser-selenium) by @isislovecruft.

## Requirements

It has been tested on the following versions of the [Tor Browser](https://www.torproject.org/projects/torbrowser.html.en):

* 2.4.7-alpha-1
* 4.0.8
* 3.5
* 5.5.3

It has been tested on Debian Wheezy and Ubuntu Wily. It has not been tested in non-Linux systems, but most of the code should be compatible with Windows and Mac OSX.

Make sure your [Selenium](http://www.seleniumhq.org/) version supports the Firefox version on which the Tor Browser you are using is based.

Also, the system (OS, libraries, etc.) should support the Tor Browser versions being used and have `Python` installed.

## Installation

You may need to install the following packages:

- python-setuptools
- git
- xvfb

With `apt-get`, you can install them by running the following command:

`sudo apt-get install python-setuptools git xvfb`

Clone this repository:

`git clone git@github.com:webfp/tor-browser-selenium.git`

Install:

`sudo easy_install .`

Download the Tor Browser Bundle:

You can find the latest Tor Browser Bundle here: https://www.torproject.org/projects/torbrowser.html.en. Download it and extract the tarball to a directory of your convenience (`TBB_PATH`).

## Use

```python
from tbselenium.tbdriver import TorBrowserDriver
with TorBrowserDriver(TBB_PATH) as driver:
    driver.get('https://check.torproject.org')
```

where `TBB_PATH` is the path to the Tor Browser Bundle directory.


## Test

- Tests assume there is an instance of  `tor` running.
- To run all the tests: `./run_tests.py $TBB_PATH`


## Examples

In the following examples we assume the `TorBrowserDriver` class has been imported, and the `TBB_PATH` variable is the path to the Tor Browser Bundle directory.


### Simple visit to a page
```python
driver = TorBrowserDriver(TBB_PATH)
driver.get('https://check.torproject.org')
driver.quit()
```

### Simple visit using the contextmanager

```python
with TorBrowserDriver(TBB_PATH) as driver:
    driver.get('https://check.torproject.org')
    sleep(1)  # stay one second in the page
```

### Take a screenshot

Currently, we need to add an exception to access the canvas in the Tor Browser permission database. We need to do this beforehand for all the URLs that we plan to visit.

```python
TorBrowserDriver.add_exception(cm.TEST_URL)
with TorBrowserDriver(TBB_PATH, ) as driver:
    driver.get('https://check.torproject.org')
    driver.get_screenshot_as_file("screenshot.png")
```

### Visit with a vitual display

The browser window is placed in a virtual display of dimension 1280x800.

```python
with TorBrowserDriver(TBB_PATH, virt_display="1280X800") as driver:
    driver.get('https://check.torproject.org')  # this won't show the browser window.
    sleep(1)
```

### Don't pollute the profile

This will make a temporary copy of the Tor Browser profile, so that we do not pollute the oiginal profile.

```python
with TorBrowserDriver(TBB_PATH, pollute=False) as driver:
    driver.get('https://check.torproject.org')
    sleep(1)
    # the temporary profile is wiped when driver quits
```

### Use old Tor Browser Bundles

We can use the driver with old Tor Browser bundles by passing specific paths to the Tor Browser binary and profile. This is helpful for using the driver with old Tor Browser Bundles, where the directory structure is different from the one that is currently used by Tor developers.

In this example we used Tor Browser Bundle 3.5, which we assume has been extracted in the home directory.

```python
tbb_3_5 = join(expanduser('~'), 'tor-browser_en-US')
tb_binary = join(tbb_3_5, "Browser", "firefox")
tb_profile = join(tbb_3_5, "Data", "Browser", "profile.default")
with TorBrowserDriver(tbb_binary_path=tb_binary,
                      tbb_profile_path=tb_profile) as driver:
    driver.get('https://check.torproject.org')
```

### TorBrowserDriver + stem

This example shows how to use [stem](https://stem.torproject.org/api/control.html) to launch the `tor` process with our own configuration. We run tor with stem listening to a custom SOCKS and Controller ports, and use a particular tor binary instead of the one installed in the system.

```python
from stem.control import Controller
from stem.process import launch_tor_with_config

# If you're running tor with the TBB binary, instead
# of a tor installed in the system, you need to set
# its path in the LD_LIBRARY_PATH:
custom_tor_binary = join(TBB_PATH, cm.DEFAULT_TOR_BINARY_PATH)
environ["LD_LIBRARY_PATH"] = dirname(custom_tor_binary)

# Run tor
custom_control_port, custom_socks_port = 9051, 9050
torrc = {'ControlPort': str(custom_control_port)}
# you can add your own settings to this torrc,
# including the path and the level for logging in tor.
# you can also use the DataDirectory property in torrc
# to set a custom data directory for tor.
# For other options see: https://www.torproject.org/docs/tor-manual.html.en
tor_process = launch_tor_with_config(config=torrc, tor_cmd=custom_tor_binary)

with Controller.from_port(port=custom_control_port) as controller:
    controller.authenticate()
    # Visit the page with the TorBrowserDriver
    with TorBrowserDriver(TBB_PATH, socks_port=custom_socks_port) as driver:
        driver.get('https://check.torproject.org')
        sleep(1)

# Kill tor process
if tor_process:
    tor_process.kill()
```

For running examples you can check the `test_examples.py` in the test directory of this repository.

## Known catches

If you get an exception with a message like this:

`AttributeError: 'TorBrowserDriver' object has no attribute 'session_id'`,

it probably is because Selenium's command to execute Firefox failed. In order to debug the issue, pass a log file to the TorBrowserDriver to obtain errors messages coming from Firefox. You can indicate the path to the logfile as a parameter to the TorBrowserDriver constructor:

```python
path_to_logfile = "ff.log"
TorBrowserDriver(TBB_PATH, tbb_logfile_path=path_to_logfile)
```
We've found that most of these errors comes from an incompatibility between the OS, Selenium and TorBrowserDriver. We recommend using the latest Tor Browser Bundle on the Ubuntu LTS.
